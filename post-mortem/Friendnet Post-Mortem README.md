# FriendNet Post-Mortem

[![Friendnet Post-Moterm](https://img.youtube.com/vi/96XJ4n2swRg/maxresdefault.jpg)](https://www.youtube.com/watch?v=96XJ4n2swRg)


**Cluster:** spiderweb | **Nodes:** M1 MacBook (ARM64) · HP Stream (x86_64) · Mini PC (x86_64) | **Status:** ✅ Resolved

---

## Incident Summary

| Field | Detail |
|---|---|
| **Incident** | Cross-architecture P2P mesh deployment failure cascade across 4 distinct failure domains |
| **Environment** | Apple M1 (ARM64 control plane) + HP Stream (x86_64 indexer) + Mini PC (x86_64 execution tank) |
| **Stack** | FriendNet (Go ≥1.26.2) · Kubernetes Jobs · Forgejo Edge Vault |
| **Core Problem** | Bypassing Kubernetes local-path storage lock to deliver 4–8GB game library payloads across physical nodes |
| **Why Not Redis** | Redis hits a hard 512MB per-object ceiling — a 4–8GB ROM payload triggers an immediate OOM crash on constrained nodes |

---

## The Blueprint: Why FriendNet

The cluster already had a Redis network for telemetry and lightweight JSON payloads. The problem was game library distribution. Trying to move a 4–8GB ROM through an in-memory message queue isn't a configuration problem — it's a physics problem. Redis isn't built for it and will OOM without hesitation.

FriendNet is a compartmentalized P2P data mesh written in Go that completely restructures the data flow. Rather than routing everything through a central storage class, it splits responsibilities across nodes by function:

- **HP Stream** — Headless indexer. Manages rooms and acts as a directory. Knows where every file lives without physically holding any of them. Runs lean.
- **M1 MacBook** — Primary vault. Holds the actual fragmented game data on local storage.
- **Mini PC** — Execution tank. Runs the emulator. When it needs a ROM, it asks the HP Stream indexer for the location, then receives the payload streamed directly from the M1 via FriendNet's internal P2P protocol. The file gets injected straight into a RAM disk (`emptyDir` / `tmpfs`) — ephemeral, fast, no SSD write required.

FriendNet's zero-trust port-forwarding bypass eliminates traditional routing friction entirely, including NAT and port-forwarding roadblocks that would otherwise make cross-node delivery unreliable on consumer hardware. The result is a secure, air-gapped peer-to-peer network running inside the cluster.

Mathematically elegant. Getting it to actually run — a different story.

---

## Critical Failures

Four distinct failure domains hit across the deployment. Each required a different root-cause resolution.

1. **The Silent Exit 0 Trap** — The Go compiler silently built the wrong binary for the wrong architecture, reported clean success, and exited. A version mismatch (Go 1.22 provided, 1.26.2 required) caused the build to fail without surfacing a non-zero exit code.
2. **The Git Worktree Vault Wipe** — Manually exporting `GIT_WORK_TREE` to a `/tmp` directory tethered Git's entire view of repository truth to a single folder. A push to the Forgejo vault from that context wiped 13,429 files in one operation.
3. **The Identity Lockout** — FriendNet enforces a strict 1:1 mapping between usernames and active room connections. Multiple nodes attempting to connect under the same identity were immediately blocked with `username already connected`.
4. **UI/Security Sanitizer Blockades** — The FriendNet web UI and custom HTML profile pages were blocked at two separate layers: protocol enforcement (HTTP vs. HTTPS) and an application-layer XSS sanitizer that treats internal IP links as malicious.

---

## Failure 1: The Silent Exit 0 Trap

**What Happened:** FriendNet binaries had to be compiled from source. The build was running inside a container on the Asahi Ubuntu Linux partition of the M1. The goal was to compile a Darwin (macOS) binary for the M1 node. What actually executed was a Linux ELF binary, because the container's build context defaulted `GOOS=linux` — the host OS environment the container was running on. The compiler did exactly what it was told. It just wasn't told the right thing.

Compounding this: the Go version mismatch. FriendNet required Go 1.26.2. The build environment was providing Go 1.22. Rather than crashing with a non-zero exit code, the build process completed, the script echoed a success message, and the pod exited with a clean `0`. The binary existed. It was just an ARM64 Linux ELF executable being dropped onto a machine expecting a macOS binary.

**The Mechanism:** `exit 0` means the process executed and completed. It says nothing about whether the output matched the intent. The build script was correct syntax. It ran without panicking. The computer reported delivery. The package was in a ditch.

**Resolution:**

Standardized the entire build environment to the correct Go version and forced the pod to actually fail on compiler errors:

```dockerfile
# Pin the exact Go version FriendNet requires
FROM golang:1.26.2-alpine3.23

# set -e forces the shell to exit immediately on any non-zero return code
# Without this, a failed compile step doesn't fail the pod — it just continues
RUN set -e
```

Cross-compilation for AMD64 targets was automated via a Kubernetes Job running directly on the ARM hardware, with `GOOS` and `GOARCH` explicitly injected as environment variables so the compiler target is never inherited from the host context:

```yaml
env:
  - name: GOOS
    value: "<TARGET_OS>"
  - name: GOARCH
    value: "<TARGET_ARCH>"
```

The `set -e` injection is the critical fix. Without it, a Kubernetes Job pod can complete with `Succeeded` status while the actual build step inside it silently failed. `set -e` converts compiler errors into pod failures, which surfaces as a real `Failed` job status — something the cluster can actually alert on.

---

## Failure 2: The Git Worktree Vault Wipe

**What Happened:** The goal was to push FriendNet configurations to the Forgejo Edge Vault on the HP Stream without placing a `.git` tracking folder on the M1's physical SSD. The solution was to manually override the Git working tree via environment variable, pointing it at a `/tmp` directory containing only the FriendNet configs:

```bash
export GIT_WORK_TREE=/tmp/git-final
```

This tethered Git's entire definition of repository truth to a single temporary directory holding exactly one folder's worth of files. Then Git was told to stage everything, commit it, and push to main.

From Forgejo's perspective, the incoming push was authoritative. Git calculates truth based solely on what is present in the working tree at the time of the commit. Everything not in `/tmp/git-final` — the Redis and nn JSON configs, the K3s manifests, the AMD64 binaries, the rest of the cluster infrastructure — was absent from that snapshot. Git's mathematical conclusion: those files were intentionally deleted. Forgejo executed the sync faithfully.

**13,429 files wiped. One push.**

**Recovery:** A surgical timeline revert in the terminal pulled the broken history out and force-pushed the correct file state back to the vault. No permanent data loss — but a serious near-miss on the entire cluster's configuration backup.

**Resolution:** Abandoned Git worktree manipulation entirely. The correct approach is physical 1:1 directory mirroring — the local directory structure matches the remote exactly, no abstract pointer overrides, no environment variable redirects. Git's working tree should always reflect the full repository state, not an isolated subdirectory snapshot.

---

## Failure 3: The Identity Lockout

**What Happened:** Once FriendNet was running, multiple cluster nodes attempted to connect to the same room using the same username. FriendNet enforces a strict 1:1 mapping between a username and an active room connection. The second connection hit an immediate `username already connected` rejection and was locked out. No fallback, no queuing — the session was blocked outright.

**Resolution:** Each physical node was assigned a distinct persistent identity before connecting. Node identity is not a cosmetic label in FriendNet — it is the session key.

```
<NODE_NAME>  → M1 MacBook (vault)
<NODE_NAME>  → Mini PC (execution tank)
<NODE_NAME>  → HP Stream (indexer)
```

Unique identity per node is a prerequisite, not a configuration option.

---

## Failure 4: UI / Security Sanitizer Blockades

This failure domain had two separate layers, both blocking access through different mechanisms.

### Layer 1: Protocol Mismatch (404 and "Too Many Colons")

**What Happened:** After launch, hitting the server in a browser returned a 404. Looking at the raw JSON logs, the server was running correctly — port 8080 was serving gRPC API endpoints for strict machine-to-machine communication, not visual web pages. The 404 was proof of life, not a crash. The API simply had no handler for a browser's generic HTTP GET request.

The actual web UI was running on port `2042`. Separately, the Admin UI was inaccessible because entering the address with an `http://` prefix caused a "too many colons" parsing error in the network dialer — the protocol prefix conflicted with the address format FriendNet expected.

**Resolution:** Strip the `http://` prefix entirely when entering addresses into the FriendNet dialer. The application handles protocol negotiation internally.

### Layer 2: XSS Sanitizer (Blue Monday Package)

**What Happened:** After finding the correct web UI port, the next step was building static HTML profiles that linked internally to shared files using direct URLs (local IPs like `127.0.0.1` or internal subnet addresses). Every link was immediately stripped by the application's built-in sanitizer.

FriendNet's sanitizer is powered by Go's **Blue Monday** package — a strict HTML sanitization library that operates on a zero-trust model. It automatically purges any HTML content containing:

- Local IP addresses (`127.0.0.1`, `localhost`)
- Internal subnet IPs
- Relative hash routes

The logic is intentional. From the sanitizer's perspective, an internal IP inside a user-generated HTML field is indistinguishable from a malicious XSS payload trying to scan the local network. It doesn't attempt to distinguish between the two — it just nukes it.

**Resolution:**

- For the network dialer: remove `http://` prefixes from all addresses
- For HTML profile pages: replace direct IP links with relative hash routing

```
# Blocked by sanitizer
http://192.168.x.x:PORT/file

# Accepted
#/user/<USERNAME>/files
```

Profile pages in FriendNet are visual billboards only. File access must go through the native browse button — the sanitizer enforces this at the application layer regardless of how the HTML is constructed.

---

## Final Architecture State

| Node | Role | Identity | Holds Data |
|---|---|---|---|
| M1 MacBook (ARM64) | Primary vault · Control plane | `<VAULT_IDENTITY>` | Game library fragments |
| HP Stream (x86_64) | Headless indexer | `<INDEXER_IDENTITY>` | Directory only — no payloads |
| Mini PC (x86_64) | Execution tank | `edge_node_1` | RAM disk (`emptyDir`) — ephemeral |

**Delivery flow:** Mini PC requests ROM → HP Stream locates it on M1 → payload streams directly M1 → Mini PC via FriendNet P2P → injected into `emptyDir` RAM disk → emulator runs from memory.

---

## Lessons

**1. Exit 0 is not a success signal — it is a completion signal.**
The compiler did not fail. The pod did not crash. The script printed a success message. The binary was wrong. `set -e` and explicit environment variable injection (`GOOS`, `GOARCH`) are not optional hygiene — they are the only way to make build pipelines actually fail when the output is wrong.

**2. Git's definition of truth is exactly what is in the working tree — nothing more.**
Pointing `GIT_WORK_TREE` at a `/tmp` subdirectory doesn't tell Git to "only sync this folder." It tells Git that this folder is the entire repository. Everything absent from it is treated as deleted. Abstract worktree manipulation on a live cluster backup vault is how you wipe 13,000 files in one push.

**3. Application-layer sanitizers enforce architecture, not just security.**
Blue Monday blocking internal IPs is not a bug or a misconfiguration — it is a deliberate design constraint that shapes how the application can be used. Trying to route around it with clever HTML is the wrong approach. Understanding what the sanitizer is enforcing and working within its model (hash routing, native UI controls) is the right one.

**4. Identity is infrastructure.**
In a system that enforces 1:1 session mapping, node identity is not a label — it is a session key. It needs to be planned before deployment, not improvised during it.

---

## Skills

**Cross-Architecture Binary Compilation on Kubernetes**
Used Kubernetes Jobs to automate cross-compilation of AMD64 binaries directly on ARM hardware. Managed `GOOS` and `GOARCH` environment injection to decouple compiler target from host build context, and used `set -e` to convert silent build failures into actual pod failures the cluster can surface.

**Exit Code Semantics and Build Pipeline Integrity**
Diagnosed a class of failure where correct syntax, clean exit codes, and explicit success messages masked completely wrong output. Understood that `exit 0` reflects execution completion, not output correctness, and implemented shell-level failure enforcement to close that gap.

**Git Internals: Working Tree Scope and Repository Truth**
Traced a 13,429-file vault wipe to a single `GIT_WORK_TREE` override that redefined the entire repository's scope to a `/tmp` subdirectory. Understood how Git calculates additions and deletions from working tree snapshots, performed a timeline revert and force-push recovery, and rebuilt the backup strategy around physical directory parity instead of abstract pointer manipulation.

**Decentralized P2P Mesh Architecture**
Designed and deployed a compartmentalized data mesh that splits indexing (HP Stream), storage (M1), and execution (Mini PC) across physical nodes with distinct roles. Leveraged FriendNet's zero-trust port-forwarding bypass to eliminate NAT friction and deliver 4–8GB payloads into ephemeral RAM disks without touching persistent storage on the execution node.

**Application-Layer Security Model Analysis**
Diagnosed sanitizer blocks at the HTML profile layer by identifying the Go Blue Monday package's zero-trust constraint set. Distinguished between what the sanitizer blocks (internal IPs, local routes) and why (XSS lateral scan prevention), and resolved it by conforming to the application's intended interaction model rather than working around it.

**Protocol and Port Disambiguation in Distributed Services**
Diagnosed a 404 response as proof of server life rather than a crash — identified that gRPC API endpoints and visual web UI ports are distinct services running on different ports. Resolved protocol dialer conflicts by removing prefix assumptions that conflicted with the application's internal address format.

**Multi-Node Session Identity Management**
Identified FriendNet's strict 1:1 username-to-session enforcement model and established persistent, unique node identities as a deployment prerequisite rather than a post-hoc fix.

---

*Part of the spiderweb cluster postmortem series.*
