# Ani-web post-mortem 

[![
Ani-web Post-Mortem](https://img.youtube.com/vi/tN2Szm73Mg0/maxresdefault.jpg)](https://www.youtube.com/watch?v=tN2Szm73Mg0)
 
> **Cluster:** spiderweb | **Hardware:** M1 MacBook | **Build Env:** Kali Linux VM (UTM/macOS) → **Production:** Asahi-Ubuntu (dual-boot, same machine) | **Status:** ✅ Resolved
 
---
 
## Incident Summary
 
| Field | Detail |
|---|---|
| **Application** | ani-web (self-hosted anime media library) |
| **Incident Type** | Monorepo Compilation Failure + SQLite Runtime Crash |
| **Root Cause 1** | `--prefix` isolation bypassed monorepo hoisting, starving the compiler |
| **Root Cause 2** | Node.js v20 base image lacks native `node:sqlite` built-in module |
| **Environment** | M1 MacBook — Kali VM (UTM/macOS) → Docker → Asahi-Ubuntu (dual-boot) |
| **Resolution** | Standard root-level `npm install` per repo docs + Node 22 base image swap |
 
---
 
## Critical Configuration
 
- **Node.js runtime version** — Native SQLite requires `v22.5.0` or higher. Node 20 does not ship with it.
- **`npm install` root-path targeting** — Using `--prefix` to target subdirectories in a monorepo bypasses the root-level dependency hoisting the architecture requires.
---
 
## Phase 1 — The Setup
 
**Objective:** Containerize ani-web from a Kali Linux VM running inside macOS on an M1 MacBook, and migrate it cleanly into the Asahi-Ubuntu dual-boot environment on the same machine (spiderweb cluster).
 
ani-web is a **monorepo** — a single repository where the frontend client and backend server live together under a shared root, sharing dependencies and build tooling at the root level. This architecture requires specific handling at install time.
 
The operator built a **hazmat suit pipeline** to isolate the host machine from potentially malicious npm post-install scripts:
 
1. Nuke local `node_modules`
2. Force all dependency installation and compilation entirely inside a Docker container
3. Validate locally before pushing to production
The strategy was sound. The implementation is where things cascaded.
 
---
 
## Phase 2 — The npm Monorepo Trap
 
### What Happened
 
The build failed immediately. Vite disappeared. TypeScript threw a wall of errors claiming it couldn't resolve core modules.
 
```
error TS2304: Cannot find name 'process'
error TS2304: Cannot find name 'Buffer'
error: Could not resolve entry module 'index.html'
```
 
Classic starved compiler. The toolchain had nothing to work with.
 
### Why It Failed
 
To harden the pipeline, the operator used `--ignore-scripts` and `--prefix` flags to restrict dependency installation to isolated client and server subdirectories. In a standard single-package repo, this works fine. In a monorepo, it's a trap.
 
Monorepos rely on **hoisting** — root-level installs that pull shared core dependencies (TypeScript, Vite, etc.) up to the repository root where both the client and the server can access them. The `--prefix` flag bypasses the root entirely. The subfolders got their scoped packages. The root got nothing. The compiler had no foundation.
 
### The Fix
 
Drop the custom flags. Run the standard root-level install exactly as documented in the repository.
 
```bash
# What the operator ran (broken)
npm install --ignore-scripts --prefix ./client
npm install --ignore-scripts --prefix ./server
 
# What the repo documentation actually specified (correct)
npm install
```
 
The monorepo hoisted correctly. Build passed clean.
 
---
 
## Phase 3 — The SQLite Runtime Crash
 
### What Happened
 
The container compiled clean. It booted and immediately crashed.
 
```
Error [ERR_UNKNOWN_BUILTIN_MODULE]: No such built-in module: node:sqlite
    at internalBinding (<anonymous>)
    at NativeModule.load (node:internal/bootstrap/realm:397:27)
```
 
### Why It Failed
 
The Dockerfile was using `node:20-alpine` as the base image. Node.js 20 does not ship with a native SQLite engine. The ani-web developer wrote the application against the `node:sqlite` built-in introduced natively in **Node.js v22.5.0**. The app expected a runtime capability that v20 simply doesn't have. The server had no fallback — it panicked and died on boot.
 
### The Fix
 
One line in the Dockerfile.
 
```dockerfile
# Before
FROM node:20-alpine
 
# After
FROM node:22-alpine
```
 
Server booted clean.
 
---
 
## Phase 4 — The Air Gap Bridge (Production Deployment)
 
With a verified, working container, the operator transported it into the spiderweb cluster without ever touching the host machine directly. The full pipeline spans two isolated OS environments on the same M1 MacBook — the Kali VM (UTM/macOS side) as the build node and Asahi-Ubuntu (dual-boot side) as the final destination — with GHCR serving as the public transit layer between them.
 
### Step 1 — Build on the Kali VM
 
The fixed image is built entirely inside the Kali VM. The host machine never touches the compiled output.
 
```bash
# Tag directly for GHCR at build time so the image is push-ready
docker build -t ghcr.io/<username>/ani-web:latest .
```
 
### Step 2 — Push to GHCR
 
GHCR acts as the air gap bridge — a public transit layer that neither OS environment has to reach across a local network to access.
 
```bash
# Authenticate with a GitHub personal access token (write:packages scope)
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin
 
# Push the image up
docker push ghcr.io/<username>/ani-web:latest
```
 
### Step 3 — Pull on Asahi-Ubuntu (dual-boot)
 
Boot into the Asahi-Ubuntu side of the M1 MacBook to enter the spiderweb cluster's production environment. Podman is used here instead of Docker — no daemon required, rootless by default.
 
```bash
# Authenticate Podman with GHCR
podman login ghcr.io -u <username> --password $GITHUB_TOKEN
 
# Pull the verified image down
podman pull ghcr.io/<username>/ani-web:latest
```
 
### Step 4 — Vault into Forgejo
 
The image is re-tagged and pushed into the self-hosted Forgejo registry for permanent, private storage. From this point forward, the cluster pulls from the internal vault — not from GHCR.
 
```bash
# Re-tag for the internal Forgejo registry
podman tag ghcr.io/<username>/ani-web:latest <forgejo-host>/<namespace>/ani-web:latest
 
# Push into the vault
podman push <forgejo-host>/<namespace>/ani-web:latest
```
 
The four-phase plan addressed three distinct goals simultaneously: transport a compiled image across two completely different operating system kernels dual-booted on the same physical machine — Kali VM (UTM/macOS) to Asahi-Ubuntu Linux — without direct partition access; contain the npm install process inside Docker to shield both environments from potentially malicious scripts; and vault the final image in a self-hosted Forgejo registry rather than remaining dependent on GitHub Packages as the permanent source of truth.
 
---
 
## Lessons
 
**1. Prompt engineer your AI assistant — force it to the source.**
This entire restoration was executed with an AI assistant in the loop. The real lesson wasn't just "read the docs" — it was learning to explicitly force the AI to search and pull from actual repository documentation rather than generating scripts from memory. Left unprompted, AI assistants will confidently produce plausible-looking configs that don't match the real codebase. The operator had to actively direct the AI to use web search against the live repo instead of letting it freewheel. That prompt discipline is what unlocked the correct build path.
 
**2. Developers should still audit the documentation themselves.**
Even with an AI assistant doing the research, don't outsource the verification entirely. AI can retrieve and summarize documentation, but the developer needs to read it and confirm the approach makes sense for their specific architecture. Sole reliance on an AI's interpretation is how subtle mismatches — like monorepo hoisting requirements — slip through.
 
**3. Version alignment is non-negotiable.**
If the code targets a bleeding-edge runtime feature, the Dockerfile base image has to match. Node.js v22 introduced native SQLite as a built-in. Deploying on v20 will always fail at runtime regardless of how clean the build is. Check runtime requirements before you write your Dockerfile.
 
**4. Multi-stage Docker builds are a valid hazmat strategy — with the right flags.**
Compiling dependencies inside a container to protect the host from malicious npm scripts is a legitimate and solid approach. The failure wasn't the strategy. It was the flags used to implement it.
 
---
 
## Skills
 
**Monorepo architecture & dependency hoisting**
Gained a working understanding of how npm hoisting functions in a monorepo — specifically why shared dependencies like TypeScript and Vite must resolve at the root level and how targeting subdirectories with `--prefix` silently breaks that contract. Reading a repo's structure before touching its install process is now a concrete habit, not an afterthought.
 
**Container image lifecycle management**
Executed a full build → tag → push → pull → retag → vault pipeline end to end across two registries (GHCR and Forgejo) using both Docker and Podman. Reinforced the practical difference between using a public registry as a transit layer versus a permanent source of truth.
 
**Runtime environment diagnosis**
Developed the ability to read a crash log and trace a fatal error back to a base image version mismatch rather than the application code itself. `ERR_UNKNOWN_BUILTIN_MODULE` is a runtime environment problem, not a code problem — that distinction now registers immediately.
 
**Cross-OS image transport**
Built a repeatable workflow for moving a verified container image between two completely isolated OS environments on the same physical machine without direct partition access. GHCR as a neutral transit layer is now a tool in the kit, not just a hosting option.
 
**AI-assisted debugging & prompt engineering**
Actively directed an AI assistant to pull from live repository documentation rather than generating output from training data. Learned that the quality of AI assistance in debugging is directly tied to how precisely you constrain its research behavior — vague prompts produce confident hallucinations, targeted prompts with explicit search requirements produce usable output.
 
---
 
*Part of the spiderweb cluster postmortem series.*