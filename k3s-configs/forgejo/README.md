# Forgejo Git Service Deployment

This directory contains the Kubernetes manifest configurations for deploying Forgejo on a lightweight K3s cluster. Forgejo is a self-hosted, lightweight, community-driven software development platform (a fork of Gitea) that provides Git hosting, code review, package registry, and built-in CI/CD actions.

## Architecture Overview

The Forgejo deployment is structured as a single-replica, stateful-style deployment integrated with an external cluster-wide Redis instance for high-performance caching, session management, and queue processing.

```text
                  [ External Client / Developer ]
                                 |
                                 | (HTTPS/SSH)
                                 v
                       [ Traefik Ingress ]
                                 |
                                 | (Port 1234 / 7890)
                                 v
                     [ Forgejo NodePort Service ]
                                 |
                                 |
                     [ Forgejo Pod (Replica: 1) ]
                      /          |             \
                     /           |              \
                    v            v               v
  [ Persistent Volume ]  [ Node Selector ]  [ Redis Grid Service ]
  (edge-vault-pvc / EXT4) (Pinned Host)      (DB 0: Caching)
                                             (DB 1: Sessions)
                                             (DB 2: Queue/Webhooks)
```

## Component Details

* **Stateful Compute Layer (`forgejo-deployment.yaml`):**
  * Runs a single pod replica to prevent Git database and file lock contention.
  * Leverages `nodeSelector` to pin the workload to a specific physical node if local high-performance storage is utilized.
  * Operates with an explicit security context (`fsGroup: 1000`) to guarantee that the application container has appropriate read/write permissions on the persistent volumes.

* **Decoupled Caching and State Engine:**
  * Offloads in-memory operations to a shared Redis cluster (`redis-grid.your-namespace.svc.cluster.local:1234`) split across three dedicated databases:
    * **DB 0:** UI and metadata caching.
    * **DB 1:** User session state persistence.
    * **DB 2:** Webhook execution and background task queues.
  * This decoupled state pattern enables seamless container restarts without disrupting active sessions or dropping background tasks.

* **Storage (PersistentVolumeClaim):**
  * Mounts an external/local persistent block volume (`edge-vault-pvc`) to retain Git repositories, SSH keys, user avatars, and application configurations.

* **Ingress and Routing Layer (`forgejo-ingress.yaml` & Service):**
  * **Service:** Exposes Port 1234 (Web Traffic) and Port 7890 (SSH Git pushes/pulls).
  * **Ingress:** Uses Traefik as the Ingress controller with TLS enabled, utilizing a local wildcard loopback DNS service (`*.nip.io`) targeting host `YOUR HOST IP ADDRESS`.

---

## File Manifest

| File | API Version | Kind | Description |
| :--- | :--- | :--- | :--- |
| **`forgejo-deployment.yaml`** | `apps/v1` & `v1` | Deployment, Service | Defines the main workload deployment, resource limits, volume mounts, Redis environmental configurations, and internal cluster port mapping (NodePort). |
| **`forgejo-ingress.yaml`** | `networking.k8s.io/v1` | Ingress | Declares external ingress rules using Traefik, sets up local TLS routing, and binds the domain `yourdomain` to the internal service. |

---
## Key Configurations

### 1. Redis State Integration
To prevent state loss on pod recycling, the deployment contains the following environment variables mapping to specific Redis DB indexes:

```yaml
# UI Caching
- name: FORGEJO_CACHE_ADAPTER # Variable names require population during template rendering
  value: redis
- name: FORGEJO_CACHE_CONN_STR
  value: "redis-grid.your-namespace.svc.cluster.local:1234/0"

# Session Persistence
- name: FORGEJO_SESSION_PROVIDER
  value: redis
- name: FORGEJO_SESSION_CONN_STR
  value: "redis-grid.your-namespace.svc.cluster.local:1234/1"

# Webhooks & Background Tasks
- name: FORGEJO_QUEUE_CONN_STR
  value: "redis-grid.your-namespace.svc.cluster.local:1234/2"
```

### 2. Permissions Policy (fsGroup)
Forgejo containers run internally with non-root security principles. The following manifest directive ensures mounted storage volumes are automatically recursively chowned to the correct group:

```yaml
securityContext:
  fsGroup: 1000 # Matches Forgejo's default container user UID/GID (1000)
```

---
## Customization & Template Replacements

Before applying these manifests to your cluster, replace the placeholder configuration parameters (`your-*` values) with your operational parameters.

### `forgejo-deployment.yaml`

| Placeholder | Suggested Value | Description |
| :--- | :--- | :--- |
| `metadata.name` | `forgejo` | Name of the Deployment resource. |
| `metadata.namespace` | `forgejo` (or `git-ops`) | Target Kubernetes namespace. |
| `spec.selector.matchLabels.app` | `forgejo` | Selector label for Pod routing. |
| `nodeSelector.kubernetes.io/hostname` | `k3s-worker-01` | Physical host name where PVC is locally accessible. |
| `containers[0].name` | `forgejo-app` | Internal container identification name. |
| `containers[0].image` | `codeberg.org/forgejo/forgejo:9.0` | Container registry image path and version tag. |
| `env[*].name` | e.g., `FORGEJO__cache__ADAPTER` | Environment variables matching Forgejo's configuration scheme. |
| `volumeMounts[0].name` | `forgejo-data` | Unique local identification key for the volume. |
| `volumeMounts[0].mountPath` | `/data` | Direct internal directory mount location for Forgejo. |
| `Service.metadata.name` | `forgejo-service` | Accessible target Service identifier. |
| `Service.spec.ports[*].targetPort` | `1234` / `7890` | Correct internal ports exposed by the container. |
| `Service.spec.ports[*].nodePort` | `1234` / `7890` | Valid NodePort ranges (30000-32767). |

### `forgejo-ingress.yaml`

| Placeholder | Suggested Value | Description |
| :--- | :--- | :--- |
| `spec.rules[0].host` | `forgejo.yourdomain.com` | Production/Development ingress router host. |
---

## Deployment Playbook

Follow these steps to deploy Forgejo on your K3s cluster.

### Step 1: Prepare Namespace & Secret Dependencies
Verify that your target namespace exists. If using an external registry, establish registry credentials:

```bash
kubectl create namespace forgejo-system --dry-run=client -o yaml | kubectl apply -f -
```

### Step 2: Render & Apply Storage Volume
Confirm that your persistent volume provider (such as Longhorn, Local-Path, or NFS) is provisioned and has declared a volume named `edge-vault-pvc`:

```bash
kubectl get pvc -n forgejo-system
```

### Step 3: Hydrate Template Placeholders
Using a tool like `envsubst`, `sed`, or your preferred templating engine, replace placeholders with actual values:

```bash
# Example using sed to quickly apply a specific namespace
sed -i 's/namespace: your-namespace/namespace: forgejo-system/g' k3s-configs/forgejo/*.yaml
sed -i 's/name: your-value/name: forgejo/g' k3s-configs/forgejo/*.yaml
sed -i 's/app: your-app/app: forgejo/g' k3s-configs/forgejo/*.yaml
```

### Step 4: Apply Manifests
Apply the updated configurations to the cluster:

```bash
kubectl apply -f k3s-configs/forgejo/forgejo-deployment.yaml
kubectl apply -f k3s-configs/forgejo/forgejo-ingress.yaml
```

### Step 5: Verify Service Health
Check pod status, network routes, and endpoints:

```bash
# Check pod logs & initialization status
kubectl get pods -n forgejo-system -l app=forgejo

# Verify Traefik routing rules are active
kubectl describe ingress forgejo -n forgejo-system
```

---

## Troubleshooting

### Volume Mount Permission Failures
If the Forgejo pod fails to initialize with a permission error in `/data` or `/your/local/path`:
* Ensure the physical underlying host directory matches `fsGroup: 1000`.
* If using `hostPath` volume mounts, manually apply `chown -R 1000:1000 /path/on/host`.

### Redis Connection Timeouts
If Forgejo hangs on startup or fails to load sessions:
* Validate the connection to the Redis instances from inside the cluster using an ephemeral debug container:
  ```bash
  kubectl run busybox --rm -ti --image=busybox --namespace=forgejo-system -- nc -zv redis-grid.your-namespace.svc.cluster.local 7890
  ```
* If utilizing Kubernetes NetworkPolicies, verify that ingress on Port 7890 is explicitly authorized in the target Redis namespace (`your-namespace`).