# Redis Grid & High-Availability Sentinel Architecture

This directory (`k3s-configs/redis`) contains the Kubernetes manifests required to deploy a multi-tiered, heterogeneous, and highly available Redis cache and database layer onto a K3s cluster.

The architecture is designed to handle uneven bare-metal environments (ranging from high-end "powerhouse" nodes to resource-constrained edge machines) using node-affinity-driven scheduling, while using Redis Sentinel for intelligent monitoring and failover coordination.

---

## Architecture Overview

```
                  ┌──────────────────────────────────────────┐
                  │          Kubernetes Client /             │
                  │          In-Cluster Consumer             │
                  └────────────────────┬─────────────────────┘
                                       │
                                       ▼ (Port 1234)
                        ┌──────────────────────────────┐
                        │   redis-service (ClusterIP)   │
                        └──────────────┬───────────────┘
                                       │
                         Translates to │ TargetPort 1234
                                       ▼
                        ┌──────────────────────────────┐
                        │    spider-web-grid Service   │
                        │          (Headless)          │
                        └──────────────┬───────────────┘
                                       │
       ┌───────────────────────────────┼──────────────────────────────┐
       ▼ (Lane 1: Mac)                 ▼ (Lane 2: Mini PC)            ▼ (Lane 3: HP Stream)
┌──────────────────────────────┐┌──────────────────────────────┐┌──────────────────────────────┐
│  StatefulSet: Lane-1         ││  StatefulSet: Lane-2         ││  StatefulSet: Lane-3         │
│  - Limit: 2Gi                ││  - Limit: 1Gi                ││  - Limit: 512Mi              │
│  - Node: lane: mac           ││  - Node: lane: mini-pc       ││  - Node: lane: hp-stream     │
└──────────────────────────────┘└──────────────────────────────┘└──────────────────────────────┘
                                               ▲
                                               │ Monitors via DNS
                                ┌──────────────┴───────────────┐
                                │       Redis Sentinel         │
                                │   (sentinel-config.yaml)     │
                                └──────────────────────────────┘
```

### Core Architecture Highlights

**Heterogeneous "Lane" Topology (The Spider-Web Grid):**
Instead of a homogeneous cluster where all replicas require identical specs, the cluster is divided into three specialized lanes mapped to specific hardware classes (a high-end Mac, a mid-tier Mini PC, and a lightweight low-end HP Stream laptop). This allows K3s to run on mismatched physical hardware without running out of memory on smaller nodes.

**Dedicated Powerhouse Nodes:**
An optional standalone "powerhouse" deployment is configured to target hardware optimized for heavy processing, complete with defined CPU/Memory requests and limits.

**High-Availability Sentinel Oversight:**
A ConfigMap controls the configuration of a Redis Sentinel container, enabling automated failover monitoring of the primary Redis node across the spider-web service mesh.

---

## File Directory & Component Breakdown

### 1. spider-web-grid.yaml

This is the core topology blueprint of the multi-tiered cache network. It provisions a headless discovery service and three asymmetric StatefulSets acting as target lanes.

- **Headless Service (spider-web-grid):** Declares `clusterIP: None` to allow direct DNS resolution to the individual Pod IPs of the backend Lanes rather than load-balancing across them.
- **Lane 1: The Mac (High-Performance):**
  - Memory Limit: `2Gi`
  - Scheduling: Targeted to nodes labeled with `lane: mac-node-value` (templated placeholder).
- **Lane 2: The Mini PC (Standard Performance):**
  - Memory Limit: `1Gi`
  - Scheduling: Targeted to nodes labeled with `lane: minipc-node-value`.
- **Lane 3: The HP Stream (Low-End Edge):**
  - Memory Limit: `512Mi`
  - Scheduling: Targeted to nodes labeled with `lane: hpstream-node-value`.

### 2. sentinel-config.yaml

Defines the ConfigMap for Redis Sentinel, which provides high-availability monitoring, notification, and automatic failover.

- **Monitoring Target:** Monitors the master instance reachable at:
  `redis-powerhouse-0.your-value.your-value.svc.cluster.local` (Port 1234)
- **Quorum:** Configured to `2` (requires at least two Sentinels to agree that the master is unreachable before initiating a failover).
- **Timeouts:**
  - `down-after-milliseconds`: 5000 (5 seconds before considering the master down).
  - `failover-timeout`: 10000 (10 seconds for the failover operation to complete).
- **Hostname Support:** `resolve-hostnames yes` and `announce-hostnames yes` are enabled to ensure proper operation within the Kubernetes CoreDNS environment.

### 3. redis-powerhouse.yaml

Deploys a dedicated, high-priority Redis node alongside a target service. It uses Node Affinity to run only on enterprise-grade hardware.

- **Node Affinity:** Uses `requiredDuringSchedulingIgnoredDuringExecution` to enforce scheduling only on nodes with the label `hardware-tier: powerhouse`.
- **Resource Allocations:**
  - CPU: Request `500m` / Limit `1000m`
  - Memory: Request `1Gi` / Limit `2Gi`
- **Dedicated Service:** Exposes internal container port `1234` under custom port bindings.

### 4. redis-service.yaml

The primary external interface for other applications running inside the K3s cluster.

- **Type:** ClusterIP (Internal cluster-only IP).
- **Port Mapping:** Exposes port `1234` (the standard Redis port) and maps it to target port `1234` internally. This allows client microservices to use standard connection strings while securing/customizing the container backend ports.

### 5. redis-deployment.yaml

A baseline, single-replica standard Deployment used as a fallback or a default isolated cache instance, running on port `1234` without custom affinity requirements.

---

## Important Configuration and Port Mappings

To avoid connection failures, take note of the intentional port-mapping translation implemented across these configuration files:

| Resource / Component | External Port | Internal Container Port | Notes |
|---|---|---|---|
| `redis-service.yaml` | `1234` | `1234` | Exposes standard Redis port to the cluster, but forwards to non-standard custom ports. |
| `redis-powerhouse.yaml` | `2345` | `2345` | Exposes service port 2345 targeting container port 2345. |
| `spider-web-grid.yaml` (Lanes) | `3456` (Headless Service) | `3456` (StatefulSets) | Headless service monitors port 3456, but the lane container ports listen on 3456. |
| `sentinel-config.yaml` | `7890` | N/A | Sentinel control port. Monitors the master at port 7890. |

---

## Tailoring Placeholders for Your Cluster

Before deploying these files to production, you must replace the template variables (containing `your-value`, `your-app`, `your-registry`, etc.) with your actual environment configurations.

### Required Replacements Checklist

| Placeholder String | Example Real-World Value | Description |
|---|---|---|
| `your-value` | `redis-lane-mac`, `redis-sentinel-conf` | The specific metadata name of the resource or component. |
| `your-app` | `redis` | The selector label used to bind services to deployments/StatefulSets. |
| `your-namespace` | `your-value` | The target Kubernetes namespace where your service mesh runs. |
| `your-registry/your-image:latest` | `docker.io/library/redis:7.2-alpine` | Your private or public container registry path and tag. |

---

## Deployment Steps

To apply these configurations to your K3s cluster, run the commands below in order:

### 1. Initialize Namespace and ConfigMaps

First, create the targeted namespace (if not already existing) and load the Sentinel configuration:

```sh
kubectl create namespace your-value --dry-run=client -o yaml | kubectl apply -f -

# Apply Sentinel ConfigMap
kubectl apply -f k3s-configs/redis/sentinel-config.yaml
```

### 2. Deploy the Heterogeneous Grid (StatefulSets)

Apply the Node-Affinity/Lane-based StatefulSets and headless routing service:

```sh
kubectl apply -f k3s-configs/redis/spider-web-grid.yaml
```

### 3. Deploy the Powerhouse Instance

Ensure your K3s nodes are properly labeled before applying this manifest:

```sh
# Label your powerhouse node(s)
kubectl label nodes <your-powerhouse-node-name> hardware-tier=powerhouse

# Apply Deployment and Service
kubectl apply -f k3s-configs/redis/redis-powerhouse.yaml
```

### 4. Apply Default Routing Services

```sh
kubectl apply -f k3s-configs/redis/redis-service.yaml
kubectl apply -f k3s-configs/redis/redis-deployment.yaml
```

---

## Verification & Troubleshooting

### Confirm Pod Scheduling across Lanes

Verify that the asymmetric pods are scheduled on their respective physical hardware nodes:

```sh
kubectl get pods -n spider-web -o wide --selector=app=your-app
```

Verify that the IP and NODE columns match your physical Mac, Mini PC, and HP Stream layouts.

### Test Sentinel Master Discovery

Exec into the sentinel container (or a debugging pod) and run `redis-cli` to check the status of Sentinel's connection to the master:

```sh
kubectl exec -it <sentinel-pod-name> -n spider-web -- redis-cli -p 1234 sentinel master mymaster
```

### Check Port Routing Status

Use an interactive alpine debug container to verify that port translation is functioning properly from cluster-internal resources:

```sh
kubectl run netdebug --rm -it --image=alpine -- sh
# Inside the container:
apk add --no-cache bind-tools tcptraceroute
nslookup your-service.your-value.svc.cluster.local
```