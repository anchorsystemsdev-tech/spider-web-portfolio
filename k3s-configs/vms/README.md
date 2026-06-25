# KubeVirt Virtual Machine Configurations (k3s-configs/vms)

This directory houses the declarative manifests for deploying and managing kernel-native, virtualization-based workloads alongside containerized workloads inside our ARM64 K3s cluster (optimized for Apple Silicon/M1 Pro virtualization targets).

Using KubeVirt and the Containerized Data Importer (CDI), these configurations provision hardware-accelerated Fedora 43 Virtual Machines utilizing native hardware features.

---

## Architecture Overview

The virtual machines defined here bridge traditional virtualization and container native infrastructure. Instead of isolating VMs on separate hypervisors, KubeVirt runs VM instances inside Kubernetes Pods (virt-launcher), allowing them to leverage standard Kubernetes storage, networking, and monitoring tools.

```
                  +----------------------------------------------+
                  |                 K3s Cluster                  |
                  |                                              |
                  |   +------------------+   +----------------+  |
                  |   |  fedora-desktop  |   |   fedora-lab   |  |
                  |   |    (VM / Pod)    |   |   (VM / Pod)   |  |
                  |   +--------+---------+   +--------+-------+  |
                  |            |                      |          |
                  |      (Masquerade)            (Masquerade)    |
                  |            |                      |          |
                  |            +-----------+----------+          |
                  |                        |                     |
                  |                        v                     |
                  |               [ k8s Pod Network ]            |
                  |                        |                     |
                  |                        v                     |
                  |               [ redis-service:1234 ]         |
                  +----------------------------------------------+
```

### Key Technical Pillars

**Host-Passthrough CPU Optimization:** Both VMs leverage `model: host-passthrough`. This exposes the underlying host CPU capabilities (specifically optimized for Apple Silicon M1 Pro / ARM64 cores) directly to the guest OS.

**16K Page Size Support:** The virtual machine configurations utilize the Fedora 43 Cloud Base Generic aarch64 image. Fedora 43 natively supports 16KB page sizes, which is an absolute requirement for stable, high-performance virtualization on Apple Silicon chips.

**CDI-Driven Disk Provisioning:** Rather than manually mounting block storage, CDI automatically fetches, decompresses, and provisions the OS disks into PVs utilizing the `local-path` storage class via custom `DataVolumeTemplates`.

**Masquerade Networking:** VMs are configured with `masquerade: {}` interfaces. This assigns a private, internal NAT IP to the VM inside its host pod, allowing outbound internet access while safely isolating guest OS networking.

---

## File Registry & System Specs

| VM Name | Target Use Case | CPU Cores / Memory | Disk Spec | UI Capability | Unique Integration |
|---|---|---|---|---|---|
| Fedora Desktop | Cloud Graphical Workstation | 4 Cores / 4Gi RAM | 8Gi (local-path) | VNC / XFCE4 Graphical Environment | Python Redis Heartbeat Monitor |
| Fedora Lab | Headless Dev / CLI Sandbox | 4 Cores / 4Gi RAM | 8Gi (local-path) | Serial Console / SSH only | Minimal overhead, native ARM64 compilation lab |

---

## Detailed Manifest Deep-Dive

### 1. Fedora Desktop (fedora-desktop.yaml)

This VM acts as an automated, graphical remote workstation. Its `cloudInitNoCloud` user data executes a bootstrap script to deploy a lightweight user interface and register the VM's state within the cluster.

**Graphical Stack:** Installs Xfce Desktop and LightDM via `dnf`. It auto-configures LightDM to log the user `fedora` directly into an active graphical session on boot.

**State Reporting (Redis Heartbeat):** Installs `python3-redis` and writes a telemetry script to `/home/fedora/test-float.py`. On startup, this script runs as a persistent background daemon, sending an active heartbeat key (`heartbeat:<hostname>`) to an in-cluster Redis instance (`redis-service:6379`) every 60 seconds with a 120-second TTL.

### 2. Fedora Lab (fedora-lab.yaml)

This is a stripped-back, high-performance workspace designed for systems programming, performance benchmarking, and automation.

**Zero Overhead:** Does not compile or execute any display servers or remote management daemons.

**Development-Ready:** Provisions a fresh, clean Fedora 43 kernel environment optimized directly for heavy developer payloads on ARM64 architectures.

---

## Template Customization Guide

Both files contain placeholder tokens (such as `your-value`) and configuration values that must be customized before deployment.

### Required Placeholders Reference

Ensure you replace the following fields in the YAML files before running `kubectl apply`:

- `metadata.name`: Define unique names for the VirtualMachine objects.
- `spec.template.spec.domain.devices.inputs[0].name`: Input device identifiers.
- `spec.template.spec.domain.devices.disks[*].name`: Match these with your volume attachment names.
- `spec.template.spec.domain.devices.interfaces[0].name`: Unique interface identifier.
- `spec.template.spec.networks[0].name`: Network attachment identifier.
- `spec.template.spec.volumes[*].name`: Volume mapping links.
- `spec.template.spec.volumes[*].dataVolume.name`: Point these to your CDI data volumes.
- `spec.dataVolumeTemplates[0].metadata.name`: Unique name for the generated DataVolume.

### Security Configurations

Both files contain a placeholder for the default administrative user password:

```yaml
user: your-value
password: "REDACTED_SECRET"
```

Before deployment, replace `"REDACTED_SECRET"` with a secure, hashed, or clear-text password (or inject authorized SSH keys via standard Cloud-Init configuration).

---

## Deployment Instructions

### Prerequisites

- **KubeVirt Operator:** Ensure KubeVirt is running on your K3s cluster.
- **CDI Operator:** Ensure the Containerized Data Importer is operational to process `dataVolumeTemplates`.
- **Storage Class:** A storage class named `local-path` must be configured and active.
- **Redis Instance (For Desktop VM telemetry):** A service named `redis-service` listening on port `6379` must be accessible within the `default` namespace.

### Step-by-Step Deployment

1. Clone and navigate to configs:
```sh
   cd k3s-configs/vms
```

2. Replace Placeholders:
   Run a find-and-replace for the `your-value` identifiers, or construct a localized patch layer using Kustomize.

3. Deploy the Virtual Machines:
```sh
   kubectl apply -f fedora-lab.yaml
   kubectl apply -f fedora-desktop.yaml
```

4. Monitor Provisioning Status:
   Track the CDI importing process:
```sh
   kubectl get dv
```
   Note: On first boot, CDI will download the ~400MB QCOW2 image from the mirror repository and provision the PV. This can take several minutes.

5. Verify Running State:
```sh
   kubectl get vms
   kubectl get vmis
```

---

## Access & Operations

### 1. Interacting via Serial Console (CLI)

To access the text-based console of either virtual machine directly from your workstation, use the `virtctl` tool:

```sh
virtctl console <vm-name>
```

### 2. Accessing the Graphical UI (Fedora Desktop)

Because `autoattachGraphicsDevice: true` is enabled on the Desktop VM, KubeVirt maps the display server directly to a VNC port. You can forward this port to your local machine:

```sh
virtctl vnc <fedora-desktop-vm-name>
```

This will open your native system VNC client and connect you directly to the automated LightDM/Xfce session.

### 3. Monitoring Telemetry (Redis Diagnostics)

For the Desktop VM, you can confirm connection to your telemetry database by inspecting your Redis instance:

```sh
kubectl exec -it deployment/redis-server -- redis-cli KEYS "heartbeat:*"
```