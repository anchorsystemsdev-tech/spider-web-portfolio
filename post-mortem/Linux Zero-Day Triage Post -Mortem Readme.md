# Linux Zero-Day Triage Post-Mortem
 
[![Linux Zero-Day Triage Post -Mortem](https://img.youtube.com/vi/fhhKWqMmgM8/maxresdefault.jpg)](https://www.youtube.com/watch?v=fhhKWqMmgM8)

> **Cluster:** spiderweb | **Hardware:** M1 MacBook + HP Stream + Windows Mini PC | **Threat Level:** Critical | **Status:** ✅ Fully Mitigated
 
---
 
## Incident Summary
 
| Field | Detail |
|---|---|
| **CVEs** | CVE-2026-31431 (Copy Fail), CVE-2026-43284 & CVE-2026-43500 (Dirty Frag) |
| **Incident Type** | Universal Linux Local Privilege Escalation (LPE) — AI-discovered zero-days |
| **Affected Scope** | All Linux distributions running a kernel updated since 2017 |
| **Exploit Mechanism** | `splice()` system call — shared attack vector; Copy Fail exploits AEAD scatter list reuse via AF_ALG socket; Dirty Frag exploits nonlinear socket buffer (SKB) fragment list handling |
| **Architecture** | x86_64 Xubuntu (HP Stream) + ARM64 Asahi Ubuntu (M1 MacBook) + x86_64 WSL2 (Windows Mini PC) |
| **Resolution** | Three architecturally distinct mitigations deployed across three nodes — no two fixes were the same |
 
---
 
## Critical Configuration
 
- **AF_ALG socket interface & AEAD module** — The shared exploit path for Copy Fail across all three nodes. How each node handles this module at the kernel level determines which mitigation strategy is possible.
- **`splice()` system call** — The shared attack vector across both CVEs. In both cases the attacker uses `splice()` to poison the kernel page cache, causing it to write attacker-controlled bytes into a target file's memory without touching the disk.
- **Distinct root flaws per CVE** — Copy Fail: AEAD scatter list reuse via the AF_ALG socket interface — no SKBs involved. Dirty Frag: a logic flaw in how the kernel handles nonlinear socket buffers (SKBs) that skips Copy-on-Write protection during in-place decryption of fragment lists.
- **Node kernel ownership** — The core variable that splits the three mitigations. HP Stream owns its kernel. Asahi Ubuntu owns a custom-fused kernel. WSL2 does not own its kernel at all — Windows does.
---
 
## Section 1 — Universal Zero-Day Threats
 
### Copy Fail (CVE-2026-31431)
 
A logic flaw in the kernel's AEAD (Authenticated Encryption with Associated Data) cryptographic subsystem, introduced by a 2017 performance optimization. The flaw uses the same memory reference for both the input and output scatter lists during an HMAC computation. By leveraging the `splice()` system call, an attacker can append the file descriptor of a read-only SUID binary directly into the socket's input scatter list. The shared reference causes the kernel to inadvertently write attacker-controlled bytes into that file's page cache. Repeated execution overwrites the binary in memory with shellcode, which then executes as root.
 
**Error signature:**
```
AF_ALG socket → splice() → page cache overwrite → SUID binary → instant root
```
 
### Dirty Frag (CVE-2026-43284 & CVE-2026-43500)
 
A follow-on pair of vulnerabilities exploiting the same `splice()` + page cache poisoning primitive, but targeting nonlinear socket buffers (SKBs) across two distinct kernel networking paths.
 
| Variant | CVE | Path | Mechanism | Patch Status |
|---|---|---|---|---|
| **ESP** | CVE-2026-43284 | IPsec ESP receive path | Overwrites SUID binaries via out-of-bounds writes | ✅ Patched — May 7, 2026 (commit `f4c50a4034e6`) |
| **RXRPC** | CVE-2026-43500 | AFS backend protocol | Brute-forces decryption key to zero out root password field in `/etc/passwd` page cache | ⚠️ Unpatched in most standard repositories |
 
### Why These Keep Appearing
 
Both vulnerabilities were found by AI systems scanning the Linux codebase for deterministic logic flaws — bugs that don't rely on narrow timing windows but on fundamental errors in how the kernel manages memory. Copy Fail was reportedly discovered by Theori's Xint Code AI in roughly one hour using a single prompt. These are not edge-case race conditions. They are reliable, repeatable root exploits sitting inside performance fast paths that were introduced years ago to save CPU cycles. AI is now systematically surfacing this legacy optimization debt.
 
---
 
## Section 2 — Spiderweb Architecture (The Target)
 
A 64GB multi-architecture spiderweb cluster spanning three completely distinct OS environments — all running some variant of Ubuntu, all vulnerable, all requiring a different fix.
 
| Node | Role | Architecture | OS | Kernel Type |
|---|---|---|---|---|
| HP Stream | Worker node | x86_64 | Xubuntu | Standard Canonical kernel |
| M1 MacBook | Control plane | ARM64 | Asahi Ubuntu | Custom reverse-engineered edge kernel |
| Windows Mini PC (The Tank) | Worker node | x86_64 | WSL2 (Ubuntu guest) | Microsoft-compiled, Windows-managed |
 
A single compromised pod on any node can use these primitives to escape the container, overwrite host memory, and gain root access to the entire node — and through the mesh, the entire cluster.
 
---
 
## Section 3 — Front 1: Standard Patching (HP Stream)
 
**Node:** HP Stream | x86_64 | Xubuntu
 
The only node running a standard Canonical kernel, which means the only node that gets the easy button.
 
```bash
sudo apt update && sudo apt upgrade -y && sudo reboot
```
 
The vendor pushed the compiled binary patch (Commit A664B) through the standard package manager. The upgrade swaps out the vulnerable AEAD C code in the kernel. The reboot is non-negotiable — skipping it leaves the old broken kernel actively running in RAM. The system remains 100% vulnerable until a full memory flush occurs.
 
**Verify the patched kernel is loaded:**
```bash
uname -rs
```
 
Cross-reference the output against Ubuntu's security notice for CVE-2026-31431 to confirm the version is at or above the patched release.
 
---
 
## Section 4 — Front 2: User Space Amputation (M1 MacBook — Asahi Ubuntu)
 
**Node:** M1 MacBook | ARM64 | Asahi Ubuntu | Kernel: `6.17.0-1001-asahi-arm`
 
Hard mode. The Asahi kernel is a custom reverse-engineered build that unlocks KVM/EL2 support on Apple Silicon. It does not follow the standard Ubuntu release cycle for security patches. Running `apt upgrade` here updates user-space tools and nothing else — the vulnerable kernel logic stays intact.
 
**Why standard module removal failed:**
```
rmmod: ERROR: Module aead is builtin.
```
 
The Asahi maintainers physically fused the vulnerable AEAD backend directly into the monolithic kernel binary (`CONFIG_CRYPTO_AEAD=y`). There is no module to unload. However, they left the `af_alg` socket interface — the user-space bridge the exploit requires — as a loadable module. That's the attack surface.
 
**The play:** Destroy the bridge, not the flaw. The `aead` backend is fused into the kernel and unreachable, but `af_alg` — the user-space socket the exploit needs to reach it — is a loadable module. Blacklist `af_alg` and it can never open the socket. However, `af_alg` itself had active dependents (`algif_hash` and `algif_skcipher`) holding it open, so all three had to be blacklisted in dependency order before rebuilding the initramfs to seal them into the bootloader.
 
**Copy Fail mitigation:**
```bash
# 1. Blacklist af_alg and its dependencies
echo "install af_alg /bin/true" | sudo tee /etc/modprobe.d/disable-af-alg.conf
echo "install algif_hash /bin/true" | sudo tee /etc/modprobe.d/disable-algif-hash.conf
echo "install algif_skcipher /bin/true" | sudo tee /etc/modprobe.d/disable-algif-skcipher.conf
 
# 2. Compile the locked state into the bootloader
sudo update-initramfs -u
 
# 3. Flush the active locked modules from RAM
sudo reboot
```
 
**Dirty Frag mitigation:**
```bash
echo -e "blacklist esp4\nblacklist esp6\nblacklist rxrpc" | sudo tee /etc/modprobe.d/dirtyfrag-asahi.conf
sudo update-initramfs -u
sudo reboot
```
 
Skipping `update-initramfs` is fatal to this approach. Without rebuilding the boot image, the system loads the vulnerability back into RAM before it ever reads the blacklist.
 
**Verify Copy Fail modules are gone:**
```bash
lsmod | grep -iE 'af_alg|algif'
# Expected: no output
```
 
**Verify Dirty Frag modules are gone:**
```bash
lsmod | grep -E "esp|rxrpc"
# Expected: no output
```
 
---
 
## Section 5 — Front 3: Hypervisor Kill Switch (Windows Mini PC — WSL2)
 
**Node:** Windows Mini PC (The Tank) | x86_64 | WSL2 Ubuntu guest
 
The rules flip entirely here. WSL2 is not a standard Linux installation. The Ubuntu guest is a passenger — it does not own its kernel. Microsoft compiles and manages the WSL2 kernel from the Windows host, and both the AEAD backend and the `af_alg` socket interface are baked directly into it as built-in, non-removable features (`CONFIG_CRYPTO_USER_API_AEAD=y`).
 
**Why the Asahi fix can't be applied inside WSL2:**
The `update-initramfs` approach requires writing to the initramfs. Inside WSL2, the kernel image and initramfs are mounted as read-only files by the Windows hypervisor. Anything written there is silently ignored during boot. `rmmod` fails for the same reason the Asahi attempt failed — there is no module to remove. The code is part of the kernel core.
 
**The play:** Step outside the Linux guest entirely. Edit the global WSL configuration on the Windows host and inject a kernel command-line argument that instructs the hypervisor to abort the vulnerable module's initialization before the virtual machine finishes booting.
 
**Copy Fail mitigation — run in native Windows PowerShell (not inside WSL):**
```powershell
Add-Content -Path "$env:USERPROFILE\.wslconfig" -Value "`n[wsl2]`nkernelCommandLine=initcall_blacklist=algif_aead_init"
wsl --shutdown
wsl -d Ubuntu
```
 
**Dirty Frag mitigation — append to existing `kernelCommandLine` in `.wslconfig`:**
```ini
[wsl2]
kernelCommandLine=initcall_blacklist=algif_aead_init,esp4_init,esp6_init,rxrpc_init
```
 
```powershell
wsl --shutdown
```
 
**Verify the hypervisor injected the kill command — run inside WSL Ubuntu:**
```bash
cat /proc/cmdline | grep initcall_blacklist
```
 
**Confirmed output from the actual cluster:**
```
initrd=\initrd.img WSL_ROOT_INIT=1 panic=-1 nr_cpus=4 ... initcall_blacklist=algif_aead_init
```
 
The module is technically still present in the binary. The kernel is simply ordered to abort its initialization routine at boot. The execution pathway is dead.
 
---
 
## Section 6 — Patch Status at Time of Incident
 
| CVE | Variant | HP Stream (x86_64) | M1 Asahi (ARM64) | WSL2 (Windows) |
|---|---|---|---|---|
| CVE-2026-31431 | Copy Fail | ✅ Patched via `apt upgrade` | ⚠️ Mitigated — User Space Amputation | ⚠️ Mitigated — Hypervisor Kill Switch |
| CVE-2026-43284 | Dirty Frag ESP | ✅ Patched (May 7 headers) | ⚠️ Mitigated — module blacklist | ⚠️ Mitigated — `initcall_blacklist` |
| CVE-2026-43500 | Dirty Frag RXRPC | ⚠️ Mitigated — blacklist (no upstream patch) | ⚠️ Mitigated — module blacklist | ⚠️ Mitigated — `initcall_blacklist` |
 
Only one node received a true patch. The other two received surgical mitigations that destroy the attack path without touching the vulnerable code itself. Mitigations should be treated as temporary. Monitor Asahi release feeds for kernel `6.17+` xfrm fix and Microsoft's Patch Tuesday for an official WSL2 kernel update.
 
---
 
## Lessons
 
**1. The same bug can require three completely different fixes.**
The vulnerability was identical across all three nodes. The remediation was not. The fix is always a function of the environment's kernel ownership model, not the vulnerability itself. HP Stream owned its kernel — standard patch. Asahi Ubuntu owned a fused custom kernel — destroy the attack path at the socket layer. WSL2 doesn't own its kernel at all — go above it and talk to the hypervisor directly.
 
**2. "Vendor support" is not a binary.**
Apple does not support Asahi Ubuntu. Microsoft does support WSL2. Both situations result in the same outcome: you cannot use a standard `apt upgrade` to patch the kernel. Vendor support only matters if the vendor has compiled and shipped the specific fix for your specific kernel build. Until they have, you are on your own regardless of the official support status.
 
**3. Not rebooting after patching is the same as not patching.**
The old kernel stays active in RAM until a full reboot occurs. This applies to the HP Stream's standard patch as much as it does to the initramfs rebuild on Asahi. The patch being installed and the patch being loaded are two different events.
 
**4. Prompt engineer your AI assistant — force it to the source.**
The mitigations here were developed in real time with an AI assistant. The key discipline was explicitly directing the AI to search for live patch status and architecture-specific constraints rather than generating generic commands from training data. An AI left unprompted will produce confident, plausible commands that may be completely wrong for a custom kernel environment. Specifying the hardware, the kernel build, and demanding a live search is what produced usable output.
 
---
 
## Skills
 
**Cross-architecture incident response**
Deployed the same zero-day mitigation across three fundamentally different kernel ownership models — bare-metal standard, bare-metal custom, and hypervisor-managed — without disrupting cluster state or persistent data. The core skill is recognizing that the environment determines the fix, not the CVE.
 
**Kernel module dependency mapping**
Traced the exploit path from the user-space socket interface (`af_alg`) down to the fused kernel backend (`CONFIG_CRYPTO_AEAD=y`) and identified exactly which layer was accessible for surgical removal. Understanding the dependency tree between user-space interfaces and compiled kernel modules is what made the Asahi mitigation possible.
 
**initramfs engineering**
Rebuilt the boot image on Asahi Ubuntu to bake module blacklists into the early boot sequence before any user-space tooling initializes. Reinforced why `update-initramfs` is a required step — without it, the blacklist is ignored and the vulnerability reloads into RAM on every boot.
 
**Hypervisor-level kernel configuration**
Bypassed WSL2's read-only guest environment entirely by injecting `initcall_blacklist` arguments into the Windows `.wslconfig` file. Developed a working understanding of how the Windows hypervisor controls the Linux kernel initialization sequence and where in that chain intervention is actually possible.
 
**Live vulnerability triage with AI tooling**
Used real-time web search via an AI assistant to track patch rollout status across distributions, differentiate between the ESP and RXRPC variants, and identify architecture-specific constraints (Asahi kernel fusing, WSL2 built-in module compilation) that generic documentation does not cover. The skill is in how you direct the tool, not just that you use it.
 
---
 
*Part of the spiderweb cluster postmortem series.*
 