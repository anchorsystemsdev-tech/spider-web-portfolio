# PostgreSQL 18 Service on K3s

This directory contains the Kubernetes orchestration manifests for deploying a high-performance, single-node PostgreSQL 18 database instance on a K3s cluster.

The configuration is optimized for bare-metal/local deployments (specifically targeted for local macOS partitions with dedicated storage) and is pre-configured to act as the primary database backend for automation tooling such as n8n.

---

## 1. Architectural Overview

```
                        +------------------------------------------+
                        |              K3s Cluster                 |
                        |                                          |
                        |   +------------------+                   |
                        |   |  Service         |                   |
                        |   |  Port: <port>    |                   |
                        |   +--------+---------+                   |
                        |            |                             |
                        |            v                             |
                        |   +------------------+                   |
                        |   |  StatefulSet     |                   |
                        |   |  Replicas: 1     |                   |
                        |   +--------+---------+                   |
                        |            | (HostPath Mount)            |
                        +------------+-----------------------------+
                                     |
                                     v
                        +------------------------------------------+
                        |          Underlying Host OS              |
                        |  Path: /your/local/path                  |
                        |  (e.g. Dedicated 17GB Partition)  |
                        +------------------------------------------+
```

### Key Design Patterns

**Statefulset Topology:** Uses a StatefulSet with a single replica (`replicas: 1`) to ensure stable network identifiers, graceful startup/shutdown guarantees, and strict enforcement of a single-writer pattern.

**Node Anchoring & Taints:** Bypasses dynamic scheduling to bind the pod to a specific host machine (e.g., a dedicated Mac mini/Studio node) using a `nodeSelector` and control-plane tolerations. This minimizes disk latency and ensures the container runs immediately adjacent to its hostpath storage.

**Direct HostPath Storage:** Leverages a direct `hostPath` mount to a local directory. This is optimal for development/edge environments utilizing high-speed local NVMe partitions (such as a dedicated 17GB partition) where standard CSI overhead (like Longhorn or Ceph) is undesirable.

---

## 2. File Directory Structure

```
k3s-configs/postgres/
└── postgres.yaml     # Combined Manifest (StatefulSet & ClusterIP Service)
```

---

## 3. Configuration Deep Dive

The `postgres.yaml` file contains two primary resource definitions separated by standard YAML dividers (`---`):

### 1. StatefulSet (apps/v1)

This controller manages the deployment and scaling of the PostgreSQL container, ensuring stateful guarantees.

**Targeting & Tolerations:**
```yaml
nodeSelector:
  lane: your-value # Labels the specific host architecture
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```
This allows the PostgreSQL database to run directly on control-plane nodes if needed, ensuring high availability even in single-node master topologies.

**Storage Mount:**
Maps host machine directories directly to the database container to persist transactional data bypass-style.

**Environment Configuration:**
Preconfigured to ingest n8n connection parameters (n8n user schema paired with legacy parameters from Docker Compose migrations).

### 2. Service (v1)

Exposes the database internally to other pods in the cluster (e.g., n8n, pgAdmin, custom APIs).

**Port Mapping:** Forwards the non-standard or standard external connection requests down to the cluster port seamlessly.

---

## 4. Parameter Customization Matrix

Before applying this configuration, replace the placeholders (`your-value`, `your-namespace`, etc.) with your targeted cluster parameters.

| YAML JSONPath | Placeholder | Recommended Value | Description |
|---|---|---|---|
| `metadata.namespace` | `your-namespace` | `databases` or `n8n` | Target namespace for resources. |
| `metadata.name` | `your-value` | `postgres` | Resource identifier. |
| `spec.template.spec.containers[0].image` | `your-registry/your-image:latest` | `postgres:18-alpine` | Container image registry and tag. |
| `spec.template.spec.nodeSelector.lane` | `your-value` | `database-node` | Label value of your targeted database host machine. |
| `spec.template.spec.containers[0].env[0].name` | `your-value` | `POSTGRES_USER` | Environment key for the master DB user. |
| `spec.template.spec.containers[0].env[1].name` | `your-value` | `POSTGRES_PASSWORD` | Environment key for the user password. |
| `spec.template.spec.containers[0].env[2].name` | `your-value` | `POSTGRES_DB` | Environment key for the default database name. |
| `spec.template.spec.containers[0].volumeMounts[0].name` | `your-value` | `postgres-data` | Logical name of the volume within the pod. |
| `spec.template.spec.containers[0].volumeMounts[0].mountPath` | `/your/local/path` | `/var/lib/postgresql/data` | Database engine internal directory path. |
| `spec.volumes[0].hostPath.path` | `/your/local/path` | `/Volumes/DataPartition/postgres` | Host absolute directory path (e.g., Mac mount). |
| `spec.ports[0].port` | `1234` | `1234` | Port exposed by the Service to the cluster. |

---

## 5. Deployment Guide

### Step 1: Prepare Host Storage

Ensure the directory on your physical machine (e.g., your dedicated partition) exists and has the correct permissions for the PostgreSQL container user (UID 70 for official Docker Postgres images):

```sh
# Create directory on host machine
sudo mkdir -p /your/local/path

# Assign ownership to the PostgreSQL system user inside the container
sudo chown -R 70:70 /your/local/path
```

### Step 2: Replace Placeholders and Apply

You can use a utility like `envsubst`, find-and-replace in your IDE, or run a `sed` command to swap parameters:

```sh
# Apply the manifest directly using kubectl
kubectl apply -f postgres.yaml
```

### Step 3: Verify the Deployment

Verify Pod Status:
```sh
kubectl get pods -n your-namespace -l app=your-app
```

Review DB Startup Logs:
```sh
kubectl logs -f -n your-namespace statefulset/your-value
```

Verify Internal Cluster Access:
```sh
kubectl run pg-client --rm -i --tty --image postgres:18-alpine --namespace your-namespace -- \
  psql -h your-value -U your-value -d your-value
```

---

## 6. Security & Operational Best Practices

**Sensitive Secrets:** In production environments, replace inline environment variable definitions (`POSTGRES_PASSWORD`) with references to Kubernetes Secrets:
```yaml
valueFrom:
  secretKeyRef:
    name: postgres-secrets
    key: database-password
```

**Backups:** Since a `hostPath` mount is used, set up a native cron job or Kubernetes CronJob to run `pg_dump` on a schedule, saving backups outside of the host path to avoid loss if physical host drives fail.