# Ollama Service on K3s: Deployment & Architecture Guide

This repository contains the deployment configurations for running Ollama, an open-source Large Language Model (LLM) runner, within a K3s Kubernetes cluster.

This service is optimized for running resource-intensive AI models locally, leveraging containerized execution and hardware acceleration (specifically targeting platforms requiring Vulkan compatibility layers, such as Apple Silicon or specific embedded Linux/GPU nodes).

---

## 1. Architectural Overview

The Ollama service is deployed as a single-replica Deployment (`apps/v1`) to prevent write conflicts on model files and ensure sequential, high-performance GPU utilization.

```
       [ Client / Ingress ]
                │
                ▼ (Port 1234)
         [ Ollama Pod ]
  ┌─────────────┴─────────────┐
  ▼                           ▼
[ Host-Path Volume ]   [ Vulkan Driver Bootstrap ]
(Model Persistence)    (Mesa-Vulkan Driver Installation)
```

### Key Architectural Patterns

**Bootstrap Hack (Vulkan Drivers on Boot):**
The deployment uses an inline shell override (`command` and `args`) to update package lists and install `mesa-vulkan-drivers` at runtime before starting the main process (`ollama serve`). This is a specialized pattern used to dynamically provide Apple-compatible or integrated-GPU Vulkan support to the container environment without rebuilding the base Docker image.

⚠️**Privileged Security Context:**
The container runs with `privileged: true`. This elevated security context is required to allow the container direct access to the host's underlying hardware interfaces (e.g., GPU/NPU chipsets, memory-mapped devices).

**Targeted Node Scheduling (`nodeSelector`):**
To ensure the workload lands on a node with the necessary hardware accelerators (like Apple Silicon lanes or discrete GPUs), the deployment enforces a node selector (`lane: your-value`).

**Hybrid Storage Strategy:**

- **HostPath Volume:** Used to mount a local directory from the host node into the pod. This ensures that downloaded model weights (which range from 3GB to over 40GB+) persist across pod restarts and deployments.
- **EmptyDir Volume:** Utilized for high-speed ephemeral storage (e.g., temporary runtime caches).

---

## 2. File Analysis

### ollama.yaml

This manifest defines the workload configuration.

| Kubernetes Object | Purpose | Key Configurations |
|---|---|---|
| Deployment | Manages the life cycle of the Ollama pod. | `replicas: 1` / `privileged: true` / `nodeSelector: lane` |
| Volumes | Handles local storage mapping and ephemeral cache space. | `hostPath` (for model persistence) / `emptyDir` (for transient data) |

---

## 3. Configuration & Customization Guide

Before applying this configuration to your K3s cluster, you must replace the placeholder values (`your-*`) in the YAML file. Below is a guide to mapping these placeholders to production-ready values:

### Placeholder Mapping Table

| Placeholder | Description |
|---|---|
| `your-namespace` | The Kubernetes namespace to isolate your AI resources. |
| `your-value` (Metadata Name) | The identifier for your Deployment resources. |
| `your-app` (Labels) | Selector label used to tie Pods, Deployments, and Services together. |
| `lane: your-value` | Label of the specific K3s node that has GPU/NPU acceleration. |
| `your-value` (Container Name) | The name of the container running inside the pod. |
| `your-registry/your-image:latest` | The official Ollama Docker image (or your private registry mirror). |
| `your-value` (Environment Name) | Environment variables, e.g., enabling parallel request processing (value: `"1"`). |
| `/your/local/path` (Host Path) | A fast-performance local directory on the physical host machine where models will reside. |
| `/your/local/path` (Mount Path) | The directory inside the container where Ollama expects its model storage directory. |

---

## 4. Production-Ready Deployment Walkthrough

### Step 1: Label Your Accelerated Node

Identify which physical K3s node has access to the target hardware acceleration, and label it to match the `nodeSelector`:

```sh
kubectl label nodes <your-node-name> lane=apple-silicon
```

### Step 2: Create the Target Namespace

Create the namespace isolated for your artificial intelligence workloads:

```sh
kubectl create namespace ai-workloads
```

### Step 3: Apply the Configuration

Once you have updated the placeholder values in `ollama.yaml` according to the Configuration Guide, apply it via kubectl:

```sh
kubectl apply -f ollama.yaml -n ai-workloads
```

---

## 5. Verification & Troubleshooting

### Check Pod Status

Verify that the pod is running and has successfully pulled the image:

```sh
kubectl get pods -n ai-workloads -l app=ollama
```

### Monitor Bootstrapping & Installation Logs

Because the deployment runs an `apt-get` installation step prior to launching the daemon, you can watch the installation progress using the logs:

```sh
kubectl logs -f deployment/ollama-service -n ai-workloads
```

You should see output indicating `mesa-vulkan-drivers` downloading, followed by the typical Ollama initialization logs:

```
Reading package lists...
Building dependency tree...
...
Setting up mesa-vulkan-drivers:amd64 ...
time=2024-10-24T12:00:00.000Z level=INFO source=images.go:710 msg="total blobs: 0"
time=2024-10-24T12:00:00.000Z level=INFO source=images.go:717 msg="total unused blobs: 0"
time=2024-10-24T12:00:00.000Z level=INFO source=routes.go:1200 msg="Listening on [::]:1234 (version 0.3.14)"
```

### Accessing the Ollama Instance

To test the API from your local workstation, run a port-forwarding tunnel to the pod:

```sh
kubectl port-forward deployment/ollama-service 1234:1234 -n ai-workloads
```

In a separate terminal, test the local API generation:

```sh
curl http://localhost:1234/api/generate -d '{
  "model": "llama3",
  "prompt": "Why is the sky blue?"
}'
```

---

## 6. Security & Performance Considerations

⚠️**Privileged Escalation Risk:** Running with `privileged: true` gives the container full root access to the host kernel and hardware. Ensure that only trusted administrators have RBAC access to deploy or edit workloads in the `ai-workloads` namespace.

**First-Boot Latency:** Because `apt-get update && apt-get install` occurs on every container start, pod initializations may experience a 30-90 second delay. If high availability/fast recovery times are required, consider baking the `mesa-vulkan-drivers` directly into a custom golden Docker image instead of using this startup script hack.

**Host I/O Performance:** Ensure the physical directory mapped to the `hostPath` is located on an enterprise-grade SSD (e.g., NVMe). Low-speed HDDs or network-attached storage (NAS) will cause significant bottlenecks during model initialization and token-generation startup phases.