# System Resource Governance & Storage Infrastructure

This directory contains the core system-level configurations for managing resource allocation, host protection, and local persistent storage within the K3s edge cluster.

These configurations collectively establish cluster-wide guardrails to prevent resource exhaustion on host machines, provide default compute baselines for workloads, and provision high-performance local storage pinned to specific edge nodes.

---

## Architecture Overview

The configurations in this directory form a three-tier system governance model:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Host Machine (e.g., Mac)                        │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                     K3s Cluster / Namespace                    │   │
│   │                                                                │   │
│   │   ┌────────────────────────────────────────────────────────┐   │   │
│   │   │                     ResourceQuota                      │   │   │
│   │   │                 (mac-safety-net.yaml)                  │   │   │
│   │   │  Limits total cluster/namespace memory (10Gi) &        │   │   │
│   │   │  storage (10Gi) to protect host OS from starvation.     │   │   │
│   │   └───────────────────────────┬────────────────────────────┘   │   │
│   │                               │ enforces                       │   │
│   │                               ▼                                │   │
│   │   ┌────────────────────────────────────────────────────────┐   │   │
│   │   │                      LimitRange                        │   │   │
│   │   │                (resource-limits.yaml)                  │   │   │
│   │   │  Applies default CPU/Memory requests & limits to       │   │   │
│   │   │  individual pods/containers automatically.             │   │   │
│   │   └────────────────────────────────────────────────────────┘   │   │
│   │                                                                │   │
│   │   ┌────────────────────────────────────────────────────────┐   │   │
│   │   │                   PersistentVolume                     │   │   │
│   │   │              (edge-vault-storage.yaml)                 │   │   │
│   │   │  Binds 108Gi of local host directory storage on node   │   │   │
│   │   │  "hp-stream" for stateful applications.                 │   │   │
│   │   └────────────────────────────────────────────────────────┘   │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

- **Host-Level Protection (Resource Quotas):** Protects the underlying host system by capping aggregate RAM and disk requests.
- **Pod-Level Governance (Limit Ranges):** Automatically injects safe CPU and memory constraints into any container deployed without explicit resource declarations.
- **Hardware-Pinned Storage (Local PV/PVC):** Provisions a static local-path volume directly mapped to a physical disk directory on a dedicated edge node (hp-stream).

---

## File Breakdown & Deep Dive

### 1. Host Protection: mac-safety-net.yaml

This manifest defines a ResourceQuota that caps the total compute and storage footprints. It is especially vital when running K3s on developer hardware to prevent Kubernetes from consuming all host resources and causing system instability.

**Key Controls:**

- `requests.memory: 10Gi`: The cumulative RAM requested by all running pods/VMs in the target namespace cannot exceed 10 GiB.
- `requests.storage: 10Gi`: Prevents stateful applications from claiming more than 10 GiB of storage aggregate, leaving a safety buffer of free space on the host machine disk.

### 2. Pod Standardization: resource-limits.yaml

This manifest uses a LimitRange to enforce resource boundaries at the container level. It acts as an automatic "sizing profile" for any microservices deployed without specified resources.

**Defaults Enforced:**

- Memory Limit (`256Mi`) / Request (`128Mi`): Prevents "noisy neighbors" from ballooning in memory usage, while ensuring they start with a lightweight footprint.
- CPU Limit (`500m` / 0.5 cores) / Request (`100m` / 0.1 cores): Standardizes CPU shares to prevent single-threaded processes from hogging host CPU cycles.

### 3. Edge Storage: edge-vault-storage.yaml

This file handles high-performance, node-specific storage provisioning. It targets a physical machine in the cluster network (hp-stream) to host a local 108 GiB vault storage.

> ⚠️ **Implementation Note:** The second object in this file has a redacted kind: `"REDACTED_BASE64_DATA"`. Based on its schema, it represents a PersistentVolumeClaim (PVC) designed to bind directly to the PersistentVolume (PV).

- **Node Affinity:** Pinned specifically to the node `hp-stream` using the node label `kubernetes.io/hostname: hp-stream`.
- **Reclaim Policy:** Set to `Retain`. If the claim is deleted, the physical data on the host disk is not destroyed automatically, preserving edge data integrity.

---

## Configuration & Setup Guide

Before applying these files to your cluster, update the placeholder values (`your-value`, `your-namespace`, etc.) to match your target environment.

### Target Environment Adjustments

| File | Parameter | Description |
|---|---|---|
| `edge-vault-storage.yaml` | PV Metadata Name | Must match the `volumeName` field in the PVC. |
| `edge-vault-storage.yaml` | PV Local Path | Absolute path on the node where vault data will be stored. |
| `edge-vault-storage.yaml` | Node Hostname | Change to actual node hostname if not using `hp-stream`. |
| `edge-vault-storage.yaml` | PVC Kind | Replace `"REDACTED_BASE64_DATA"` with `PersistentVolumeClaim`. |
| `edge-vault-storage.yaml` | PVC Metadata Name | The name of the PVC resource. |
| `edge-vault-storage.yaml` | PVC Namespace | The destination namespace for the PVC. |
| `mac-safety-net.yaml` | Quota Name | The metadata name for the ResourceQuota resource. |
| `resource-limits.yaml` | LimitRange Name | The metadata name for the LimitRange resource. |

---

## Deployment & Verification

Apply the configurations in order of dependency: first the boundary limits, then the physical storage.

### 1. Apply Resource Governance & Quotas

Apply these to your target namespace (e.g., `default` or your custom namespace):

```sh
# Apply host safety net and default resource limits
kubectl apply -f resource-limits.yaml -n <your-namespace>
kubectl apply -f mac-safety-net.yaml -n <your-namespace>
```

### 2. Apply Local Edge Storage

```sh
# Ensure the local directory exists on node "hp-stream" before applying
ssh user@hp-stream "mkdir -p /your/local/path"

# Apply PV and PVC
kubectl apply -f edge-vault-storage.yaml
```

### 3. Verify System Health

Verify that the resource quotas and limit ranges are active:

```sh
kubectl describe quota -n <your-namespace>
kubectl describe limitrange -n <your-namespace>
```

Verify that the local storage PV has successfully bound to the PVC:

```sh
kubectl get pv,pvc -n <your-namespace>
```

Expected Output Status: `Bound`

---

## Operational Best Practices

**Eviction Avoidance:** The `mac-safety-net` quota is designed to trigger validation errors at deploy-time rather than allowing pods to run and cause host-level disk/memory pressure. If a deployment fails with `Forbidden` errors, check your aggregate resource requests using `kubectl describe ns <your-namespace>`.

**Local Disk Backups:** Because `edge-vault-storage` utilizes local volume provisioning, standard Kubernetes volume snapshots are not natively supported. Ensure that the physical directory on `hp-stream` is backed up via local cron jobs or an external daemon (e.g., `rsync`, `restic`).

**Node Maintenance:** If node `hp-stream` needs to go down for maintenance, workloads dependent on `edge-vault-storage` will become unschedulable due to the strict `nodeAffinity` rules. Gracefully drain other workloads but be prepared for stateful workload downtime during host maintenance.