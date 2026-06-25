# Stremio Web & Streaming Engine Stack on K3s

This directory contains the Kubernetes manifests required to deploy a self-hosted, decoupled Stremio media delivery system on a K3s cluster.

The stack is split into two specialized components:

- **Stremio Web (Frontend):** The client user interface.
- **Stremio Streaming Server/Engine (Backend):** The core engine that handles P2P torrent streaming, caching, metadata resolution, and video transcoding.

---

## 1. Architectural Architecture

By separating the user interface (Frontend) from the streaming execution engine (Backend), this deployment achieves high modularity.

Both workloads are target-bound to a dedicated hardware node to utilize its hardware-accelerated video decoder pipelines (Apple VideoToolbox / NEON instruction sets) for energy-efficient, high-throughput video transcoding.

```
                    +------------------------------------------+
                    |               User Browser               |
                    +--------------------+---------------------+
                                         |
                       HTTP / HTTPS Port | (Web UI Traffic)
                                         v
                    +--------------------+---------------------+
                    |       K3s NodePort / Ingress Router      |
                    +--------------------+---------------------+
                                         |
                     +-------------------+-------------------+
                     |                                       |
                     v                                       v
        +------------+------------+             +------------+------------+
        |  Stremio Web Service    |             |  Stremio Engine Service |
        |  (NodePort: 1234/1234)  |             | (NodePort: 1234/1234)  |
        +------------+------------+             +------------+------------+
                     |                                       |
                     v                                       v
        +------------+------------+             +------------+------------+
        |   Stremio Web Pod       |             |   Stremio Engine Pod    |
        |   (Port 1234)           |             |   (Port 1234)           |
        |   Node: your-host (M1)  |             |   Node: your-host (M1)  |
        +-------------------------+             +------------+------------+
                                                             |
                                                             | Torrent / Stream
                                                             v
                                                    [ Media Torrent Swarm ]
```

---

## 2. Manifest Breakdown

### 📂 stremio-deployment.yaml (Frontend Web Client)

This manifest manages the Stremio Web UI container. It serves the web application assets to the client browser.

- **Replicas:** Set to 1 (suitable for personal/home media services).
- **Node Placement Policy:**
```yaml
  nodeSelector:
    kubernetes.io/hostname: your-host
```
  Guarantees execution on the Apple M1 Mac host , ensuring local loopback/low-latency network hops to the local streaming engine if accessed on-prem.
- **Container Port:** Exposed on port 1234.
- **Pull Policy:** Set to `Always` to guarantee that local image builds/rebuilds are automatically refreshed by the kubelet.
- **Service:** Exposes the Web UI via a NodePort mapping. (Note: Port configurations must be customized to map physical traffic from outer-cluster networks).

---

### 📂 stremio-engine.yaml (Backend Streaming Engine)

This manifest manages the Stremio Engine daemon. This container handles peer-to-peer torrent connections, media block caching, and video translation.

- **Node Placement Policy:** Like the UI, it is pinned to the M1 Mac node. Video transcoding is processor-intensive; running this on Apple Silicon ensures low CPU overhead via hardware-accelerated transcoding streams.
- **Container Port:** Port 4567.
- **Cross-Origin Environment Variable:**
  The environment variable configuration:
```yaml
  env:
  - name: your-value # Typically "STREMIO_ENGINE_CORS_ALL" or "CORS"
    value: "1"
```
  **Critical Security & Integration Note:** This variable allows the decoupled Frontend (Stremio Web) running on a different domain/port to bypass Same-Origin policies and securely issue commands, load dynamic streaming endpoints, and interface with the torrent daemon backend.

---

## 3. Preparation & Template Customization

Before deploying these manifests, you must replace all template placeholder fields (marked as `your-*`). Below is a mapping directory of required substitutions:

| Placeholder Name | Target YAML File(s) | Description |
|---|---|---|
| `your-namespace` | Both | Target Kubernetes Namespace. |
| `your-value` (Metadata Name) | Both | Resource identification label for workloads. |
| `your-app` (Labels/Selectors) | Both | Pod assignment selectors. Must match within each deployment/service pair. |
| `your-registry/your-image:latest` | Both | Path to your built or mirrored Stremio images. |
| `your-value` (Env Var Name) | stremio-engine.yaml | Key variable to activate CORS on the engine. |

### Service Port Configuration Warning ⚠️

In the raw templates, both files use static service templates:

```yaml
ports:
  - port: 1234
    targetPort: 1234
    nodePort: 1234
```

You must modify these values according to standard Kubernetes networking guidelines:

- **Kubernetes NodePort Ranges:** By default, standard Kubernetes/K3s clusters only allow NodePorts in the range 30000-32767. Port 1234 will trigger a validation error unless you configured your api-server flag `--service-node-port-range` to allow low-number port overrides.
- **Target Ports:**
  - For the Web Deployment Service, update `targetPort` to match the container port `1234`.
  - For the Engine Deployment Service, update `targetPort` to match the container port `4567`.

---

## 4. Production-Ready Configuration Example

Below is a production-hardened implementation mapping out the corrected networking targets, structural labels, and actual environment variables.

### Production stremio-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stremio-web
  namespace: your-value
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stremio-web
  template:
    metadata:
      labels:
        app: stremio-web
    spec:
      nodeSelector:
        kubernetes.io/hostname: your-value
      containers:
      - name: stremio-web-app
        image: stremio/web:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 1234
          name: http-web
---
apiVersion: v1
kind: Service
metadata:
  name: stremio-web-svc
  namespace: your-value
spec:
  type: NodePort
  selector:
    app: stremio-web
  ports:
    - name: http
      port: 1234
      targetPort: 1234
      nodePort: 1234  
```

### Production stremio-engine.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stremio-engine
  namespace: your-value
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stremio-engine
  template:
    metadata:
      labels:
        app: stremio-engine
    spec:
      nodeSelector:
        kubernetes.io/hostname: your-value
      containers:
      - name: stremio-streaming-server
        image: stremio/server:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 4567
          name: streaming-api
        env:
        - name: STREMIO_ENGINE_CORS_ALL
          value: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: stremio-engine-svc
  namespace: your-value
spec:
  type: NodePort
  selector:
    app: stremio-engine
  ports:
    - name: streaming
      port: 4567
      targetPort: 4567
      nodePort: 4567  
```

---

## 5. Deployment Guide

Execute the following commands to apply the customized configurations directly to your K3s cluster:

```sh
# 1. Create the target namespace (if it does not exist)
kubectl create namespace media --dry-run=client -o yaml | kubectl apply -f -

# 2. Apply the Streaming Engine manifest
kubectl apply -f k3s-configs/media/stremio-web/stremio-engine.yaml

# 3. Apply the Web Client UI manifest
kubectl apply -f k3s-configs/media/stremio-web/stremio-deployment.yaml
```

---

## 6. Verification and Troubleshooting

### Pod Lifecycle Check

Verify that the pods are deployed successfully and targeting node gbuntu:

```sh
kubectl get pods -n media -o wide
```

Expected status should be `Running` with IP addresses assigned.

### Logs & Engine Diagnostics

If Stremio Web displays connection errors pointing to the Streaming Server, check the engine's standard output to ensure CORS policies are active:

```sh
kubectl logs deployment/stremio-engine -n your-value -c your-value --tail=100
```

Look for initialization strings indicating that the server is listening on `0.0.0.0:4567` with CORS wildcards active.

### Validating Transcoding Node Mapping

Ensure that the engine and web server are physically bound to the your-node using the following query:

```sh
kubectl get pods -n your-value -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```