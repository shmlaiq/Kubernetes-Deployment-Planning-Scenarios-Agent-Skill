# Kubernetes Deployment Plan — AI Employee (OpenClaw)

**Author:** Muhammad Faisal Laiq Siddiqui  
**Version:** 1.0  
**Date:** 2026-03-30

---

## Overview

**OpenClaw** is a Personal AI Employee system — an autonomous agent that performs work on behalf of a user or organization. It handles tasks like reading emails, browsing the web, managing files, executing code, interacting with external APIs, and coordinating with other agents.

### Core Components

| Component | Role |
|---|---|
| **Agent Orchestrator** | Brain — plans, delegates, and tracks tasks |
| **Tool Executor** | Runs tools: browser, code runner, file manager, API caller |
| **Memory Service** | Stores short-term (session) and long-term (vector DB) memory |
| **MCP Server** | Model Context Protocol server — exposes tools to the AI |
| **Auth & Identity Service** | Handles user auth, OAuth tokens, and session management |
| **Audit Logger** | Records all agent actions for compliance and debugging |
| **Admin UI** | Web dashboard for monitoring and controlling the AI employee |

---

## 1. Namespaces

Security-first namespace design — each concern is isolated.

| Namespace | Purpose |
|---|---|
| `openclaw-prod` | All production agent workloads |
| `openclaw-staging` | Pre-release testing |
| `openclaw-security` | Auth service, secrets management |
| `openclaw-monitoring` | Audit logs, metrics, tracing |
| `openclaw-infra` | Databases, vector store, message queue |

**Namespace-level isolation prevents:**
- Staging agents from affecting production data
- Security components from being accessed by general workloads
- Audit logs from being tampered with by agent pods

---

## 2. Pods and Deployments / StatefulSets

### 2.1 Agent Orchestrator — Deployment

- **Type:** `Deployment` (stateless — state stored externally in Memory Service)
- **Replicas:** 2 (one per active user session, scale via HPA)
- **Pod Labels:** `app: agent-orchestrator`, `tier: ai`, `security-level: high`
- **Strategy:** RollingUpdate

| Setting | Value |
|---|---|
| Container Image | `openclaw/orchestrator:latest` |
| Port | 8000 |
| Liveness Probe | HTTP GET `/health` every 30s |
| Readiness Probe | HTTP GET `/ready` every 10s |

**HPA Configuration:**
- Min replicas: 2, Max replicas: 20
- Scale trigger: Active sessions > 5 per pod OR CPU > 60%

---

### 2.2 Tool Executor — Deployment

- **Type:** `Deployment` (stateless — executes tools on demand)
- **Replicas:** 3
- **Pod Labels:** `app: tool-executor`, `tier: ai`, `security-level: critical`
- **Security Context:** Run as non-root user. Read-only root filesystem. Drop all Linux capabilities.
- **Strategy:** RollingUpdate

> ⚠️ **Critical Security Note:** Tool Executor runs untrusted code (browser automation, shell commands). It must run in a **sandboxed environment** (gVisor or Kata Containers runtime) and have **strict network egress policies**.

| Setting | Value |
|---|---|
| Container Image | `openclaw/tool-executor:latest` |
| Port | 8001 |
| Runtime Class | `gvisor` (sandboxed container runtime) |
| Liveness Probe | HTTP GET `/tools/health` every 30s |

---

### 2.3 Memory Service — Deployment + StatefulSet

Two parts:

**a) Memory API — Deployment** (stateless API layer)
- Replicas: 2
- Handles read/write requests to vector DB and cache
- Port: 8002

**b) Vector Database — StatefulSet** (e.g., Qdrant or Weaviate)
- Replicas: 3 (1 primary + 2 replicas)
- Persistent storage: 100Gi per pod
- Pod Labels: `app: vector-db`, `tier: storage`

**c) Redis (Short-term Memory) — StatefulSet**
- Replicas: 3 (Redis Cluster mode)
- Persistent storage: 10Gi per pod
- Used for session context, recent conversation history

---

### 2.4 MCP Server — Deployment

- **Type:** `Deployment`
- **Replicas:** 2
- **Pod Labels:** `app: mcp-server`, `tier: ai`
- Exposes tools (web search, file read, API calls) to the Agent Orchestrator via MCP protocol

| Setting | Value |
|---|---|
| Container Image | `openclaw/mcp-server:latest` |
| Port | 8003 |
| Liveness Probe | HTTP GET `/mcp/health` every 30s |

---

### 2.5 Auth & Identity Service — Deployment

- **Type:** `Deployment`
- **Replicas:** 3 (HA — auth must never go down)
- **Pod Labels:** `app: auth-service`, `tier: security`
- **Namespace:** `openclaw-security`

Handles:
- User login (OAuth2 / OIDC)
- JWT issuance and validation
- OAuth token management for external services (Gmail, Calendar, Slack, etc.)
- Session management

| Setting | Value |
|---|---|
| Container Image | `openclaw/auth:latest` |
| Port | 8004 |
| Liveness Probe | HTTP GET `/auth/health` every 15s |

---

### 2.6 Audit Logger — Deployment + StatefulSet

**a) Audit API — Deployment**
- Replicas: 2
- Receives audit events from all services
- Port: 8005

**b) Audit Database — StatefulSet** (append-only PostgreSQL or ClickHouse)
- Replicas: 3
- Persistent storage: 200Gi per pod
- Write-once, read-many (WORM policy for compliance)

> ⚠️ Agent actions must be logged with: timestamp, user ID, action type, tool used, input params, output summary, success/failure.

---

### 2.7 Admin UI — Deployment

- **Type:** `Deployment`
- **Replicas:** 2
- **Pod Labels:** `app: admin-ui`, `tier: frontend`
- Dashboard for admins to view agent activity, pause agents, review audit logs

| Setting | Value |
|---|---|
| Container Image | `openclaw/admin-ui:latest` |
| Port | 3000 |

---

## 3. Services and Their Types

| Service Name | Target | Type | Port | Purpose |
|---|---|---|---|---|
| `admin-ui-service` | Admin UI pods | **LoadBalancer** | 443 | Admin dashboard — HTTPS only |
| `orchestrator-service` | Agent Orchestrator | **ClusterIP** | 8000 | Internal — called by UI and MCP |
| `tool-executor-service` | Tool Executor | **ClusterIP** | 8001 | Internal — called by Orchestrator only |
| `memory-service` | Memory API | **ClusterIP** | 8002 | Internal — called by Orchestrator |
| `mcp-service` | MCP Server | **ClusterIP** | 8003 | Internal — called by Orchestrator |
| `auth-service` | Auth & Identity | **ClusterIP** | 8004 | Internal — all services validate tokens here |
| `audit-service` | Audit Logger | **ClusterIP** | 8005 | Internal — all services push audit events |
| `vector-db-service` | Vector DB StatefulSet | **ClusterIP (Headless)** | 6333 | Stable DNS for StatefulSet pods |
| `redis-service` | Redis StatefulSet | **ClusterIP (Headless)** | 6379 | Stable DNS for Redis cluster |
| `audit-db-service` | Audit DB StatefulSet | **ClusterIP (Headless)** | 5432 | Stable DNS for audit DB |

**Security Note:** No `NodePort` services in production. Admin UI is accessible only via LoadBalancer with IP allowlisting. No service is publicly exposed except the Admin UI behind authentication.

---

## 4. Resource Requests and Limits

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| Agent Orchestrator | 500m | 2000m | 512Mi | 2Gi |
| Tool Executor | 1000m | 4000m | 1Gi | 4Gi |
| Memory API | 250m | 1000m | 256Mi | 1Gi |
| MCP Server | 250m | 1000m | 256Mi | 1Gi |
| Auth Service | 250m | 1000m | 256Mi | 1Gi |
| Audit Logger API | 100m | 500m | 128Mi | 512Mi |
| Admin UI | 100m | 500m | 128Mi | 512Mi |
| Vector DB (per pod) | 1000m | 4000m | 2Gi | 8Gi |
| Redis (per pod) | 250m | 1000m | 512Mi | 2Gi |
| Audit DB (per pod) | 500m | 2000m | 1Gi | 4Gi |

> Tool Executor gets highest limits — browser automation and code execution are resource-intensive.

---

## 5. ConfigMaps

### 5.1 `orchestrator-config`
```
ANTHROPIC_MODEL=claude-sonnet-4-20250514
MAX_TOOL_CALLS_PER_TASK=50
TASK_TIMEOUT_SECONDS=600
MCP_SERVER_URL=http://mcp-service:8003
MEMORY_SERVICE_URL=http://memory-service:8002
AUTH_SERVICE_URL=http://auth-service:8004
AUDIT_SERVICE_URL=http://audit-service:8005
LOG_LEVEL=info
```

### 5.2 `tool-executor-config`
```
ALLOWED_DOMAINS=*.google.com,*.github.com,*.anthropic.com
BROWSER_HEADLESS=true
CODE_EXECUTION_TIMEOUT=30
MAX_FILE_SIZE_MB=50
SANDBOX_MODE=gvisor
LOG_LEVEL=info
```

### 5.3 `memory-config`
```
VECTOR_DB_URL=http://vector-db-service:6333
REDIS_URL=redis://redis-service:6379
SESSION_TTL_SECONDS=3600
LONG_TERM_MEMORY_COLLECTION=openclaw_memory
EMBEDDING_MODEL=text-embedding-3-small
```

### 5.4 `auth-config`
```
JWT_ALGORITHM=RS256
JWT_EXPIRY_SECONDS=3600
REFRESH_TOKEN_EXPIRY_DAYS=30
OAUTH_CALLBACK_URL=https://admin.openclaw.example.com/auth/callback
ALLOWED_OAUTH_PROVIDERS=google,github,microsoft
SESSION_STORE=redis
```

### 5.5 `audit-config`
```
AUDIT_DB_HOST=audit-db-service
AUDIT_DB_PORT=5432
AUDIT_DB_NAME=openclaw_audit
RETENTION_DAYS=365
IMMUTABLE_RECORDS=true
LOG_LEVEL=warn
```

---

## 6. Secrets Management and Expiry Handling

### 6.1 Secrets List

| Secret Name | Contains | Used By | Namespace |
|---|---|---|---|
| `anthropic-api-key` | `ANTHROPIC_API_KEY` | Orchestrator | openclaw-prod |
| `jwt-signing-keys` | `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY` | Auth Service | openclaw-security |
| `db-credentials` | `DB_USER`, `DB_PASSWORD` | Memory API, Audit Logger | openclaw-prod |
| `redis-credentials` | `REDIS_PASSWORD` | Memory Service | openclaw-prod |
| `vector-db-credentials` | `QDRANT_API_KEY` | Memory API | openclaw-prod |
| `oauth-client-secrets` | `GOOGLE_CLIENT_SECRET`, `GITHUB_CLIENT_SECRET` | Auth Service | openclaw-security |
| `user-oauth-tokens` | Per-user OAuth access/refresh tokens | Orchestrator | openclaw-prod |
| `tls-cert` | TLS certificate and private key | Ingress | openclaw-prod |
| `audit-db-credentials` | `AUDIT_DB_USER`, `AUDIT_DB_PASSWORD` | Audit Logger | openclaw-monitoring |

---

### 6.2 Security Considerations for Secrets

| Concern | Strategy |
|---|---|
| **Encryption at rest** | Enable EncryptionConfiguration in K8s API server (AES-256) |
| **External secret store** | Use HashiCorp Vault or AWS Secrets Manager via External Secrets Operator |
| **JWT key rotation** | RS256 key pair rotated every 7 days. Old public key kept for 24h to validate existing tokens |
| **API key rotation** | Anthropic and OAuth keys rotated every 90 days via automation |
| **User OAuth token refresh** | Refresh tokens auto-renewed before expiry using background job |
| **TLS certificates** | cert-manager + Let's Encrypt, auto-renewed 30 days before expiry |
| **Secrets never in logs** | All services must mask secrets in log output |
| **Minimal secret access** | Each service account can only access its own secrets (RBAC enforced) |
| **Secret audit trail** | All secret reads logged in Vault audit log |

---

### 6.3 User OAuth Token Lifecycle

```
User grants permission
        │
        ▼
Auth Service stores access_token + refresh_token in Secret (per user)
        │
        ▼
Access token used by Orchestrator for API calls
        │
        ▼ (when token expires or 15 min before expiry)
Auth Service refreshes token automatically
        │
        ▼
New access_token stored back in Secret
        │
        ▼ (if refresh_token expires — typically 30-90 days)
User must re-authenticate via Admin UI
```

---

## 7. RBAC — Roles and RoleBindings

### 7.1 Service Accounts

| Service Account | Namespace | Used By |
|---|---|---|
| `sa-orchestrator` | openclaw-prod | Agent Orchestrator |
| `sa-tool-executor` | openclaw-prod | Tool Executor |
| `sa-memory` | openclaw-prod | Memory Service |
| `sa-mcp` | openclaw-prod | MCP Server |
| `sa-auth` | openclaw-security | Auth & Identity Service |
| `sa-audit` | openclaw-monitoring | Audit Logger |
| `sa-admin-ui` | openclaw-prod | Admin UI |

---

### 7.2 Role: `orchestrator-role`

| Resource | Verbs |
|---|---|
| ConfigMaps | get, list |
| Secrets (`anthropic-api-key`, `user-oauth-tokens`) | get |
| Pods | get, list, watch |
| Jobs | create, get, list, delete |

**RoleBinding:** `orchestrator-role` → `sa-orchestrator` in `openclaw-prod`

---

### 7.3 Role: `tool-executor-role`

| Resource | Verbs |
|---|---|
| ConfigMaps | get |
| Secrets | **none** (tool executor must NOT have secret access — principle of least privilege) |
| Pods | get |

**RoleBinding:** `tool-executor-role` → `sa-tool-executor` in `openclaw-prod`

> ⚠️ Tool Executor is sandboxed and intentionally denied secret access. It receives only what Orchestrator explicitly passes.

---

### 7.4 Role: `auth-role`

| Resource | Verbs |
|---|---|
| Secrets (`jwt-signing-keys`, `oauth-client-secrets`, `user-oauth-tokens`) | get, update |
| ConfigMaps | get |

**RoleBinding:** `auth-role` → `sa-auth` in `openclaw-security`

---

### 7.5 Role: `audit-role`

| Resource | Verbs |
|---|---|
| Secrets (`audit-db-credentials`) | get |
| ConfigMaps | get |
| Events | create, list |

**RoleBinding:** `audit-role` → `sa-audit` in `openclaw-monitoring`

---

### 7.6 ClusterRole: `admin-ui-reader`

Admin UI needs read access across namespaces for monitoring:

| Resource | Verbs |
|---|---|
| Pods, Deployments, StatefulSets | get, list, watch |
| Services, Endpoints | get, list |
| Events | get, list |
| Secrets | **none** |

**ClusterRoleBinding:** `admin-ui-reader` → `sa-admin-ui`

---

## 8. Inter-Service Communication

### 8.1 Communication Flow

```
Admin (Browser)
      │ HTTPS (LoadBalancer + IP allowlist)
      ▼
[Admin UI :3000]
      │ HTTP (ClusterIP)
      ▼
[Auth Service :8004] ◀──── Token validation ────── All services
      │
      ▼
[Agent Orchestrator :8000]
      │
      ├──▶ [MCP Server :8003] ──▶ External APIs (Google, GitHub, etc.)
      │
      ├──▶ [Tool Executor :8001] (sandboxed) ──▶ Browser / Code Sandbox
      │
      ├──▶ [Memory Service :8002]
      │          ├──▶ [Vector DB :6333]
      │          └──▶ [Redis :6379]
      │
      └──▶ [Audit Logger :8005]
                 └──▶ [Audit DB :5432]
```

### 8.2 Communication Protocols

| From | To | Protocol | Auth Method |
|---|---|---|---|
| Admin Browser | Admin UI | HTTPS | OAuth2 via Auth Service |
| Admin UI | Orchestrator | HTTP/REST | JWT Bearer token |
| Orchestrator | Auth Service | HTTP/REST | Internal service account token |
| Orchestrator | MCP Server | HTTP/MCP | JWT |
| Orchestrator | Tool Executor | HTTP/REST | JWT |
| Orchestrator | Memory Service | HTTP/REST | JWT |
| Orchestrator | Audit Logger | HTTP/REST | JWT (fire-and-forget) |
| Memory Service | Vector DB | HTTP/REST | API key (from Secret) |
| Memory Service | Redis | TCP | Password (from Secret) |
| MCP Server | External APIs | HTTPS | OAuth tokens (from Auth Service) |
| Tool Executor | Browser/Web | HTTPS (egress) | Sandboxed — no credentials |

### 8.3 Network Policies — Security-First

| Rule | Description |
|---|---|
| Default deny all | All ingress and egress blocked by default in every namespace |
| Admin UI → Orchestrator | Allow only port 8000 |
| Orchestrator → Tool Executor | Allow only port 8001 |
| Orchestrator → Memory Service | Allow only port 8002 |
| Orchestrator → MCP Server | Allow only port 8003 |
| All services → Auth Service | Allow port 8004 for token validation |
| All services → Audit Logger | Allow port 8005 |
| Tool Executor egress | Allow HTTPS (443) to internet only via egress proxy |
| MCP Server egress | Allow HTTPS (443) to specific whitelisted domains only |
| No cross-namespace access | `openclaw-prod` pods cannot reach `openclaw-security` pods directly |

### 8.4 Service Mesh (Recommended)

Deploy **Istio** or **Linkerd** for:
- Mutual TLS (mTLS) between all services — encrypts all internal traffic
- Traffic observability — traces every request
- Circuit breaking — prevents cascade failures
- Fine-grained traffic policies

---

## 9. Security Considerations Summary

| Area | Measure |
|---|---|
| **Container Security** | Non-root user, read-only root filesystem, drop all capabilities |
| **Tool Executor Sandbox** | gVisor runtime class — kernel-level isolation |
| **Secrets** | External Secrets Operator + HashiCorp Vault |
| **Network** | Default deny + explicit allow Network Policies |
| **Service Mesh** | mTLS between all pods via Istio/Linkerd |
| **Auth** | JWT RS256 with short expiry + refresh token rotation |
| **Audit** | Immutable audit log — all agent actions recorded |
| **RBAC** | Least privilege — each SA has minimal permissions |
| **Image Security** | Scan images with Trivy in CI/CD. No `latest` tag in prod |
| **Pod Security** | PodSecurityAdmission: `restricted` policy in prod namespace |
| **Egress Control** | All outbound traffic via egress proxy with domain allowlist |

---

## 10. Summary Checklist

| Item | Status |
|---|---|
| Namespaces with security isolation | ✅ |
| Deployments for all stateless services | ✅ |
| StatefulSets for Vector DB, Redis, Audit DB | ✅ |
| Services with correct types (no unnecessary public exposure) | ✅ |
| Resource requests and limits | ✅ |
| ConfigMaps per service | ✅ |
| Secrets with rotation and expiry strategy | ✅ |
| User OAuth token lifecycle | ✅ |
| RBAC with least privilege | ✅ |
| Tool Executor sandboxed (gVisor) | ✅ |
| Network policies (default deny) | ✅ |
| Service mesh (mTLS) recommended | ✅ |
| HPA for Orchestrator | ✅ |
| Audit logging for all agent actions | ✅ |
| TLS via cert-manager | ✅ |
