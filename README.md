# K8s PaaS — Kubernetes Resource Management API Server

> A RESTful API backend built on top of `client-go`, exposing Kubernetes resource operations — Pod lifecycle, Node scheduling, ConfigMap, and Secret management — through a clean, structured Go service layer.

---

## What Problem Does This Solve

Kubernetes' native API is powerful but verbose. Directly calling `kubectl` or raw API endpoints from application code requires deep familiarity with K8s object specs and error handling. This project wraps those operations into a structured, validated HTTP API that:

- Abstracts K8s resource CRUD behind a consistent REST interface
- Translates between user-friendly request models and native K8s object specs
- Enforces input validation and sensible defaults before any cluster operation
- Supports full Pod spec configuration — volumes, probes, affinity, tolerations, resource limits, and secrets — through a single unified request model

---

## Three Core Engineering Decisions

### Decision 1: Strict Request/Response Model Separation from K8s Objects

**Problem**: K8s native objects (e.g., `corev1.Pod`) are deeply nested and contain dozens of fields irrelevant to most API consumers. Exposing them directly couples the API contract to K8s internals.

**Decision**: Define independent `request` and `response` models for each resource type, with a dedicated `convert` layer handling bidirectional translation.

```
HTTP Request (JSON)
      │
      ▼
Request Model (e.g., pod_req.Pod)
      │  convert/pod/pod_req2k8s.go
      ▼
K8s Native Object (corev1.Pod)
      │  client-go API call
      ▼
K8s Cluster
      │
      ▼
K8s Native Object → convert/pod/pod_k8s2req.go → Response Model → HTTP Response
```

**Outcome**: The API surface stays stable even when K8s API versions change. Adding a new field only requires updating the model and converter, not the handler or service layer.

---

### Decision 2: Input Validation + Default Injection Before Cluster Operations

**Problem**: Submitting an incomplete Pod spec to the K8s API returns cryptic errors. Catching these at the boundary — before any cluster call — produces cleaner error messages and prevents partial state.

**Decision**: A dedicated `validate` layer runs before every write operation, checking required fields and injecting defaults for optional ones.

```go
// Example: pod_validate.go
// Required: Pod name, at least one container, container image
// Defaults injected: ImagePullPolicy → IfNotPresent, RestartPolicy → Always
```

**Outcome**: All validation errors are caught at the API boundary with human-readable messages. The service layer only receives well-formed, complete request objects.

---

### Decision 3: Node Scheduling Operations via Strategic Merge Patch

**Problem**: Updating Node labels and taints via a full object replace risks overwriting fields set by the K8s scheduler or other controllers.

**Decision**: Use `types.StrategicMergePatchType` to send only the diff, letting the API server merge changes safely.

```go
// node.go — label update via strategic merge patch
patchData := map[string]any{
    "metadata": map[string]any{
        "labels": labelsMap,  // "$patch": "replace" for atomic label replacement
    },
}
client.CoreV1().Nodes().Patch(ctx, name, types.StrategicMergePatchType, ...)
```

**Outcome**: Label and taint updates are safe for production clusters — no risk of overwriting scheduler-managed fields or triggering unnecessary pod evictions.

---

## API Overview

| Resource | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| Pod | POST | `/k8s/pod` | Create or update a Pod |
| Pod | GET | `/k8s/pod/:namespace` | List Pods or get detail |
| Pod | DELETE | `/k8s/pod/:namespace/:name` | Delete a Pod (immediate) |
| Namespace | GET | `/k8s/namespace` | List all namespaces |
| Node | GET | `/k8s/node` | List nodes or get detail |
| Node | PUT | `/k8s/node/label` | Update node labels |
| Node | PUT | `/k8s/node/taint` | Update node taints |
| ConfigMap | POST | `/k8s/configmap` | Create or update ConfigMap |
| ConfigMap | GET | `/k8s/configmap/:namespace` | List or get detail |
| ConfigMap | DELETE | `/k8s/configmap/:namespace/:name` | Delete ConfigMap |
| Secret | POST | `/k8s/secret` | Create or update Secret |
| Secret | GET | `/k8s/secret/:namespace` | List or get detail |
| Secret | DELETE | `/k8s/secret/:namespace/:name` | Delete Secret |

---

## Pod Spec Coverage

The unified Pod request model covers the full range of K8s Pod configuration:

| Category | Supported Fields |
| :--- | :--- |
| **Basic** | Name, Namespace, Labels, RestartPolicy |
| **Networking** | HostNetwork, HostName, DnsPolicy, DnsConfig, HostAliases |
| **Containers** | Init containers, main containers, image pull policy, resource limits (CPU/Mem) |
| **Health Checks** | Liveness / Readiness / Startup probes (HTTP, TCP, Exec) |
| **Volumes** | EmptyDir, HostPath, NFS, ConfigMap, Secret, PVC, Downward API |
| **Scheduling** | NodeName, NodeSelector, Node affinity, Pod affinity/anti-affinity, Tolerations |
| **Security** | ImagePullSecret, privileged mode |

---

## Tech Stack

| Layer | Technology |
| :--- | :--- |
| Language | Go 1.18 |
| HTTP Framework | Gin |
| K8s Client | client-go v0.25.2 |
| Config | Viper |
| Containerization | Docker |

---

## Repository Structure

```
k8s-paas-main/
├── api/k8s/          # HTTP handlers (Pod / Node / ConfigMap / Secret / Namespace)
├── service/          # Business logic, client-go calls
├── model/            # Request & response structs (per resource)
├── convert/          # Bidirectional K8s object ↔ model translation
├── validate/         # Input validation + default injection
├── router/           # Route registration
├── middleware/        # CORS
├── initiallize/      # App bootstrap (K8s client, router, config)
├── global/           # Global singletons (KubeConfigSet)
├── k8s_use/          # Reference YAML manifests (volumes, scheduling, probes)
└── docs/             # API request examples (JSON)
```

## 📝 Related Articles

📚 [Kubernetes Internals Series on dev.to](https://dev.to/jamesli/series/39827)

