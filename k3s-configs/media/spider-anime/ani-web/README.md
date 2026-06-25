# Ani-Web Deployment Configuration

This directory contains the production Kubernetes manifests for ani-web, a core web application component of the spider-anime media stack. This service runs on a K3s cluster and is optimized for deployment on dedicated hardware nodes (e.g., Apple Silicon/M1 platforms) using local storage volumes for data persistence.

---

## Architecture Overview

The ani-web application is deployed as a single-replica stateful workload designed to manage local anime media metadata, scraping configurations, or streaming capabilities.

```
       [ Client / Ingress ]
                │
                ▼ (NodePort: 1234)
         [ Service: NodePort ]
                │
                ▼ (TargetPort: 1234 / 3000)*
       [ Pod: ani-web Container ]
                │
                ▼ (HostPath Volume Mount)
     [ Host Directory: /your/local/path ] ──► (Stores /root/.local/share/ani-web)
```

### Key Architectural Decisions

**Node Pinning (nodeSelector):**
The deployment targets high-performance nodes labeled with `lane: powerhouse` (e.g., M1 Mac Minis or high-compute cluster workers). This ensures CPU-bound tasks like rendering or metadata scraping do not degrade the performance of lighter cluster workloads.

**Stateful HostPath Storage:**
The application mounts a local directory from the host filesystem straight into `/root/.local/share/ani-web`. This path is typical for applications saving SQLite databases, local configurations, or session states. Using a HostPath with `DirectoryOrCreate` guarantees that database files persist across pod restarts on the pinned node.

**External Port Exposure:**
The manifest configures a NodePort service, allowing the web interface to be reached directly via any cluster node's IP address on a dedicated static port.

---

## File Manifests

### ani-web-prod.yaml

This file contains the combined resources required to run the service in a production environment:

- **Deployment:** Defines the pod spec, image registry location, replicas, node affinity rules, container ports, and local volume mounts.
- **Service:** Exposes the application to the network via a static port.

---

## Configuration & Customization Guide

The provided configuration template contains placeholder variables (formatted as `your-value`, `your-namespace`, etc.). Before applying this configuration to your production cluster, update these placeholders with your environment-specific values.

### Variable Mapping Table

| Template Variable | Description |
|---|---|
| `your-namespace` | The target Kubernetes Namespace for isolation (e.g., `media` or `spider-anime`) |
| `your-app` | The unique label key-value pair used for pod routing and service selectors |
| `your-value` (Deployment Name) | The name of the Kubernetes Deployment resource |
| `your-value` (Container Name) | The name of the container running inside the pod |
| `your-value` (Volume Mount) | The identifier used to reference the host volume within the pod spec |
| `your-registry/your-image:latest` | Path to your container image inside your local Forgejo/Harbor registry |
| `/your/local/path` | The absolute path on your host node where application data will persist |

---

## Crucial Implementation Note: Port Discrepancy

> [!WARNING]
> **Port Mismatch Detected in Template**

In the template configuration, there is a mismatch between the container's configured port and the service target port:

- **Container Port:** 3000 (typically used by web engines like Node.js or Go).
- **Service Port / TargetPort / NodePort:** 1234.

If your application listens internally on port 3000, update the Service configuration to map the ports correctly:

```yaml
spec:
  type: NodePort
  selector:
    app: ani-web
  ports:
    - port: 3000       # Cluster-internal port
      targetPort: 3000 # Matches the Container Port
      nodePort: 31234  # A valid NodePort range port (30000-32767)
```

---

## Deployment Instructions

Follow these steps to deploy the ani-web service to your K3s cluster:

### 1. Label the Targeted Node

Ensure that your target worker node (e.g., M1 Mac) is properly labeled so the scheduler can place the pod correctly:

```sh
kubectl label nodes <your-node-name> lane=powerhouse
```

### 2. Customize the Manifest

Replace the placeholder values in `ani-web-prod.yaml` using a text editor, or programmatically using `sed`/`envsubst`:

```sh
# Example substituting placeholders dynamically
sed -i 's/your-namespace/media/g' ani-web-prod.yaml
sed -i 's/your-app/ani-web/g' ani-web-prod.yaml
sed -i 's/your-value/ani-web/g' ani-web-prod.yaml
```

### 3. Apply the Manifests

Deploy the service using kubectl:

```sh
kubectl apply -f ani-web-prod.yaml
```

### 4. Verify the Deployment

Confirm that the deployment is running, the host path volume is mounted, and the service is routing traffic:

```sh
# Check the pod status
kubectl get pods -n <your-namespace> -l app=<your-app> -o wide

# Check the service exposure
kubectl get svc -n <your-namespace>
```