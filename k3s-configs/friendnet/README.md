# FriendNet Cluster Deployment & Build Pipeline

This directory contains the declarative Kubernetes (K3s) configuration files required to build, cross-compile, and run the FriendNet service on a hybrid-architecture K3s cluster.

The architecture leverages a high-performance ARM64 node (Apple Silicon M1 Mac) for compilation workloads (both native Asahi-linux mac and cross-compiled Linux x86_64 targets) and deploys the running FriendNet daemon on a dedicated x86_64 edge node (your-hostname).

---

## 1. System Architecture & Topology

The deployment architecture is split into two phases: Build/Compilation Jobs and the Daemon Runtime Deployment.

```
                                      +-----------------------------------------+
                                      |               K3s Cluster               |
                                      +-----------------------------------------+
                                                           |
                          +--------------------------------+--------------------------------+
                          |                                                                 |
            [ Node Selector: arm64 ]                                              [ Node Selector: your-hostname ]
               (Apple Silicon Mac)                                                    (x86_64 Mini PC / Laptop)
                          |                                                                 |
       +------------------+------------------+                                              |
       |                                     |                                              |
+------v------------------+       +----------v--------------+                    +----------v--------------+
| build-friendnet-mac     |       | friendnet-edge-builder  |                    | friendnet-server        |
| (Job: Go compiler)      |       | (Job: Go cross-compiler)|                    | (Deployment: Server)    |
+------+------------------+       +----------+--------------+                    +----------+--------------+
       |                                     |                                              |
       | (Saves Binary)                      | (Saves Binary)                               | (Persistence)
       v                                     v                                              v
+-----------------------------------------------------------+                    +-------------------------+
| HostPath Volume:                                          |                    | HostPath Volume:        |
| `/your/local/path/friendnet`                 |                    | Custom Host Directory   |
+-----------------------------------------------------------+                    +-------------------------+
```

### Key Architectural Patterns

**Targeted Scheduling (nodeSelector):** Build workloads are computationally expensive and platform-specific. The build jobs are pinned to an ARM64 system (kubernetes.io/arch: arm64) representing your Apple Silicon node. The runtime server is pinned via hostname (kubernetes.io/hostname: your-hostname) to the x86 edge node.

**Bare-Metal Port Binding (hostNetwork: true):** The server pod bypasses the virtual container network interface (CNI) and binds directly to the host network interface of the x86 node. This replicates standard network_mode: host behavior from Docker Compose, allowing low-overhead, direct routing to the host's ports.

**HostPath Compilation Outbox:** Compiling binaries inside isolated ephemeral containers is clean, but extracting them can be difficult. These configurations use hostPath volumes (/your/local/path/friendnet) to drop compiled binaries directly onto the host node's filesystem for simple extraction and provisioning.

---

## 2. File Analysis

### 2.1. build-friendnet-mac.yaml (Asahi-linux mac Compilation Job)

- **Resource Type:** Kubernetes Job
- **Target Node:** ARM64 Node (kubernetes.io/arch: arm64)
- **Base Image:** Custom Golang/Node compiler image (templated)

**Execution Flow:**

1. Installs compilation utilities (git, make, nodejs, npm) on top of an Alpine-based container.
2. Clones the remote repository: https://github.com/termermc/FriendNet.git.
3. Builds frontend web assets utilizing the Node toolchain (make webui).
4. Targets Apple Silicon execution by compiling the Go binary with:
```
   GOOS=darwin GOARCH=arm64 go build -o friendnet-client ...
```
5. Copies the finished binary to /output/ (bound to the host folder /your/local/path/friendnet).

---

### 2.2. friendnet-edge-builder.yaml (Linux x86 Cross-Compilation Job)

- **Resource Type:** Kubernetes Job
- **Target Node:** ARM64 Node (kubernetes.io/arch: arm64)
- **Base Image:** Custom Golang/Node compiler image (templated)

**Execution Flow:**

1. Performs the same environment setup and Git cloning sequence as the Mac builder.
2. Cross-compiles the code targeting standard x86 Linux servers/mini-PCs:
```
   env GOOS=linux GOARCH=amd64 go build -o friendnet-client-amd64 ...
```
3. Outputs the compiled friendnet-client-amd64 binary to /output/ (bound to /your/local/path/friendnet).

---

### 2.3. friendnet-server.yaml (Server Runtime Deployment)

- **Resource Type:** Deployment (1 Replica)
- **Target Node:** x86 Node labeled kubernetes.io/hostname: your-hostname
- **Networking:** Host network namespace mode.
- **Persistence:** Mounts two distinct hostPath directories from the hosting x86 machine into the container to preserve runtime database states, logs, or static web configurations.

---

## 3. Configuration & Template Customization

These manifests use placeholder configurations containing your-value, your-namespace, and related templates. You must customize these values before deploying them to your cluster.

### Placeholder Reference Table

| Placeholder | Context | Recommended Value |
|---|---|---|
| your-namespace | Metadata namespace for resource isolation | e.g., friendnet or default |
| your-value (Metadata names) | Name of specific resource instances | e.g., friendnet-mac-builder, friendnet-edge-builder, friendnet-server |
| your-app | Selector labels for Deployment and Pod grouping | e.g., friendnet-server |
| your-registry/your-image:latest | Container image containing Go, GCC, Git, and Node toolchains | e.g., golang:1.21-alpine (or a prebuilt internal registry image) |
| /your/local/path | Host and mount paths for configuration files and databases | e.g., /var/lib/friendnet/data (Host) and /data (Container) |

---

## 4. Step-by-Step Deployment Instructions

Follow these steps to customize, trigger, and verify the FriendNet ecosystem within your cluster.

### Step 1: Prepare the Namespace

Create the target namespace in your cluster if it does not already exist:

```sh
kubectl create namespace friendnet
```

### Step 2: Replace Template Placeholders

You can use sed (or your preferred editor) to replace the placeholders across the files. For example, to target the namespace friendnet and a public Alpine-based Go image golang:1.21-alpine:

```sh
# Set the namespace
sed -i 's/namespace: your-namespace/namespace: friendnet/g' *.yaml

# Define standard toolchain image for build jobs
sed -i 's|image: your-registry/your-image:latest|image: golang:1.21-alpine|g' build-friendnet-mac.yaml friendnet-edge-builder.yaml

# Set the host mount paths in the server deployment file
# (Replace with your actual local disk mount locations)
sed -i 's|path: /your/local/path|path: /var/lib/friendnet/data|g' friendnet-server.yaml
```

### Step 3: Run the Build Pipeline

Apply the Kubernetes Jobs to start compiling the binaries.

To compile the Asahi-linux-mac ARM64 binary:

```sh
kubectl apply -f build-friendnet-mac.yaml
```

To compile the Linux x86_64 Edge binary:

```sh
kubectl apply -f friendnet-edge-builder.yaml
```

### Step 4: Monitor and Extract Binaries

Monitor the execution status of your compilation jobs:

```sh
kubectl get jobs -n friendnet -w
```

Once the jobs complete successfully, check the host path on the ARM64 host machine:

```sh
ls -la /your/local/path/friendnet
```

You should see:

- friendnet-client (Asahi-linux-mac arm64 binary)
- friendnet-client-amd64 (Linux x86_64 binary)

### Step 5: Deploy the Server Daemon

Now that your binaries are compiled, apply the daemon configurations to run the backend server instance on the x86 node:

```sh
kubectl apply -f friendnet-server.yaml
```

Verify that the server pod is running in host-network mode:

```sh
kubectl get pods -n friendnet -o wide
```

---

## 5. Security and Operational Guidelines

- **Using `hostNetwork: true`:** Pods running with host networking bypass standard network isolation. This means port collisions can occur if another application on the `your-hostname` node is utilizing the same ports as FriendNet. Ensure those ports are open and not blocked by local host firewalls (such as `ufw` or `iptables`).
- **Root Execution in Compilation Jobs:** These jobs execute compiling scripts that install alpine packages (`apk add`). Ensure the base image is running as a user with root access (or utilizing rootless containers if your cluster configurations dictate strict security policies).
- **Using `hostPath` vs. Persistent Volume Claims (PVCs):**
   - The build jobs write directly to a host directory (`/your/local/path/friendnet`). Ensure this path exists on your ARM64 build node with write permissions for the user running the K3s agent process.
   - For production-grade resilience on the `friendnet-server` deployment, consider migrating your persistence strategy from `hostPath` to a K3s-compatible storage solution (such as the default Local Path Provisioner or Longhorn) to decouple the service from absolute path states on the host disk.