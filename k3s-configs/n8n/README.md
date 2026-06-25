# Operational & Architecture Guide: n8n Automation Engine & LocalSend Bridge

This repository contains the Kubernetes configuration manifests for the cluster's core workflow automation engine and its localized file transfer egress bridge. Located in the `k3s-configs/n8n` directory, these resources establish a highly available, high-performance automation ecosystem integrated with local infrastructure networks.

---

## System Architecture

The ecosystem consists of two primary services running in the cluster:

- **n8n Workflow Engine:** The automation platform running in worker/orchestration mode, backed by high-availability queuing (Redis Sentinel) and persistent relational storage (Postgres).
- **LocalSend Bridge:** A specialized helper microservice that acts as a proxy, translating HTTP API payloads from n8n into native LocalSend protocol traffic to beam files directly to physical devices (e.g., workstations, mobile devices, servers) on the local network.

```
graph TD
    subgraph K3s Cluster
        direction TB
        n8n[n8n Workflow Engine]
        LSBridge[LocalSend Bridge]
        Redis[Redis Sentinel Cluster<br>redis-grid:1234]
        Postgres[(Postgres Database)]
        
        n8n -->|State/Queue| Redis
        n8n -->|Metadata/History| Postgres
        n8n -->|Egress File Transfer| LSBridge
    end

    subgraph Physical Network
        TargetDevice[Target LocalSend Device<br>Mobile/Workstation]
    end

    LSBridge -->|LocalSend Protocol / TCP 4567| TargetDevice
```

---

## 1. n8n Workflow Engine (n8n.yaml)

This deployment configures a production-grade instance of n8n optimized for scalable, high-volume workflow processing.

### Key Architectural Patterns

**External Execution Mode (`N8N_RUNNERS_MODE: external`):** n8n is configured to decouple UI and scheduling tasks from workflow execution, ensuring that heavy workflow loads do not impact system responsiveness or crash the primary control plane.

**Redis Sentinel Integration:** Instead of a standalone Redis server, this setup integrates with a sentinel-backed Redis high-availability mesh (`redis-grid`) on port 1234. The queue management framework (Bull) uses Sentinel discovery with the master group name `mymaster` to guarantee lock and queue state persistence during failover events.

**HostPath Volume Bindings:** To handle larger file imports/exports and preserve execution states across pod restarts without cloud-dependent storage, the pod mounts local system paths directly (`/your/local/path and /your/local-path/.n8n-files`).

**Targeted Scheduling:** Due to its reliance on local host paths, the pod uses `nodeSelector` policies (e.g., target lane) and explicit tolerations to ensure it binds to the exact physical node where the underlying storage resources exist.

### Core Environment Variables

| Variable Name | Value / Type | Purpose |
|---|---|---|
| `N8N_RUNNERS_ENABLED` | `true` | Activates scale-out execution runners. |
| `N8N_RUNNERS_MODE` | `your_runners_mode` | Instructs the node to delegate running tasks to secondary runners. |
| `DB_TYPE` | `your_db_type` | Configures the metadata backend database type. |
| `DB_POSTGRESDB_HOST` | `your_postgres_host` | Internal cluster DNS service name for Postgres. |
| `DB_POSTGRESDB_USER` | `your_postgres_user` | Database system username. |
| `N8N_REDIS_HOST` | `your_redis_host` | DNS locator for the Sentinel cluster. |
| `N8N_REDIS_PORT` | `your_redis_port` | Standard Redis Sentinel monitor port. |
| `QUEUE_BULL_REDIS_TYPE` | `your_queue_type` | Directs Bull Queue to operate in Sentinel high-availability mode. |

---

## 2. LocalSend Bridge (localsend-bridge.yaml)

The LocalSend Bridge is a customized integration engine that provides a headless CLI adapter for the LocalSend protocol. This bridge allows n8n workflows (or any cluster-internal HTTP client) to programmatically send physical files to any active device running LocalSend on the local network.

### Execution Workflow

**Dynamic Architecture Bootstrapping:** Upon startup, the container inspects its CPU architecture (`uname -m`).

- On ARM-based nodes (e.g., Apple Silicon running virtualized k3s layers), it resolves to `aarch64` and pulls the corresponding ARM binary.
- On x86_64 nodes, it pulls the standard 64-bit Linux binary.

**LocalSend Engine Installation:** It dynamically downloads the highly efficient Go-based `localsnd` headless CLI tool (v0.5.23) developed by pepa65 and places it onto `/usr/local/bin/`.

**Flask HTTP Wrapper Initialization:** A Python-based Flask microservice is spawned to expose port 1234. It processes ingress requests, buffers payloads to local disk space, coordinates target node handshakes, executes the file transfer via the `localsnd` command line, and automatically cleans up after execution.

```
[ POST /beam ] ──> ( Flask Web Server ) ──> [ Write Temp File ] ──> [ Execute localsnd send <IP> <Path> ] ──> [ Delete Temp File ]
```

---

## API Specifications: LocalSend Bridge

The Flask microservice accepts standard HTTP multipart requests.

### Endpoint: `POST /beam`

**Request Headers**

- `Content-Type: multipart/form-data`

**Request Body (Form Parameters)**

- `ip` (String, Required): The target device IP address on the local area network.
- `file` (File Binary, Required): The file resource payload to transfer.

**Example Integration (cURL)**

```sh
curl -X POST http://localsend-bridge.your-namespace.svc.cluster.local:your-port/beam \
  -F "ip=192.168.1.150" \
  -F "file=@/workspace/invoice_report.pdf"
```

**Expected Responses**

- `200 OK`: File successfully received, buffered, and accepted by the remote LocalSend receiver client.
- `400 Bad Request`: Missing either the destination target IP parameter or the physical file payload.
- `500 Internal Server Error`: The target device rejected the payload, the CLI timed out, or the IP was unreachable. The console traceback log will bubble up raw stderr logs from the underlying binary.

---

## Security & Operational Safeguards

**Storage Lifecycles:** The LocalSend bridge buffers files in `/tmp` during transfer. The Python application manages file handles using strict `try...finally` sequences to ensure that local system disk allocation remains clear even during transmission failures.

**Access Tolerations:** Both services utilize precise tolerations and selector rules. n8n relies heavily on consistent directory architectures; if scheduled on an incorrect worker node, it will fail to start to prevent data corruption or split-brain behaviors. Ensure your nodes are labeled correctly before initializing deployments:

```sh
kubectl label nodes <your-node-name> lane=your-value
```

**Database & Cache Failover:** By coupling n8n with Sentinel and an external Postgres deployment, the system is protected against container restarts. Workflows running in the background are tracked inside Redis and will resume automatically on alternative nodes if an unexpected failover occurs.
