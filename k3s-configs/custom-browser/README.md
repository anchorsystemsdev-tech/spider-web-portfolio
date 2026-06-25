# Custom Browser Deployment on K3s

This directory contains the declarative Kubernetes manifests required to deploy a containerized, hardware-accelerated, and stateful web browser (Firefox) with interactive desktop access (VNC) and audio redirection inside a K3s cluster.

By leveraging native node storage, Linux device passthrough, and socket-based audio forwarding, this deployment delivers a bare-metal interactive browser experience running entirely inside a lightweight container orchestration engine.

## 1. Architectural Overview

The deployment pattern leverages a single-replica stateful application topology running on a targeted bare-metal/local node (`gbuntu`). Below is the high-level system architecture:

```text
                  +-------------------------------------------------+
                  |                   Client LAN                    |
                  +-------------------------------------------------+
                                           |
                                           | (VNC/TCP Port 1234)
                                           v
                  +-------------------------------------------------+
                  |               K3s Node (gbuntu)                 |
                  |                                                 |
                  |  +------------------+     +------------------+  |
                  |  |  NodePort (1234) | --> | Service (7890)   |  |
                  |  +------------------+     +------------------+  |
                  |                                    |            |
                  |                                    v            |
                  |                           +------------------+  |
                  |                           |   Browser Pod    |  |
                  |                           | (TargetPort 1234)|  |
                  |                           +------------------+  |
                  |                                    |            |
                  |       +----------------------------+            |
                  |       | (Host Mounts)                           |
                  |       v                                         |
                  |  +---------+  +-------------+  +-------------+  |
                  |  |  audio  |  | /dev/shm    |  | /dev/udev   |  |
                  |  | socket  |  | (Memory     |  | (Hardware   |  |
                  |  | (Pulse) |  |  emptyDir)  |  | acceleration|  |
                  |  +---------+  +-------------+  +-------------+  |
                  |       |                                         |
                  |       +-----> Persistent Storage (5Gi PV)       |
                  |               /your/local/path                  |
                  +-------------------------------------------------+
```

### Core Features

* **Persistent Browser Profile:** Keeps cookies, history, and extension state intact between pod restarts using a local directory-bound PersistentVolume.
* **Low-Latency Audio:** Forwards host-level audio streams through a shared UNIX socket bound directly to PulseAudio.
* **Hardware Acceleration & Input Handling:** Operates in privileged mode with explicit `udev` and hardware volume bindings to allow GPU acceleration and peripheral device access.
* **Shared Memory Optimization:** Dynamically mounts an in-memory `emptyDir` volume to bypass the standard container runtime shared memory (`/dev/shm`) limit of 64MB, preventing rendering engine crashes.

---

## 2. Manifest Breakdown

### `pv.yaml` & `pvc.yaml` (Stateful Storage Layer)
To maintain browser state (user profiles, sessions, and bookmarks), persistent storage is established via a node-affinity bound Local Persistent Volume.

* **`pv.yaml`:** Standardizes a 5Gi local storage block pointing to `/your/local/path`. The PV is pinned explicitly to node `your-hostname` using `nodeAffinity` rules to ensure that the pod is always scheduled on the host hosting the physical disk data.
* **`pvc.yaml`:** Declares a request for 5Gi of storage matching the `local-storage` StorageClass.

> **Note:** The manifest contains a placeholder key `kind: "REDACTED_BASE64_DATA"`. During deployment, this must be normalized to `kind: PersistentVolumeClaim`.

### `firefox.yaml` (Deployment Specification)
The central deployment template. It schedules a single browser instance on node `gbuntu`.

* **Security Context:** Run as `privileged: true` to access host-level graphics APIs (DRI, rendering nodes) and hardware events.
* **Environment Variables:**
  * Displays resolution config (1728x1080).
  * Maps Unix audio sockets (`unix:/tmp/pulse/socket`).
* **Volume Mount Strategy:**
  * **User Profile Persistence:** Mounts the Persistent Volume Claim directly onto the browser's profile directory.
  * **Device Nodes & Hardware:** Mounts host device directory mappings (such as `udev`) to ensure correct hardware recognition and input processing.
  * **Shared Memory (`/dev/shm`):** Maps a 2Gi `emptyDir` using RAM (`medium: Memory`) to host-level memory buffers to maintain browser process stability.
  * **Audio Engine:** Maps a `hostPath` Unix domain socket representing `/tmp/pulse/socket` to stream container audio output back to the host system.

### `browser-service.yaml` (Networking Interface)
Exposes the browser container's internal graphical interface over a reliable, persistent network interface.

* **Type:** `NodePort`
* **Target Mapping:** Connects target port 1234 (the native container VNC server or socket) to an exposed NodePort 1234 on the physical LAN, bypassing the need for complex external Ingress controllers when connecting with client-side VNC viewers.

---

## 3. Configuration & Template Replacement Guide

Before applying these manifests to your K3s cluster, replace the generic placeholders (`your-value`, `your-namespace`, etc.) with your environment's parameters.

| Target File | Placeholder | Suggested Production Value | Purpose |
| :--- | :--- | :--- | :--- |
| **All Files** | `your-namespace` | `media-services` or `default` | Target Kubernetes Namespace |
| **All Files** | `your-value` (Metadata Name) | `custom-browser` | Unique identifier for the k8s resource |
| **All Files** | `your-app` (Labels/Selector) | `firefox-browser` | Relates Services to Deployment pods |
| **`firefox.yaml`** | `your-registry/your-image:latest` | `lscr.io/linuxserver/firefox:latest` | Base image (VNC graphical wrapper) |
| **`firefox.yaml`** | `kubernetes.io/hostname: your-value` | `your-hostname` | Node running physical storage and display |
| **`firefox.yaml`** | Volume Mount Paths (`/your/local/path`) | Match host locations (e.g., `/dev/shm`, `/run/udev`) | Map container ports to physical host resources |
| **`pvc.yaml`** | `REDACTED_BASE64_DATA` | `PersistentVolumeClaim` | Restore native resource classification |

---

## 4. Production Installation Flow

Follow these sequential steps to apply the browser configuration to your K3s cluster.

### Step 1: Prepare the Host Directory
Log into the host machine where the application will run and provision the storage path:

```bash
sudo mkdir -p /your/local/path
sudo chown -R 1000:1000 /your/local/path # Match default container user permissions
```

### Step 2: Customize Manifests
Ensure all YAML parameters are filled out. You can use this example `sed` inline edit pattern to quickly normalize names:

```bash
# Normalize the Redacted PVC Kind
sed -i 's/"REDACTED_BASE64_DATA"/PersistentVolumeClaim/g' pvc.yaml
```

### Step 3: Apply Configurations
Execute the deployment manifests in logical order using `kubectl`:

```bash
# Apply Storage Resources First
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml

# Apply Workload Configuration
kubectl apply -f firefox.yaml

# Open External Network Ingress Port
kubectl apply -f browser-service.yaml
```

### Step 4: Verify Deployment Status
Ensure your pods, PVC bindings, and services are fully operational:

```bash
kubectl get pods -n your-namespace -l app=your-app -o wide
kubectl get pvc -n your-namespace
kubectl get service -n your-namespace
```

---

## 5. Technical Considerations & Troubleshooting

### Shared Memory Buffer Management
Modern browsers spawn isolated, multi-threaded rendering processes for individual tabs. By default, Docker/Kubernetes containers restrict the available shared memory pool `/dev/shm` to 64MB. Under average browsing loads, this will trigger `SIGBUS` crashes.

* **Solution applied:** The configured Memory-backed `emptyDir` acts as a RAM-disk, setting `/dev/shm` dynamically up to 2Gi, resolving memory-exhaustion crashes.

### Troubleshooting Audio Redirection
If audio output is not processing to host speakers:

1. Validate that the host system has `pulseaudio` or `pipewire-pulse` active.
2. Verify host socket configuration path permissions:
   ```bash
   ls -la /tmp/pulse/socket
   ```
3. Ensure the browser volume configuration references the correct host path socket inside `firefox.yaml`.

### Graphic/Display Output Issues
If the pod fails to start with errors regarding `/dev/dri` or graphics drivers, ensure your target node's graphics cards drivers are updated and the target container user has read-write permissions over the `/dev/dri` directory. You can add the specific GPU mount pathways in the `firefox.yaml` volumes block if hardware-accelerated decoding is required.