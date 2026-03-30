# Kubernetes Deployment Plan — AI Native Task Manager

**Author:** Muhammad Faisal Laiq Siddiqui  
**Version:** 1.0  
**Date:** 2026-03-30

---

## Overview

The AI Native Task Manager is a multi-component application consisting of:

| Component | Role |
|---|---|
| **UI Interface** | Frontend served to end users |
| **Backend APIs** | Core business logic and REST/GraphQL endpoints |
| **Task Agent** | AI-powered agent that processes and executes tasks |
| **Notification Service** | Sends alerts via email/SMS/push |

---

## 1. Namespaces

Namespaces provide logical isolation between environments and components.

```
task-manager/
├── task-manager-prod       # Production workloads
├── task-manager-staging    # Staging/QA environment
└── task-manager-monitoring # Prometheus, Grafana, logging
```

**Namespace definitions:**

| Namespace | Purpose | Resource Quota |
|---|---|---|
| `task-manager-prod` | All production pods | CPU: 16 cores, RAM: 32Gi |
| `task-manager-staging` | Pre-release testing | CPU: 4 cores, RAM: 8Gi |
| `task-manager-monitoring` | Observability stack | CPU: 4 cores, RAM: 8Gi |

---

## 2. Pods and Deployments / StatefulSets

### 2.1 UI Interface — Deployment

- **Type:** `Deployment` (stateless, horizontally scalable)
- **Replicas:** 3 (for high availability)
- **Pod Labels:** `app: ui-interface`, `tier: frontend`
- **Restart Policy:** Always
- **Strategy:** RollingUpdate (maxUnavailable: 1, maxSurge: 1)

| Setting | Value |
|---|---|
| Container Image | `task-manager/ui:latest` |
| Port | 3000 (React/Next.js) |
| Liveness Probe | HTTP GET `/health` every 30s |
| Readiness Probe | HTTP GET `/ready` every 10s |

---

### 2.2 Backend APIs — Deployment

- **Type:** `Deployment` (stateless REST/GraphQL server)
- **Replicas:** 3
- **Pod Labels:** `app: backend-api`, `tier: backend`
- **Strategy:** RollingUpdate

| Setting | Value |
|---|---|
| Container Image | `task-manager/api:latest` |
| Port | 8000 (FastAPI) |
| Liveness Probe | HTTP GET `/api/health` every 30s |
| Readiness Probe | HTTP GET `/api/ready` every 10s |

---

### 2.3 Task Agent — Deployment

- **Type:** `Deployment` (stateless AI agent workers)
- **Replicas:** 2 (scale based on queue depth via HPA)
- **Pod Labels:** `app: task-agent`, `tier: ai`
- **Strategy:** RollingUpdate

> **Note:** Task Agent connects to an external LLM API (e.g., Anthropic Claude) and processes tasks asynchronously from a queue.

| Setting | Value |
|---|---|
| Container Image | `task-manager/agent:latest` |
| Port | 8001 (internal agent API) |
| Liveness Probe | HTTP GET `/agent/health` every 60s |
| Readiness Probe | HTTP GET `/agent/ready` every 15s |

**Horizontal Pod Autoscaler (HPA):**
- Min replicas: 2
- Max replicas: 10
- Scale trigger: CPU > 70% or custom metric (queue depth > 50 tasks)

---

### 2.4 Notification Service — Deployment

- **Type:** `Deployment` (stateless event-driven service)
- **Replicas:** 2
- **Pod Labels:** `app: notification-service`, `tier: backend`
- **Strategy:** RollingUpdate

| Setting | Value |
|---|---|
| Container Image | `task-manager/notifier:latest` |
| Port | 8002 |
| Liveness Probe | HTTP GET `/notify/health` every 30s |

---

### 2.5 Database — StatefulSet

- **Type:** `StatefulSet` (stateful — requires stable identity and persistent storage)
- **Replicas:** 3 (1 primary + 2 read replicas)
- **Pod Labels:** `app: postgres`, `tier: database`
- **PersistentVolumeClaim:** 50Gi per pod (SSD storage class)

| Setting | Value |
|---|---|
| Container Image | `postgres:15` |
| Port | 5432 |
| Storage | PVC — `task-db-storage`, 50Gi each |

---

### 2.6 Message Queue — StatefulSet

- **Type:** `StatefulSet` (RabbitMQ or Redis Streams)
- **Replicas:** 3
- **Pod Labels:** `app: message-queue`, `tier: infra`
- **PersistentVolumeClaim:** 20Gi per pod

Used for async communication between Backend APIs → Task Agent → Notification Service.

---

## 3. Services and Their Types

| Service Name | Target | Type | Port | Purpose |
|---|---|---|---|---|
| `ui-service` | UI Interface pods | **LoadBalancer** | 80/443 | Expose UI to the internet |
| `api-service` | Backend API pods | **ClusterIP** | 8000 | Internal access from UI and Agent |
| `api-external-service` | Backend API pods | **NodePort** | 30080 | Dev/staging external access |
| `agent-service` | Task Agent pods | **ClusterIP** | 8001 | Internal only — triggered by API |
| `notification-service` | Notification pods | **ClusterIP** | 8002 | Internal only — triggered by Agent |
| `postgres-service` | PostgreSQL StatefulSet | **ClusterIP** | 5432 | Database access (headless for StatefulSet) |
| `queue-service` | Message Queue StatefulSet | **ClusterIP** | 5672 | Queue access (headless) |

**Service Type Justification:**

- **LoadBalancer** for UI: Needs a public IP via cloud provider (AWS ELB, GCP LB, Azure LB)
- **ClusterIP** for internal services: Microservices communicate within cluster — no external exposure needed
- **NodePort** for API in staging: Allows developers to test without a LoadBalancer

---

## 4. Resource Requests and Limits

Resource management prevents noisy-neighbor problems and ensures fair scheduling.

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| UI Interface | 100m | 500m | 128Mi | 512Mi |
| Backend API | 250m | 1000m | 256Mi | 1Gi |
| Task Agent | 500m | 2000m | 512Mi | 2Gi |
| Notification Service | 100m | 500m | 128Mi | 512Mi |
| PostgreSQL (per pod) | 500m | 2000m | 1Gi | 4Gi |
| Message Queue (per pod) | 250m | 1000m | 512Mi | 2Gi |

> **Rule of thumb:** Request = guaranteed minimum. Limit = hard cap. Always set both.

---

## 5. ConfigMaps

ConfigMaps store non-sensitive configuration data as key-value pairs.

### 5.1 `ui-config`
```
API_BASE_URL=http://api-service:8000
APP_TITLE=AI Task Manager
FEATURE_FLAGS={"darkMode": true, "aiAssist": true}
LOG_LEVEL=info
```

### 5.2 `api-config`
```
DATABASE_HOST=postgres-service
DATABASE_PORT=5432
DATABASE_NAME=taskmanager
QUEUE_HOST=queue-service
QUEUE_PORT=5672
MAX_WORKERS=4
LOG_LEVEL=info
CORS_ORIGINS=https://taskmanager.example.com
```

### 5.3 `agent-config`
```
ANTHROPIC_MODEL=claude-sonnet-4-20250514
MAX_CONCURRENT_TASKS=5
TASK_TIMEOUT_SECONDS=300
RETRY_ATTEMPTS=3
QUEUE_TOPIC=task-queue
LOG_LEVEL=info
```

### 5.4 `notification-config`
```
SMTP_HOST=smtp.sendgrid.net
SMTP_PORT=587
EMAIL_FROM=noreply@taskmanager.example.com
PUSH_PROVIDER=firebase
SMS_PROVIDER=twilio
LOG_LEVEL=info
```

---

## 6. Secrets Management and Expiry Handling

Secrets store sensitive credentials. They are base64-encoded in Kubernetes and should be encrypted at rest.

### 6.1 Secrets List

| Secret Name | Contains | Used By |
|---|---|---|
| `db-credentials` | `DB_USER`, `DB_PASSWORD` | Backend API, Task Agent |
| `anthropic-api-key` | `ANTHROPIC_API_KEY` | Task Agent |
| `queue-credentials` | `QUEUE_USER`, `QUEUE_PASSWORD` | API, Agent, Notifier |
| `smtp-credentials` | `SMTP_USER`, `SMTP_PASSWORD` | Notification Service |
| `twilio-credentials` | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` | Notification Service |
| `firebase-credentials` | `FIREBASE_SERVER_KEY` | Notification Service |
| `jwt-secret` | `JWT_SECRET_KEY` | Backend API |
| `tls-cert` | TLS certificate + private key | Ingress / LoadBalancer |

### 6.2 Secrets Expiry and Rotation Strategy

| Strategy | Detail |
|---|---|
| **External Secret Operator** | Integrate with AWS Secrets Manager / HashiCorp Vault. Secrets auto-sync to K8s Secrets on rotation. |
| **Secret Rotation Schedule** | API keys: every 90 days. DB passwords: every 30 days. JWT secret: every 7 days. |
| **TLS Certificate** | Use cert-manager with Let's Encrypt. Auto-renews 30 days before expiry. |
| **Immutable Secrets** | Mark production secrets as `immutable: true` to prevent accidental edits. |
| **Audit Logging** | Enable K8s audit logs to track secret access events. |
| **No Secrets in Images** | Never bake secrets into container images. Always inject at runtime via env vars or volume mounts. |

---

## 7. RBAC — Roles and RoleBindings

RBAC restricts which service accounts and users can access which resources.

### 7.1 Service Accounts

| Service Account | Namespace | Used By |
|---|---|---|
| `sa-ui` | task-manager-prod | UI Interface pods |
| `sa-api` | task-manager-prod | Backend API pods |
| `sa-agent` | task-manager-prod | Task Agent pods |
| `sa-notifier` | task-manager-prod | Notification Service pods |

### 7.2 Role: `api-role`
**Permissions for Backend API:**

| Resource | Verbs |
|---|---|
| ConfigMaps | get, list |
| Secrets | get |
| Pods | get, list |
| Services | get, list |

**RoleBinding:** Binds `api-role` → `sa-api` in `task-manager-prod`

---

### 7.3 Role: `agent-role`
**Permissions for Task Agent:**

| Resource | Verbs |
|---|---|
| ConfigMaps | get, list |
| Secrets | get |
| Jobs | create, get, list, delete |
| Pods | get, list, watch |

**RoleBinding:** Binds `agent-role` → `sa-agent` in `task-manager-prod`

---

### 7.4 Role: `notifier-role`
**Permissions for Notification Service:**

| Resource | Verbs |
|---|---|
| ConfigMaps | get, list |
| Secrets | get |

**RoleBinding:** Binds `notifier-role` → `sa-notifier` in `task-manager-prod`

---

### 7.5 ClusterRole: `monitoring-reader`
For Prometheus/Grafana to scrape metrics across namespaces:

| Resource | Verbs |
|---|---|
| Pods, Nodes, Services | get, list, watch |
| Endpoints | get, list, watch |
| Metrics | get |

**ClusterRoleBinding:** Binds `monitoring-reader` → `sa-monitoring` in `task-manager-monitoring`

---

## 8. Inter-Service Communication

### 8.1 Communication Flow Diagram

```
Internet
   │
   ▼
[LoadBalancer]
   │
   ▼
[UI Interface :3000]
   │  (HTTP REST)
   ▼
[Backend API :8000]  ──── (ClusterIP) ────▶  [PostgreSQL :5432]
   │
   │ (Publishes to Queue)
   ▼
[Message Queue :5672]
   │
   ├──▶ [Task Agent :8001]  ──── (HTTPS) ──▶  Anthropic Claude API (external)
   │
   └──▶ [Notification Service :8002]  ──── (HTTPS) ──▶  SendGrid / Twilio / Firebase (external)
```

### 8.2 Communication Protocols

| From | To | Protocol | Method |
|---|---|---|---|
| User Browser | UI Interface | HTTPS | HTTP/2 via LoadBalancer |
| UI Interface | Backend API | HTTP | REST via ClusterIP DNS |
| Backend API | PostgreSQL | TCP | PostgreSQL protocol |
| Backend API | Message Queue | AMQP | Publish message |
| Message Queue | Task Agent | AMQP | Subscribe/consume |
| Message Queue | Notification Service | AMQP | Subscribe/consume |
| Task Agent | Anthropic API | HTTPS | External REST API |
| Notification Service | SendGrid/Twilio | HTTPS | External REST API |

### 8.3 Service Discovery

All services communicate using Kubernetes internal DNS:
- Format: `<service-name>.<namespace>.svc.cluster.local`
- Example: `api-service.task-manager-prod.svc.cluster.local:8000`

### 8.4 Network Policies

- UI pods: Can only talk to `api-service`
- API pods: Can talk to `postgres-service` and `queue-service`
- Agent pods: Can talk to `queue-service` only (egress to Anthropic API allowed)
- Notification pods: Egress allowed to external SMTP/SMS/Push providers only
- Default deny all ingress/egress unless explicitly allowed

---

## 9. Summary Checklist

| Item | Status |
|---|---|
| Namespaces defined | ✅ |
| Deployments for stateless components | ✅ |
| StatefulSets for DB and Queue | ✅ |
| Services with correct types | ✅ |
| Resource requests and limits set | ✅ |
| ConfigMaps for each service | ✅ |
| Secrets defined with rotation plan | ✅ |
| RBAC roles and bindings | ✅ |
| Inter-service communication mapped | ✅ |
| Network policies planned | ✅ |
| HPA for Task Agent | ✅ |
| TLS via cert-manager | ✅ |
