# Localsend-Beam post-mortem
[![Localsend-Beam post mortem
](https://img.youtube.com/vi/0LT7jAi4_mc/maxresdefault.jpg)](https://www.youtube.com/watch?v=0LT7jAi4_mc)


**Cluster:** spiderweb | **Nodes:** M1 MacBook (control plane) · HP Stream (edge node) · Mini PC | **Latency:** 49ms post-patch | **Success Rate:** 100% post-patch | **Status:** ✅ Resolved

---

## Incident Summary

| Field | Detail |
|---|---|
| **Incident** | Data pipeline amnesia, metadata poisoning, index lock, cyclic object crash, and protocol rejection across an autonomous n8n + LocalSend GitOps workflow |
| **Environment** | Apple M1 (ARM64 control plane) + HP Stream (x86_64 edge node) + Mini PC |
| **Stack** | n8n · LocalSend REST API · systemd · K3s · PostgreSQL · Forgejo · USB vault |
| **Core Goal** | Fully autonomous workflow that backs up, edits, and restores the cluster's own n8n brain without human intervention |
| **Payload** | 4,470 bytes — exact size of the n8n YAML file that successfully traversed the network bridge and landed on the physical USB vault |

---

## The Blueprint: The Stateless Trifecta

Before the failures, the architecture. The entire design philosophy of this build rests on a single principle: decouple compute from state.

The cluster runs on what the source logs call the **Stateless Trifecta**:

- **K3s** — The compute engine. Does the heavy lifting. Disposable.
- **PostgreSQL** — Short-term memory. Holds runtime state. Also disposable.
- **Git (USB-backed)** — The immutable blueprint. Declarative JSON and YAML. The one thing that cannot be lost.

If PostgreSQL gets wiped tomorrow, the cluster survives — because the automation brain lives in version-controlled files on a physical USB drive, not inside a container that can be evicted. Hardware is treated as cattle, not pets. The state is what matters, and the state lives outside the hardware.

### The n8n + LocalSend Protocol Bridge

The original plan was to use containerized bridges to move data between nodes. ARM and x86_64 architecture mismatches killed that approach immediately. The fix was to remove the abstraction layer entirely: configure n8n to speak the raw **LocalSend REST protocol** directly, bypassing all GUI wrappers and translation layers.

The flow is a two-step handshake-then-transfer pattern:

1. **Handshake** — The n8n HTTP node pings the target edge node and secures a receipt confirming the receiver is alive and ready
2. **Transfer** — n8n grabs the raw binary payload and blasts it directly down the network pipe to the target

No middleware. No container bridges. Pure bare-metal API communication between nodes.

**Lifecycle at stable state:**
- `~1ms` — n8n scans the directory
- `~50ms` — Handshake fires and transfer hits the HP Stream
- `5 min` — Cron job wakes for intelligent content-aware sort

### The "Trifecta" Workflow Chain

Three autonomous n8n workflows manage the cluster's own brain:

| Workflow | Role |
|---|---|
| **Creator** | Generates new workflow definitions as declarative YAML |
| **Editor** | Modifies existing workflow definitions |
| **Backup** | Packages the current brain state and ships it to the Forgejo vault and USB |

Getting all three to work together without stepping on each other is where things collapsed.

---

## Critical Failures

Seven distinct failure domains, each requiring a different root-cause resolution.

1. **Data Pipeline Amnesia** — Sequential nodes replaced the binary payload with their own JSON responses. The final transfer node received an empty suitcase.
2. **The GitHub Receipt Bug** — The backup workflow saved GitHub's API success metadata to the vault instead of the actual workflow JSON code.
3. **The Index Lock** — Sub-workflows defaulted to Index 0 on every loop iteration, causing the system to push the same workflow file regardless of which iteration was active.
4. **The Cyclic Object Error** — Requesting a full HTTP response inside n8n nodes triggered infinite serialization loops by attempting to stringify raw network socket objects.
5. **API Protocol Rejection** — LocalSend rejected incoming transfers because n8n sent human-readable file size strings instead of the raw integer bytes the protocol requires.
6. **Network Flooding via Polling Loop** — Removing the event-driven trigger without adding a state check caused the system to retransmit the same YAML file on every poll cycle, threatening to burn out the HP Stream's eMMC flash storage.
7. **Configuration Drift via YAML Orphaning** — A `grep -v` command stripped parent-level YAML annotations and left all child keys structurally orphaned, triggering an instant parser fatality and breaking the continuous delivery loop.

---

## Failure 1: Data Pipeline Amnesia

**What Happened:** n8n workflows are built as chains of sequential nodes. When the HTTP node completed the LocalSend handshake, it passed its own JSON response — the handshake receipt — downstream as the active data item. The AI Agent node after it did the same thing, replacing the stream again with its own output. By the time execution reached the final transfer node, the original binary payload was gone. The node had a receipt and a response object, but no file. It was trying to ship an empty suitcase.

Separately: even when the payload was correct, the HP Stream's Linux kernel was issuing a hard `ECONNREFUSED` — an RST packet. The reason was that LocalSend had been treated as a user-facing desktop application rather than a background service. With no user session active on the HP Stream, the process died. The port closed. The entire CI/CD pipeline collapsed silently.

**Resolution — The Multiplex Bridge:**

Injected **Merge nodes** configured to `All Possible Combinations` (Multiplex mode) at the critical junctions in the workflow. A Multiplex merge node staples every item from one input against every item from another, creating combined data items that carry both the binary file and the handshake receipt simultaneously. The transfer node now always has both.

For the daemon problem: wrapped the LocalSend execution logic in a `systemd` unit file, stripped all GUI environment variables to keep it lightweight, and appended the `--receive` flag. The unit file forces the Linux kernel to start LocalSend on boot, bind the network socket, and aggressively restart the process on crash. The receiver is now immortal from the kernel's perspective.

```ini
# systemd unit — stripped of GUI env vars, receive flag forces headless mode
[Service]
ExecStart=/usr/bin/localsend --receive -d <USB_MOUNT_POINT>
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

The `-d` flag forces the receiver to write directly to the physical USB mount point rather than the default user home directory.

---

## Failure 2: The GitHub Receipt Bug

**What Happened:** The Backup workflow's job is to push the current n8n brain state to the Forgejo vault. After executing the push, GitHub's API returns a JSON object confirming success — metadata about the operation itself. The workflow was capturing that response object and saving it to the vault as the backup artifact. The vault contained perfect records of successful API calls and nothing else. The actual workflow JSON was never written.

**Resolution:** Separated the push operation response from the payload source. The workflow now explicitly references the workflow JSON as the write target before the GitHub API call executes, so the vault receives the file rather than the receipt.

---

## Failure 3: The Index Lock

**What Happened:** The Trifecta workflows run in a loop — Creator, Editor, Backup cycling through a list of workflow files. Sub-workflows in n8n default to processing `Index 0`, the first item in the input list, regardless of which iteration the parent loop is on. Every cycle of the loop was pushing the Editor workflow. The Creator and Backup never ran.

The root cause was absolute node referencing: `$node['NodeName'].json` hardcodes a reference to a specific named node's output, which always resolves to the first item that node processed. The index never advances.

**Resolution:** Replaced all absolute node references with relative ones:

```javascript
// Absolute — always resolves to Index 0, ignores loop position
$node['HTTP Request'].json.data

// Relative — resolves to the current item in the active execution context
$json.data
$binary.data
```

Relative referencing (`$json`, `$binary`) binds to the current execution item rather than a named node's cached output, which means the loop position actually propagates correctly through the sub-workflow.

---

## Failure 4: The Cyclic Object Error

**What Happened:** Several n8n nodes were configured to return the `Full HTTP Response` rather than just the response body. A full HTTP response object in Node.js contains raw network socket references — live objects that point back to the open connection. When n8n's execution engine attempted to serialize these objects to pass them downstream, the serializer hit the socket reference, followed it, found another reference back to the parent object, and entered an infinite loop. The node crashed with a cyclic object error.

**Resolution:** Disabled `Full HTTP Response` on all nodes where the raw socket data wasn't needed. Scoped the output to the response body only, eliminating the circular reference at the source.

---

## Failure 5: API Protocol Rejection

**What Happened:** The LocalSend REST API requires file size to be passed as a raw integer — bytes, no unit suffix. n8n's default file handling expresses file sizes as human-readable strings (`"5.86 kB"`, `"1.2 MB"`). LocalSend received a string, couldn't parse it as an integer, and rejected the transfer outright before any data moved.

**Resolution:** Injected a `parseInt()` conversion in a JavaScript node before the transfer call:

```javascript
// Convert human-readable size string to raw integer bytes
const fileSizeBytes = parseInt($json.fileSize.replace(/[^0-9]/g, ''));
```

This strips all non-numeric characters and parses the remaining digits as an integer, giving LocalSend the raw byte count it expects.

---

## Failure 6: Network Flooding via Polling Loop

**What Happened:** The event-driven trigger on the transfer workflow was removed during a cleanup pass and replaced with a polling architecture. The polling loop had no state check — no condition to verify whether the file had already been sent. On every cycle, the system blindly retransmitted the identical YAML file regardless of whether the previous transfer had succeeded. The HP Stream's eMMC drive was being written to on an infinite loop. eMMC flash storage has a finite write cycle limit. This was moments away from permanent hardware damage.

**Resolution:** Reinstated event-driven triggers as the execution entry point. Added an explicit state check before any file operation executes — the workflow now verifies transfer status before firing, so retransmission only occurs on a confirmed failure, not on every tick.

---

## Failure 7: Configuration Drift via YAML Orphaning

**What Happened:** A `grep -v` command was used to strip parent-level annotations from a YAML file during a cleanup pass. The command executed correctly — it removed the target lines. But YAML indentation is structural, not cosmetic. A parent key defines the scope for everything indented beneath it. Removing the parent key leaves all child keys floating at an indentation depth that no longer has a valid anchor. The YAML parser does not attempt to guess the intended structure — it throws an immediate parsing error and rejects the file entirely. The continuous delivery loop broke and the local GitOps state was temporarily corrupted.

**Resolution:** Line-based text tools (`grep`, `sed`, `awk`) do not understand YAML's tree structure and cannot safely modify it. All YAML edits now go through structured tooling that parses the document as a tree before making changes, ensuring parent/child relationships remain intact after any modification.

---

## Final Architecture State

**The "Inside Out" Edge Vault:**

| Layer | Component | Role |
|---|---|---|
| **Automation** | M1 MacBook + n8n | Runs the Creator/Editor/Backup workflow chain autonomously |
| **Network Bridge** | LocalSend REST (direct) | Bypasses container bridges, speaks raw protocol to edge node |
| **Edge Node** | HP Stream + systemd | Receives payloads via immortal daemon, routes to USB |
| **Immutable State** | Physical USB (ext4 formatted) | Stores declarative YAML/JSON blueprints — survives node failure |

If the HP Stream's internal eMMC drive fails tomorrow, the blueprints are safe. The USB vault is physically decoupled from the node's internal storage. The cluster can rebuild itself from the USB alone.

---

## Lessons

**1. In n8n, every node replaces the data stream with its own output unless you explicitly staple the original back on.**
Sequential node chains don't carry binary payloads forward automatically. An HTTP node that fires a handshake will pass its JSON receipt downstream, not the file that triggered it. Multiplex merge nodes are the architectural fix — they combine streams so downstream nodes have the full context.

**2. Human-readable formatting is for humans, not APIs.**
`"5.86 kB"` is a string. A file size field in a REST API spec is an integer. These are not interchangeable, and the API will not attempt to parse one as the other. Type enforcement at the boundary — `parseInt()`, explicit casting — is not optional hygiene; it is the transfer happening or not happening.

**3. Absolute node references in workflow loops don't advance with the iteration.**
`$node['Name']` is a fixed pointer to a named node's first resolved output. It does not move with the loop. Relative references (`$json`, `$binary`) bind to the current execution context and do. Mixing the two causes sub-workflows to get stuck replaying the same item.

**4. YAML is a tree, not a text file.**
Line-based tools (`grep`, `sed`, `awk`) operating on YAML without understanding the indentation hierarchy will produce files that look correct and parse as broken. Parent key removal orphans every child below it. Structured YAML tooling is not a preference — it is the difference between a valid file and a parser fatality.

**5. A process that dies when no user is logged in is not a service.**
Treating LocalSend as an interactive desktop application on a headless edge node meant the process lifetime was tied to a user session. No session, no process, no port, no pipeline. `systemd` + `Restart=always` is what makes a process a service — not where it's installed.

---

## Skills

**n8n Workflow Architecture and Data Flow Debugging**
Diagnosed and resolved seven distinct failure modes inside a multi-node autonomous workflow chain: binary data loss across sequential nodes, metadata poisoning from API response capture, index lock from absolute referencing, cyclic serialization from socket objects, type mismatch at REST API boundaries, network flooding from stateless polling, and YAML structural corruption from line-based text manipulation.

**LocalSend REST Protocol Integration**
Bypassed container bridge architecture entirely by making n8n speak the raw LocalSend two-phase protocol (handshake → transfer) directly. Diagnosed and resolved API rejection caused by human-readable file size strings and enforced raw integer byte conversion at the protocol boundary.

**systemd Daemon Engineering on Headless Edge Nodes**
Converted a user-session-dependent desktop process into a persistent systemd service with aggressive restart policy, headless environment stripping, and explicit USB mount path enforcement via the `-d` flag. Process now survives kernel restarts and session absence.

**Compute/Storage Decoupling at the Architecture Level**
Implemented the Stateless Trifecta pattern — K3s (compute), PostgreSQL (runtime state), Git/USB (immutable truth) — such that any single layer can fail without data loss. Cluster state lives in declarative files outside the hardware that runs it.

**Zero-Trust Content-Aware Payload Routing**
Upgraded a file-extension-based router to a content-aware one that peeks inside each payload and scans for Kubernetes signatures (`apiVersion`) or n8n workflow fingerprints (`nodes[`) before routing. Eliminates the risk of a database manifest landing in a workflow folder.

**GitOps State Management and YAML Structural Integrity**
Diagnosed a configuration drift fatality caused by `grep -v` stripping parent YAML annotations and orphaning all child keys below them. Understood YAML's two-dimensional indentation dependency and replaced line-based text manipulation with structured tooling.

**Autonomous Backup Pipeline Design**
Engineered a three-workflow Creator/Editor/Backup chain that manages the cluster's own n8n brain without human intervention — generating, modifying, and shipping declarative workflow files to a Forgejo vault and a physically decoupled USB backpack as the ground truth.

---

*Part of the spiderweb cluster postmortem series.*