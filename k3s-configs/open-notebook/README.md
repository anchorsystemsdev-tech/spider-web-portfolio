# Open Notebook Stack

This directory contains the Kubernetes manifests required to deploy the Open Notebook service—a containerized application stack backed by a persistent SurrealDB instance.

The architecture is specifically tailored for lightweight edge environments (e.g., K3s clusters running on mini-PCs), emphasizing high performance, low-overhead host networking, and stateful storage isolation.

## 1. Architectural Overview

The Open Notebook service is split into two logical layers:

* **Application Layer (`open-notebook.yaml`):** Runs the Open Notebook web application. It uses host networking to bypass virtual container networking latencies, targeting edge nodes classified as `your-value`.
* **Database Layer (`surrealdb.yaml`):** An instance of SurrealDB using a high-performance RocksDB storage engine. It is deployed as a StatefulSet with dedicated disk provisioning to ensure transactional data safety.

```text
                  ┌──────────────────────────────────────────┐
                  │                Host Network              │
                  │   Direct Access: Port 1234 / 7890        │
                  └────────────────────┬─────────────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │    Open Notebook Pod        │
                        │    (Deployment, Replicas: 1)│
                        └──────────────┬──────────────┘
                                       │
                                       │ (WebSockets / RPC over SVC)
                                       ▼
                  ┌──────────────────────────────────────────┐
                  │         SurrealDB Service                │
                  │         Port: 1234                       │
                  └────────────────────┬─────────────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │       SurrealDB Pod         │
                        │    (StatefulSet, Replicas:1)│
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │      Persistent Volume      │
                        │      (edge-vault-pvc)       │
                        └─────────────────────────────┘
```

---

## 2. Component Analysis

### A. Open Notebook Application Deployment (`open-notebook.yaml`)
This deployment manages the stateless runtime of the Open Notebook system.

* **Host Networking (`hostNetwork: true`):** Bypasses the Kubernetes CNI plugin (e.g., Flannel, Calico) entirely. The pod binds directly to the network interface of the physical node, achieving bare-metal performance.
* **Targeted Scheduling:** Explicitly restricted to nodes labeled with `node-role: minipc` to optimize resource use across heterogeneous clusters.
* **Resource Management:** Implements soft limits and hard safety thresholds:
  * **CPU:** 250m request up to 1000m (1 Core) limit.
  * **Memory:** 512Mi request up to 2Gi limit.
* **Networking Configuration:**
  * Exposes container ports 1234 and 7890.
  * Configured to establish a stateful WebSocket RPC link to the SurrealDB service at `ws://surrealdb-service.your-value.svc.cluster.local:7890/rpc`.

### B. SurrealDB Storage Engine (`surrealdb.yaml`)
Provides the persistent data storage engine using SurrealDB.

* **Stateful Design:** Deployed as a StatefulSet rather than a standard deployment. This guarantees stable, unique network identifiers and safe volume mounting.
* **Permission Initialization (Init Container):**
  * Because SurrealDB runs as a non-root user (65532:65532) for security, it cannot inherently write to host-mounted volumes owned by root.
  * An init-container (`runAsUser: 0`) executes a pre-flight execution of `chown -R 65532:65532` on the mapped directory `/mydata` before the core engine boots.
* **RocksDB Storage Backend:** Instead of using an in-memory database, SurrealDB initiates with the file-backed RocksDB engine target `rocksdb:///mydata/mydatabase.db`, ensuring all ledger updates are written directly to disk.

---

## 3. Configuration & Parameter Mapping

To operationalize these templates, replace all occurrences of `your-value`, `your-namespace`, `your-app`, and `your-registry/your-image:latest` with actual configurations.

### Environment Variable & Secret Requirements

#### 1. Open Notebook Application
The deployment expects the following environment configurations:

| Parameter Key | Purpose |
| :--- | :--- |
| `API_ENDPOINT_URL` | Upstream endpoint API for processing. |
| `SURREALDB_URL` | Network connection URL to database. |
| `DB_NAMESPACE` | SurrealDB target namespace. |
| `DB_NAME` | SurrealDB target database name. |
| `DB_CLIENT` | Client Identifier. |
| `DB_USER` | Database administrative user credentials (via Secret Injection). |
| `DB_PASS` | Database administrative password credentials (via Secret Injection). |

#### 2. SurrealDB
The database engine expects credentials to be passed during initialization:

| Parameter Key | Source (Secret Key) | Purpose |
| :--- | :--- | :--- |
| `SECURE_USER` | `REDACTED_SECRET` | Defines the master username for database initialization. |
| `SECURE_PASS` | `REDACTED_SECRET` | Defines the master password for database initialization. |

---

## 4. Production Considerations & Critical Warnings

### ⚠️ Port Mismatch in Service Definition
In `surrealdb.yaml`, review the Service port configuration:
```yaml
spec:
  ports:
    - protocol: TCP
      port: 1234
      targetPort: 1234  
```
### ⚠️ Volume Path Synchronization
The init-container runs permission fixes on `/mydata`, but the volume mount maps to `/your/local/path`:
```yaml
volumeMounts:
- name: your-value
  mountPath: /your/local/path
  subPath: surreal-db-vault
```
To ensure RocksDB can access `/mydata/mydatabase.db` properly, update the volume's `mountPath` across both containers to:
```yaml
mountPath: /mydata
```

### ⚠️ Host Networking Port Clashes
By using `hostNetwork: true` on the Open Notebook deployment, ports 1234 and 7890 are bound directly to the host IP. Only one replica of this pod can run on any single node at a given time. Scaling replicas > 1 on the same physical host will result in a standard port binding crash loop.

---

## 5. Deployment Step-by-Step

### Step 1: Prepare Namespace and Secrets
Ensure your target namespace exists and configure the required credentials:

```bash
# Create namespace
kubectl create namespace <your-namespace>

# Create SurrealDB admin secret
kubectl create secret generic surrealdb-creds \
  --namespace=<your-namespace> \
  --from-literal=username='admin' \
  --from-literal=password='your-secure-password'
```

### Step 2: Provision Volume Claims
Ensure your PersistentVolume (PV) and PersistentVolumeClaim (`your-value-pvc`) are deployed in the cluster and ready to be bound.

### Step 3: Deploy the Stack
Deploy the manifests in order (Database first, then Application):

```bash
# Apply SurrealDB Storage layer
kubectl apply -f k3s-configs/open-notebook/surrealdb.yaml

# Monitor Database initialization 
kubectl rollout status statefulset/<db-statefulset-name> -n <your-namespace>

# Apply the Notebook Frontend Application
kubectl apply -f k3s-configs/open-notebook/open-notebook.yaml
```

### Step 4: Verification
Verify connectivity by checking logs:

```bash
# Verify DB is listening
kubectl logs -f statefulset/<db-statefulset-name> -n <your-namespace>

# Test WebSockets connection from Open Notebook
kubectl logs -f deployment/<app-deployment-name> -n <your-namespace>
```