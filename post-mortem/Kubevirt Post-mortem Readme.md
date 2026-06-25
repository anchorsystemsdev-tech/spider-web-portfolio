# KUBE LOG: Kubevirt Post-mortem

[![Kubevirt Post Mortem](https://img.youtube.com/vi/Avd3B5hTdXk/maxresdefault.jpg)](https://www.youtube.com/watch?v=Avd3B5hTdXk)

 
**Cluster:** spiderweb | **Nodes:** M1 MacBook (Asahi Ubuntu ARM64) + HP Stream (Xubuntu x86_64) | **Duration:** 48 hours | **Status:** ✅ Resolved
 
---
 
## Incident Summary
 
| Field | Detail |
|---|---|
| **Incident** | Multi-architecture KubeVirt cluster build failure cascade across 7 distinct failure domains |
| **Environment** | Apple M1 Pro (16GB) → Asahi Ubuntu dual-boot as ARM64 control plane + HP Stream (2–4GB RAM, 32GB eMMC) as x86_64 worker node |
| **Stack** | K3s + KubeVirt + CDI (Containerized Data Importer) |
| **Architecture Challenge** | Bridging ARM64 (Apple Silicon) and x86_64 (Intel) in a single K3s cluster on consumer hardware |
| **Final VM** | Fedora 43 (ARM64) — 4 vCPU · 4Gi RAM · 15Gi PVC |
 
---
 
## Critical Configurations
 
Seven distinct failure domains hit in sequence. Each required a different root-cause resolution rather than iteration on the previous fix.
 
1. **The Architecture Wall** — macOS runs Darwin; K3s requires Linux cgroups. Colima and Multipass are dead ends for this use case. Asahi Ubuntu dual-boot is the only path to bare-metal KVM on M1.
2. **The Docker 401 Trap** — Container registries denied access; the ARM64 manifest for the demo image did not exist. Bypassed entirely via CDI DataVolume pulling directly from the OS source.
3. **The KVM Permission Boundary** — The QEMU pod ran as a limited user and was blocked from `/dev/kvm`. Required a direct host-level permissions grant.
4. **The CDI Storage Binding Deadlock** — `WaitForFirstConsumer` created an infinite chicken-and-egg loop on a single-node cluster. Required an immediate-binding annotation injection.
5. **The 16K Page Size Memory Alignment Crash** — The M1 host writes memory in 16K chunks. Standard ARM64 guest OSes compiled for 4K pages. QEMU self-destructed before it could log a single line. Required a full OS pivot to Fedora 43.
6. **The WebSocket Buffer Corruption** — Multi-line script paste through the KubeVirt API WebSocket silently dropped characters, corrupting `.bash_profile` and permanently disabling `getty@tty1.service`. Required string-by-string native injection and a full display manager overhaul.
7. **The HP Stream cgroups Kernel Gap** — The HP Stream's kernel didn't track resource usage via cgroups, causing the K3s agent to drop `NotReady` heartbeats. Required a GRUB bootloader rewrite and a 2GB swap file to keep the node alive.
---
 
## Phase 1 — Bare-Metal Bootstrap (Asahi Ubuntu on M1)
 
**The Problem:** K3s cannot run on macOS. The Darwin kernel has no cgroup or namespace support. Colima, Multipass, and Docker Desktop were all false paths — they create isolated NAT VMs that the HP Stream cannot route to over the local LAN. The only engineering-correct solution is stripping macOS out of the equation at the hardware level and flashing native Linux.
 
**Execution:**
 
```bash
# Execute in macOS terminal — non-destructively resizes the APFS container
# and provisions the m1n1 bootloader into the freed space
curl -sL https://ubuntuasahi.org/install | sh
```
 
After the script completes, the machine shuts down. From a powered-off state, hold the power button until "Loading startup options" appears. Select the new Ubuntu partition and click through the enrollment flow. The Apple Secure Enclave generates an ownership ticket for the Linux kernel, granting it hardware access at Exception Level 2.
 
**Page Size Diagnostic:**
 
```bash
# Run this after booting into Asahi Ubuntu to reveal the host's memory page size
getconf PAGE_SIZE
 
# On M1 Asahi Ubuntu this returns 16384 (16K) — Apple Silicon's native page size
# This value becomes the root cause of the QEMU crash in Phase 2 (Failure 4)
# A return of 4096 would mean standard ARM64 guest OSes work without issue
```
 
**K3s Bootstrap:**
 
```bash
# Install K3s — auto-detects ARM64
curl -sfL https://get.k3s.io | sh -
 
# Extract the physical LAN IP
ip a
 
# Extract the cryptographic join token (treat as a pre-shared key)
sudo cat /var/lib/rancher/k3s/server/node-token
```
 
**KubeVirt + CDI Install:**
 
```bash
# Deploy KubeVirt operator and custom resource
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
sudo kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml"
sudo kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml"
 
# Deploy Containerized Data Importer (CDI)
export CDI_VERSION=$(curl -s https://api.github.com/repos/kubevirt/containerized-data-importer/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
sudo kubectl create -f "https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_VERSION/cdi-operator.yaml"
sudo kubectl create -f "https://github.com/kubevirt/containerized-data-importer/releases/download/$CDI_VERSION/cdi-cr.yaml"
```
 
---
 
## Phase 2 — The VM Failure Cascade
 
### Failure 1: The Docker 401 Trap
 
**What Happened:** Attempted to pull `kubevirt/debian-container-disk-demo:latest` from Docker Hub. Got immediate `pull access denied... insufficient_scope: authorization failed`. The ARM64 manifest for this image does not exist on Docker Hub. Switching to `quay.io/kubevirt/debian-container-disk-demo:v1.1.0` returned the same 401 — the tag had no valid `linux/arm64` manifest on Quay either. The entire container registry approach was a dead end for this architecture.
 
**Resolution:** Bypassed all container registries. Used CDI's `DataVolume` to pull the official Debian cloud image directly from Debian's servers.
 
```yaml
# This annotation forces immediate storage binding — critical for single-node clusters
# Without it: WaitForFirstConsumer creates an infinite deadlock (Failure 2 below)
cdi.kubevirt.io/storage.bind.immediate.requested: "true"
 
source:
 http:
   url: "https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-arm64.qcow2"
```
 
---
 
### Failure 2: The CDI Storage Binding Deadlock
 
**What Happened:** DataVolume showed no Phase — completely blank. The provisioner was stuck in `WaitForFirstConsumer`.
 
**The Mechanism:** The `local-path` storage class waits to see which node a pod schedules onto before it binds the volume. The pod refuses to start because the volume isn't bound yet. On a single-node cluster, this is a permanent deadlock — there is no scheduling ambiguity to resolve.
 
**Resolution:** Force immediate binding via the annotation above. Also required setting `local-path` as the default storage class:
 
```bash
kubectl annotate storageclass local-path storageclass.kubernetes.io/is-default-class=true --overwrite
```
 
---
 
### Failure 3: The KVM Permission Boundary (virt-handler Death Loop)
 
**What Happened:** VM deployed and immediately got stuck in `Scheduling` state. The QEMU pod — which actually starts the VM — was running as a limited user. It was knocking on the M1's hardware door and the host Linux system was returning `access denied`. It physically could not reach `/dev/kvm`.
 
**Resolution:**
 
```bash
# Grant all users access to the KVM device
# Note: resets after kernel module reloads on some Asahi builds — re-run if VM gets stuck in Scheduling
sudo chmod 666 /dev/kvm
 
# Verify the device is accessible
ls -l /dev/kvm
```
 
---
 
### Failure 4: The 16K Page Size Memory Alignment Crash / Missing PID Error
 
**What Happened:** After resolving the registry and storage issues, QEMU was dying before it could log a single line. It couldn't even create a PID file — the "Missing PID" error.
 
**Root Cause:** The M1 host's memory subsystem is physically hardwired to write memory in 16K chunks (`getconf PAGE_SIZE` = 16384). Standard Debian ARM64 cloud images are compiled expecting 4K pages. The virtualization engine attempted to map a 16K host page into 4K-sized guest memory slots — they violently miss each other and QEMU self-destructs instantly.
 
**The Fix: Pivot to Fedora 43**
 
Fedora 43's `aarch64` cloud image kernel is built page-size-agnostic. It dynamically detects that the host is running at 16K and bridges the gap natively without software emulation. This is one of the very few ARM64 distributions that explicitly supports 16K memory alignment.
 
**The Verified Fedora 43 YAML (ARM64 Mandatory Fields):**
 
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
 name: <YOUR_VM_NAME>
spec:
 runStrategy: Always
 template:
   metadata:
     labels:
       kubevirt.io/domain: <YOUR_VM_NAME>
   spec:
     domain:
       # MANDATORY: ARM64 has no q35 bus — 'virt' is the only valid machine type
       machine:
         type: virt
       # MANDATORY: host-passthrough lets Fedora see M1 Pro cores directly
       cpu:
         model: host-passthrough
         cores: <CPU_CORES>
       resources:
         requests:
           memory: <MEMORY_REQUEST>
           cpu: "<CPU_REQUEST>"
         limits:
           memory: <MEMORY_LIMIT>
           cpu: "<CPU_LIMIT>"
       features:
         smm:
           enabled: false
       devices:
         disks:
         - disk:
             bus: virtio
           name: datavolume
         - disk:
             bus: virtio
           name: cloudinitdisk
         interfaces:
         - name: default
           masquerade: {}
       # MANDATORY: Apple Silicon has no legacy BIOS — EFI boot is required
       firmware:
         bootloader:
           efi:
             secureBoot: false
     networks:
     - name: default
       pod: {}
     volumes:
     - name: datavolume
       dataVolume:
         name: <YOUR_DV_NAME>
     - name: cloudinitdisk
       cloudInitNoCloud:
         userData: |-
           #cloud-config
           user: <YOUR_VM_USER>
           password: <YOUR_VM_PASSWORD>
           chpasswd: { expire: False }
           ssh_pwauth: True
 dataVolumeTemplates:
 - metadata:
     name: <YOUR_DV_NAME>
     annotations:
       cdi.kubevirt.io/storage.bind.immediate.requested: "true"
   spec:
     storage:
       accessModes:
       - ReadWriteOnce
       volumeMode: Filesystem
       resources:
         requests:
           storage: <STORAGE_SIZE>
       storageClassName: local-path
     source:
       http:
         # Fedora 43 stable — 16K page-size compatible, bypasses metalink redirects
         url: "https://mirror.cs.odu.edu/fedora/linux/releases/43/Cloud/aarch64/images/Fedora-Cloud-Base-Generic-43-1.6.aarch64.qcow2"
```
 
**Launch Sequence:**
 
```bash
# Enable required KubeVirt feature gates
kubectl patch kubevirt kubevirt -n kubevirt --type merge --patch \
 '{"spec":{"configuration":{"developerConfiguration":{"featureGates":["DataVolumes","Snapshot"]}}}}'
 
# Re-grant KVM access
sudo chmod 666 /dev/kvm
 
# Apply and watch the DataVolume import progress
kubectl apply -f <YOUR_VM_MANIFEST>.yaml
kubectl get dv -w
# Expected: ImportInProgress → Succeeded (100.0%)
 
# Confirm VM is running
kubectl get vmi
 
# Connect to graphical console
virtctl vnc <YOUR_VM_NAME>
```
 
---
 
### Failure 5: The WebSocket Buffer Corruption / Black Screen of Death
 
**What Happened:** Attempted to configure the desktop environment by pasting multi-line bash scripts through the KubeVirt API WebSocket terminal to build an auto-login hack (intercepting `getty@tty1` and firing `startx` from `.bash_profile`). Repeated paste attempts through the websocket buffer corrupted the file with multiple malformed `if` statements (`-bash: [: missing ']'`). Bash crashed instantly on every login attempt. Systemd hit its `StartLimitBurst` failure threshold and permanently disabled `getty@tty1.service`. VNC showed: pitch black screen, single blinking cursor, no response to keyboard input.
 
**The SSH Diagnostic Path:**
 
```bash
# On the Ubuntu host — purge the dead VM's SSH fingerprint first
# (new VM spun up = new ED25519 key = host key verification failed)
ssh-keygen -f '/home/<YOUR_HOST_USER>/.ssh/known_hosts' -R 'vmi.<YOUR_VM_NAME>.default'
 
# SSH into the VM via KubeVirt tunnel (bypasses the dead VNC entirely)
virtctl ssh <YOUR_VM_USER>@vmi/<YOUR_VM_NAME>
 
# Inside the VM — annihilate the corrupted bash profile
rm -f ~/.bash_profile
 
# Restore factory defaults from the skeleton directory
cp -f /etc/skel/.bash* ~/
 
# Reset the crashed systemd service lock
sudo reboot
```
 
**The Architectural Fix:** Abandoned the `.bash_profile` auto-login hack entirely. The correct solution is a proper display manager. Installed Sway (Wayland tiling WM) + `sddm` as the display manager. `sddm` handles the boot sequence natively without fragile script injections.
 
```bash
# Run line by line — never paste multi-line scripts through KubeVirt WebSocket
sudo dnf install -y sway waybar alacritty sddm wl-clipboard spice-vdagent qemu-guest-agent
sudo systemctl set-default graphical.target
sudo systemctl enable sddm qemu-guest-agent
 
# Security hardening — cloud-init injects NOPASSWD by default, remove it
sudo passwd -l root
sudo find /etc/sudoers.d/ -type f -exec sed -i 's/NOPASSWD://g' {} +
 
# Set firewall to default-deny (drop zone = VM is cryptographically invisible on the network)
sudo firewall-cmd --set-default-zone=drop
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=vnc-server
sudo firewall-cmd --reload
 
sudo reboot
```
 
---
 
## Phase 3 — VM Hardening & Configuration
 
**Terminal Setup (Fedora VM):**
 
```bash
# Starship prompt — DNF doesn't carry it, install direct from source
curl -sS https://starship.rs/install.sh | sh
echo 'eval "$(starship init bash)"' >> ~/.bashrc
 
# Alacritty terminal config
mkdir -p ~/.config/alacritty
cat << 'EOF' > ~/.config/alacritty/alacritty.toml
[font]
size = 12.0
 
[font.normal]
family = "Fira Code"
style = "Regular"
 
[window]
opacity = 0.85
padding = { x = 10, y = 10 }
EOF
 
# Browser and fonts
sudo dnf config-manager addrepo --from-repofile=https://repo.librewolf.net/librewolf.repo
sudo dnf install -y librewolf fira-code-fonts swaybg
 
# Terminal visualizers
sudo dnf install -y fastfetch btop cmatrix
```
 
**Snapshot:**
 
```bash
# Snapshot requires the feature gate patch from Phase 2
kubectl apply -f <YOUR_SNAPSHOT_MANIFEST>.yaml
kubectl get vmsnapshot
# Expected: <SNAPSHOT_NAME> | Succeeded | READYTOUSE: true
```
 
**Asahi Ubuntu Host Hardening:**
 
```bash
# Remove desktop indexer — frees ~200MB of idle RAM for K3s and VMs
sudo apt purge tracker tracker-miner-fs -y
 
# Remove Snap daemon — frees ~500MB
sudo systemctl mask snap.snapd
sudo apt purge snapd -y
sudo apt autoremove -y
sudo apt clean
```
 
---
 
## Phase 4 — HP Stream Worker Node
 
**The Problem:** The HP Stream's Xubuntu kernel did not track resource usage via cgroups. Without cgroups, the K3s scheduler cannot monitor the node's health. The kubelet heartbeat was dropping, causing the node to oscillate between `Ready` and `NotReady`.
 
Additionally: 2–4GB of RAM is barely enough to run the OS and the K3s agent simultaneously. A swap file was required as a safety buffer to prevent the OOM killer from terminating cluster processes.
 
**Execute on HP Stream Terminal:**
 
```bash
# Inject cgroup parameters into the GRUB bootloader
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory /g' /etc/default/grub
sudo update-grub
 
# Carve out a 2GB swap file as overflow RAM
sudo fallocate -l 2G /swapfile
 
# Lock the swap file (only root kernel process can read it — Least-Privilege)
sudo chmod 600 /swapfile
 
# Format and enable swap
sudo mkswap /swapfile
sudo swapon /swapfile
 
# Persist across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
 
sudo reboot
```
 
**Join the Cluster (Execute on HP Stream):**
 
```bash
# Replace <CONTROL_PLANE_IP> and <NODE_TOKEN> with values from the control plane
curl -sfL https://get.k3s.io | K3S_URL=https://<CONTROL_PLANE_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```
 
**Verify from Control Plane:**
 
```bash
kubectl get nodes -o wide
# Expected: <CONTROL_PLANE_HOSTNAME> (control-plane, Ready) + <WORKER_HOSTNAME> (Ready)
```
 
**Ghost Node Cleanup:**
 
A misconfigured entry appeared in the cluster index from an earlier failed join attempt. Manually deleted:
 
```bash
kubectl delete node <GHOST_NODE_NAME>
```
 
**HP Stream Storage Maintenance:**
 
```bash
# Cap journald log growth to prevent eMMC exhaustion
sudo journalctl --vacuum-size=100M
sudo sed -i 's/#SystemMaxUse=/SystemMaxUse=100M/g' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
 
# Prune dead K3s container images
sudo k3s crictl rmi --prune
```
 
---
 
## Final Cluster State
 
| Component | Namespace | Type | Status | Node |
|---|---|---|---|---|
| `<CONTROL_PLANE_HOSTNAME>` | — | K3s Node (ARM64) | Ready · Control Plane | — |
| `<WORKER_HOSTNAME>` | — | K3s Node (x86_64) | Ready | — |
| `<YOUR_VM_NAME>` | default | VirtualMachineInstance | Running · `<VM_IP>` | `<CONTROL_PLANE_HOSTNAME>` |
| `virt-api` | kubevirt | Pod | 1/1 Running | — |
| `virt-controller` (×2) | kubevirt | Pod | 1/1 Running | — |
| `virt-handler` | kubevirt | Pod | 1/1 Running | — |
| `virt-operator` (×2) | kubevirt | Pod | 1/1 Running | — |
| `cdi-apiserver` | cdi | Pod | 1/1 Running | — |
| `cdi-deployment` | cdi | Pod | 1/1 Running | — |
| `cdi-operator` | cdi | Pod | 1/1 Running | — |
| `cdi-uploadproxy` | cdi | Pod | 1/1 Running | — |
| `<YOUR_DV_NAME>` | default | PersistentVolumeClaim | Bound · ~16GB | local-path |
 
**Node Resource Utilization at Stable State:**
 
| Node | CPU Usage | CPU % | Memory Usage | Memory % |
|---|---|---|---|---|
| `<CONTROL_PLANE_HOSTNAME>` | 453m | 4% | 8438Mi | 54% |
 
---
 
## Lessons
 
**1. Prompt engineering AI to use real sources — not its training data.**
Every hallucinated YAML field in this build (`spec.template.spec.domain.devices.channels`, `video.modelType`, wrong Fedora mirror URLs) came from the model synthesizing from memory rather than reading the actual KubeVirt API reference. The fix was always the same: force it to search and read the documentation before generating config.
 
**2. Developers should still audit documentation themselves rather than solely relying on AI assistants.**
Even when the AI was pointed at the right source, it still conflated ARM64-specific fields with x86 defaults and missed page-size constraints entirely. Reading the KubeVirt ARM64 operations guide directly — not a summary of it — was what isolated the `machine: { type: virt }`, EFI bootloader, and `host-passthrough` requirements. The AI surfaced the search path; the engineer had to verify it.
 
**3. Never paste multi-line scripts through WebSocket terminals.**
WebSocket buffers are not reliable clipboard channels. A single dropped `]` character brought down an entire display stack and required an SSH-based recovery from scratch. Inject configuration string-by-string, or use a proper display manager to eliminate the failure surface entirely.
 
**4. Hardware constraints force better architecture than unlimited compute ever does.**
Data centers solve 16K page size incompatibility by provisioning a different instance type. This build required understanding why QEMU was dying before its first log line, tracing it to memory alignment at the kernel level, and identifying the exact OS distribution compiled to handle it. A 2GB swap file on a constrained laptop kept a production Kubernetes worker node stable. Resource optimization under real constraint is a higher-order skill than dashboard provisioning.
 
---
 
## Skills
 
**ARM64 KVM Virtualization on Apple Silicon**
Navigated Apple's hardware security chain (1TR → m1n1 → U-Boot → GRUB) to unlock bare-metal KVM access. Understood the 16K page size constraint at the hardware level and identified the exact OS distribution built to bridge the host/guest memory alignment gap natively.
 
**Kubernetes Control Plane Operation**
Bootstrapped a K3s multi-architecture cluster from scratch. Operated KubeVirt and CDI operators, patched feature gates (`DataVolumes`, `Snapshot`), managed PersistentVolumeClaims and StorageClasses, and debugged scheduler behavior across ARM64 and x86_64 nodes simultaneously.
 
**Container Registry Architecture and Failure Mode Diagnosis**
Identified why ARM64 manifests fail on Docker Hub and Quay, understood multi-arch image manifest lists at the registry protocol level, and bypassed broken registries entirely by routing through CDI's DataVolume HTTP importer pulling directly from OS maintainers.
 
**Memory Architecture at the Hypervisor Layer**
Traced a QEMU crash that produced no logs to an M1-specific 16K memory page size mismatch. Understood why the virtualization engine self-destructs before PID creation when host and guest page sizes collide, and selected the correct guest OS based on kernel-level memory alignment support.
 
**Systemd Service Recovery Under Crash Loops**
Hit `StartLimitBurst` on `getty@tty1.service` due to a corrupted login script. Bypassed the dead VNC console via the KubeVirt SSH tunnel, diagnosed the failure through journald, and rebuilt the display stack using a proper display manager rather than fragile `.bash_profile` injection.
 
**GRUB Bootloader Configuration for Kubernetes Compatibility**
Rewrote the HP Stream's boot configuration to enable cgroup CPU and memory tracking at the kernel level — a prerequisite for K3s agent health reporting that standard desktop Linux installs don't enable by default.
 
**Zero-Trust Security Posture on Guest VMs**
Removed the cloud-init `NOPASSWD` sudoers bypass that Fedora's qcow2 image injects by default. Locked the root account, set firewalld to `drop` zone, and opened only the minimum required ports (SSH, VNC) before putting the VM into active use.
 
**Edge Resource Optimization**
Removed Snap daemon and the tracker indexer from the Asahi Ubuntu host to reclaim ~700MB of idle RAM for cluster workloads. Added 2GB swap to the HP Stream to prevent OOM kills on a constrained edge node. Applied journald caps to prevent eMMC exhaustion on a 32GB drive running a Kubernetes agent.
 
**KubeVirt YAML Schema and Strict Decoding Error Diagnosis**
Resolved repeated `strict decoding error: unknown field` rejections by distinguishing between KubeVirt API v1 schema constraints and common-but-wrong field names (`video.modelType` vs `video.type`, `channels` not supported in certain ARM64 builds). Understood that feature gates gate entire API surface areas, not just runtime behavior.
 
**Wayland Display Stack on Virtualized Hardware**
Deployed Sway + SDDM on a KubeVirt-managed Fedora 43 ARM64 VM accessed over a VNC WebSocket. Understood the difference between X11 auto-login hacks (fragile over WebSocket) and a proper display manager handoff (stable regardless of how the console session is established).
 
---
 
*Part of the spiderweb cluster postmortem series.*

