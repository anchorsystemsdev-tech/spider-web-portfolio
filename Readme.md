# 🕷️🕸️ Spider-Web Cluster — Infrastructure Portfolio
 
A hyper-converged, multi-architecture Kubernetes cluster engineered from consumer hardware. Built to function as a sovereign, self-healing digital ecosystem — running AI inference, containerized workloads, virtual machines, and a fully automated GitOps pipeline across three physically mismatched nodes.
 
---
 
## Hardware
 
The cluster runs on a strict hardware split across two instruction sets, maximizing resource efficiency across ARM64 and x86_64.
 
| Role | Identity | Architecture | Specs |
|---|---|---|---|
| Control Plane | `gbuntu` | ARM64 | Apple M1 Pro · 16GB Unified Memory · Bare-metal Asahi Ubuntu |
| Worker Node 1 | `hp-stream` | x86_64 | Intel Celeron · 4GB RAM + 2GB Encrypted Swap · Xubuntu |
| Worker Node 2 | `mini-pc` | x86_64 | Intel N100 · 16GB RAM · Windows 11 / WSL2 ("The Tank") |
 
---
 
## Architecture
 
### Orchestration Plane
- **K3s** manages all containerized and virtualized workloads across the heterogeneous node pool
- **KubeVirt** runs bare-metal Fedora 43 Virtual Machines as native Kubernetes pods, bypassing macOS's hardware lock on nested virtualization entirely
- `nodeSelector` and `nodeAffinity` rules pin heavy compute to capable nodes, preventing the HP Stream from being OOM-killed by the scheduler
### Nervous System — Telemetry & State
A 3-lane Redis highway handles real-time telemetry and state caching across the cluster:
 
| Lane | Node | Function |
|---|---|---|
| Light | HP Stream | Minimal "Alive" heartbeats for edge nodes |
| Medium | Mini PC | Intermediate state caching |
| Powerhouse | M1 Mac | 2GB RAM dedicated to deep state data and infrastructure logs |
 
### Data Mesh — Logistics & Transport
- **LocalSend REST API** — Event-driven "Just-in-Time" data mesh. The n8n control plane fires binary payloads directly across nodes, bypassing the overhead of traditional shared storage
- **FriendNet** — A decentralized Go-based P2P file-sharing mesh acting as the cluster's subconscious memory. Sidecar clients passively sync logs and datasets across nodes
### Sovereign Storage Plane
- **Inside-Out Edge Vault** — A 114GB / 128GB USB 3.0 drive formatted to ext4, mounted directly to the HP Stream worker node. Physically decoupled from internal eMMC storage — survives node failure
- **Forgejo Vault** — Self-hosted Git server running on the edge vault, storing the cluster's Infrastructure-as-Code DNA: YAML manifests and n8n workflow JSON backups
---
 
## AI Stack
 
Inference is handled by Ollama with models pinned to hardware-optimized nodes:
 
| Model | Node | Task |
|---|---|---|
| Gemma 4 | M1 Mac (Unified Memory) | High-speed text reasoning |
| Qwen 3.5 | Mini PC /  | Vision and multimodal tasks |
 
---
 
## Automation Pipeline
 
This public repository is the sanitized output of a private Forgejo instance, published through a fully autonomous n8n pipeline:
 
1. **Private Source Control** — All IaC and configuration files live in a private self-hosted Forgejo Git server
2. **n8n Orchestration** — A custom workflow monitors private Forgejo repositories for changes and triggers the pipeline on every push
3. **Secure Sanitization** — Before any public exposure, an automated module scrubs the code via regex — stripping internal IPs, API keys, credentials, and proprietary configurations
4. **AI Documentation** — A Google Gemini agent analyzes the sanitized code and commit context, then auto-generates or updates README files and architectural overviews
5. **GitHub Publication** — The cleaned, documented output is pushed to this public repository via the GitHub API
 
---
 
## KUBE LOG — Post-Mortems
 
Each major incident in the cluster's build history is documented as a first-person technical post-mortem with a companion video. These aren't tutorials — they're failure reports.
 
| Incident | Core Failure |
|---|---|
| **KubeVirt HCI Build** | 16K page size memory alignment crash — M1 host kills QEMU before first log line |
| **Linux Kernel CVE Patch** | CVE-2026-31431 + CVE-2026-43284 across three architecturally incompatible nodes |
| **FriendNet Data Mesh** | Silent exit 0 cross-compilation failure + Git worktree wipe of 13,429 files |
| **LocalSend / n8n GitOps** | 7-failure cascade across data pipeline amnesia, index locks, and cyclic object crashes |
| **ani-web Containerization** | npm monorepo trap + Node.js version mismatch causing SQLite runtime crash |
 

## Repository Structure
The architecture of this repository is organized to provide clear separation of concerns, facilitating modularity and reflecting the exact deployments running in the K3s environment, alongside the automation workflows that manage them.

```text
.
├── k3s-configs/                     # Core Kubernetes configurations and manifests
│   ├── custom-browser/              
│   │   ├── browser-service.yaml
│   │   ├── firefox.yaml
│   │   ├── pv.yaml
│   │   ├── pvc.yaml
│   │   └── README.md
│   ├── forgejo/                     
│   │   ├── forgejo-deployment.yaml
│   │   ├── forgejo-ingress.yaml
│   │   └── README.md
│   ├── friendnet/                   
│   │   ├── build-friendnet-mac.yaml
│   │   ├── friendnet-edge-builder.yaml
│   │   ├── friendnet-server.yaml
│   │   ├── profile/
│   │   │   └── index.html
│   │   └── README.md
│   ├── media/                       # Media aggregation and streaming infrastructure
│   │   ├── spider-anime/            
│   │   │   └── ani-web/             
│   │   │       ├── ani-web-prod.yaml
│   │   │       ├── data/
│   │   │       │   └── sync_manifest.json
│   │   │       └── README.md   
│   │   └── stremio-web/             
│   ├── n8n/                         
│   │   ├── localsend-bridge.yaml
│   │   ├── n8n.yaml
│   │   └── README.md
│   ├── ollama/                      
│   │   ├── ollama.yaml
│   │   └── README.md
│   ├── open-notebook/               
│   │   ├── open-notebook.yaml
│   │   ├── surrealdb.yaml
│   │   └── README.md
│   ├── postgres/                    
│   │   ├── postgres.yaml
│   │   └── README.md
│   ├── redis/                       
│   │   ├── redis-deployment.yaml
│   │   ├── redis-powerhouse.yaml
│   │   ├── redis-service.yaml
│   │   ├── sentinel-config.yaml
│   │   ├── spider-web-grid.yaml
│   │   └── README.md
│   ├── searxng/                     
│   │   ├── searxng.yaml
│   │   └── README.md
│   ├── system/                      # Core cluster administrative scripts and configurations
│   │   ├── edge-vault-storage.yaml  
│   │   ├── mac-safety-net.yaml      
│   │   ├── resource-limits.yaml     
│   │   └── README.md
│   └── vms/                         # Virtual machine definitions and configs
│       ├── fedora-desktop.yaml
│       ├── fedora-lab.yaml
│       └── README.md
├── Workflow-Backup/                 # n8n automation pipeline and agent backups
│   ├── Creator_workflow-v2.json
│   ├── Creator_workflow-v2.md
│   ├── Creator_workflow-v3_with_Logging.json
│   ├── Creator_workflow-v3_with_Logging.md
│   ├── Editor_workflow-v4.json
│   ├── Editor_workflow-v4.md
│   ├── GitHub_Backup.json
│   ├── GitHub_Backup.md
│   ├── Google_Maps_Email_Scraper.json
│   ├── Google_Maps_Email_Scraper.md
│   ├── infra-backup.json
│   ├── infra-backup.md
│   ├── Localsend-Beam.json
│   ├── Localsend-Beam.md
│   ├── Research_Agent_Sub-Workflow.json
│   └── Research_Agent_Sub-Workflow.md
├── post-mortems/                    # Incident reports
└── Readme.md                        # Master Portfolio documentation

## Tech Stack
 
`K3s` `KubeVirt` `CDI` `QEMU` `virtctl` `n8n` `LocalSend` `FriendNet` `Forgejo` `Redis` `PostgreSQL` `SurrealDB` `Ollama` `SearXNG` `Open Notebook` `systemd` `Asahi Linux` `Fedora 43` `Xubuntu` `WSL2` `Docker` `GHCR` `Go` `Python` `Node.js` `Bash` `JavaScript` `npm` `GitHub API` `Google Gemini` `Sway` `Podman` `Alacritty` `Pass` `Starship` `and more...` 
 
---
 
## Portfolio Value
 
The Spider-Web is an elite-grade DevSecOps simulator built under real resource constraints. It demonstrates:
 
- **Heterogeneous Distributed Systems** — ARM64 and x86_64 nodes operating as a single unified cluster
- **Kernel-Level Debugging** — 16K page size alignment, VXLAN packet loss from hardware checksum offloading, cgroup kernel parameter injection
- **Zero-Day CVE Patching** — Triaged and patched live Linux kernel privilege escalation vulnerabilities across three architecturally incompatible nodes simultaneously, each requiring a distinct remediation strategy
- **Zero-Trust GitOps** — Trifecta automation loop (Creator → Editor → Backup) managing the cluster's own brain
- **Hardware Sovereignty** — No cloud dependencies. Full stack runs on recycled consumer hardware with a $15 USB drive as the disaster recovery layer
