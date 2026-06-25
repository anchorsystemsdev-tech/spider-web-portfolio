# SearXNG Deployment on K3s Cluster

This directory contains the Kubernetes manifest files required to deploy SearXNG—a privacy-respecting, hackable metasearch engine—onto my K3s cluster.

The configuration is tailored for a multi-architecture or resource-constrained cluster, using node targeting to run on a specific x86 edge node (hp-stream).

---

## Architecture Overview

```
                     +---------------------------------------+
                     |             K3s Cluster               |
                     |                                       |
                     |   +-------------------------------+   |
                     |   | Node: hp-stream (x86 Edge)   |   |
                     |   |                               |   |
                     |   |  +-------------------------+  |   |
                     |   |  | Pod: SearXNG            |  |   |
                     |   |  | - Port: 1234            |  |   |
                     |   |  +------------^------------+  |   |
                     |   +---------------+---------------+   |
                     |                   |                   |
                     |   +---------------+---------------+   |
                     |   | Service: SearXNG (ClusterIP)  |   |
                     |   | - Port: 1234                  |   |
                     |   | - TargetPort: 1234        |   |
                     |   +-------------------------------+   |
                     +---------------------------------------+
```

The architecture consists of:

- **Deployment (searxng.yaml):** Spawns a single replica (`replicas: 1`) of the SearXNG container. It enforces node placement onto an x86 hardware node (hp-stream) using a `nodeSelector`.
- **Service (searxng.yaml):** Exposes the SearXNG pods internally within the cluster.

---

## File Breakdown

### searxng.yaml

This file is a multi-document YAML manifest containing both the Deployment and the internal Service abstractions.

#### 1. Deployment Specification

- **Replicas:** Configured to run exactly 1 instance to minimize resource consumption on the edge node.
- **Node Selector:**
```yaml
  nodeSelector:
    kubernetes.io/hostname: hp-stream
```
  This is a critical constraint. It pins the pod to the node named `hp-stream`. This is typical in hybrid-architecture clusters (e.g., mixed ARM64/ and x86_64 nodes) to ensure the container image (which may only support amd64) is scheduled on the correct hardware.
- **Resource Limits & Requests:**
  - CPU: Requesting `100m` (0.1 CPU cores) with a hard cap at `500m` (0.5 CPU cores).
  - Memory: Requesting `128Mi` with a limit of `512Mi`. These values prevent SearXNG from executing out-of-memory (OOM) chain-reactions on the hosting edge node during high search traffic.
- **Container Port:** Exposes port `8080`.

#### 2. Service Specification

- **Type:** Defaults to ClusterIP (internal cluster IP).
- **Port Mapping:**
  - External port: `1234`
  - Target port: `1234` (Note: See Warning below regarding port mismatch).

---

## Critical Configuration Warnings & Actions Required

Before applying this configuration to your production cluster, you must address the following placeholders and structural inconsistencies:

### 1. Port Mismatch Resolution (Action Required)

In the provided `searxng.yaml`, the Deployment exposes container port `1234`, but the Service targets port `4567`:

```yaml
# Deployment
ports:
  - containerPort: your-value

# Service
ports:
  - protocol: TCP
    port: your-value
    targetPort: your-value # <-- This mismatch will cause connection timeouts
```

Fix: Update `targetPort` in the Service configuration to match your container port:

```yaml
    - protocol: TCP
      port: 1234
      targetPort: 1234  # Corrected to match containerPort
```

### 2. Variable Substitution Checklist

Search and replace the placeholder values in `searxng.yaml` with your actual cluster values:

| Placeholder | Recommended Value | Description |
|---|---|---|
| `your-namespace` | `searxng` (or your chosen namespace) | The target namespace for logical isolation. |
| `your-value` (Deployment/Service Name) | `searxng` | Standardized resource identifier. |
| `your-app` | `searxng` | The selector label used to bind the Service to the Deployment. |
| `your-registry/your-image:latest` | `searxng/searxng:latest` | The container image source. (If using the official image, use `searxng/searxng:latest` or a pinned version). |

---

## Deployment Guide

### Step 1: Prepare the Namespace

Ensure the namespace specified in your metadata exists in the cluster:

```sh
kubectl create namespace your-namespace --dry-run=client -o yaml | kubectl apply -f -
```

### Step 2: Apply the Configurations

Run the following command from the root directory of your configuration files:

```sh
kubectl apply -f k3s-configs/searxng/searxng.yaml
```

### Step 3: Verify the Deployment

Verify that the pod is successfully scheduled, specifically ensuring it landed on the `your-hostname` node:

```sh
kubectl get pods -n your-namespace -o wide
```

Verify that the NODE column points to `your-hostname`.

Check the service endpoint mapping:

```sh
kubectl get endpoints your-value -n your-namespace
```

---

## Accessing the Service

Since the Service is defined as a ClusterIP, it is only accessible inside the K3s cluster. To expose SearXNG to external users, you should deploy an Ingress resource (using Traefik, which is packaged by default with K3s) or change the Service type to `LoadBalancer`.

### Example Ingress Route (Optional)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: searxng-ingress
  namespace: your-namespace
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  rules:
  - host: search.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: your-value
            port:
              number: 1234
```