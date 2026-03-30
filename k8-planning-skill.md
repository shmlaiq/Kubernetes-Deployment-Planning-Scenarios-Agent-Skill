# K8 Planning Skill — Kubernetes Deployment Plan Generator

**Author:** Muhammad Faisal Laiq Siddiqui  
**Version:** 1.0  
**Date:** 2026-03-30

---

## What Is This Skill?

The **K8 Planning Skill** is a reusable framework for generating professional Kubernetes deployment plans for any application — without writing a single line of code. It produces structured, decision-ready documentation covering all critical K8s planning areas.

Use this skill whenever you need to plan a Kubernetes deployment for any project, from a simple web app to a complex AI agent system.

---

## Who Should Use This Skill?

- Developers preparing to deploy applications to Kubernetes
- DevOps/Platform engineers creating infrastructure documentation
- Architects designing cloud-native systems
- Students completing K8s planning assignments
- Teams doing pre-deployment planning reviews

---

## Input: What You Provide

To generate a K8s deployment plan, provide the following information about your application:

### Required Inputs

| Input | Description | Example |
|---|---|---|
| **Application Name** | Name of the system | `AI Task Manager` |
| **Component List** | All services/parts of the app | `UI, API, Agent, Database` |
| **Component Types** | Stateless or stateful? | `UI=stateless, DB=stateful` |
| **Traffic Pattern** | How users access the system | `Public web app`, `Internal tool`, `API-only` |
| **Security Level** | How sensitive is the data | `Low`, `Medium`, `High`, `Critical` |

### Optional Inputs (for richer plans)

| Input | Description |
|---|---|
| **Expected load** | Number of users or requests/second |
| **External integrations** | Third-party APIs, OAuth providers |
| **Compliance requirements** | GDPR, HIPAA, SOC2, etc. |
| **Cloud provider** | AWS, GCP, Azure, on-prem |
| **Preferred service mesh** | Istio, Linkerd, none |

---

## Output: What This Skill Produces

For each application, this skill generates a complete markdown document covering **8 planning sections:**

```
1. Namespaces
2. Pods and Deployments / StatefulSets
3. Services and Their Types
4. Resource Requests and Limits
5. ConfigMaps
6. Secrets Management and Expiry Handling
7. RBAC Roles and RoleBindings
8. Inter-Service Communication
```

---

## The Planning Framework

### Section 1: Namespaces

**Decision Rules:**

| Condition | Recommendation |
|---|---|
| Production system | Always create separate `prod`, `staging`, `monitoring` namespaces |
| Security-sensitive components | Isolate in dedicated namespace (e.g., `auth`, `security`) |
| Shared infrastructure | Separate `infra` namespace for databases, queues |
| Compliance requirement | Each regulated component in its own namespace |

**Template:**
```
<app-name>-prod          → Production workloads
<app-name>-staging       → Pre-release testing
<app-name>-monitoring    → Observability (Prometheus, Grafana)
<app-name>-infra         → Databases, queues, caches
<app-name>-security      → Auth, secrets (if security-critical)
```

---

### Section 2: Pods and Deployments / StatefulSets

**The Core Decision:**

```
Does the component store data that must survive pod restarts?
  │
  ├─ YES → StatefulSet
  │         Examples: PostgreSQL, MySQL, Redis, MongoDB,
  │                   Elasticsearch, Kafka, RabbitMQ,
  │                   Vector DBs (Qdrant, Weaviate)
  │
  └─ NO  → Deployment
            Examples: Web servers, APIs, AI agents,
                      Notification services, Auth services,
                      Any stateless microservice
```

**Deployment Planning Checklist:**

For each component, decide:

| Question | Options |
|---|---|
| How many replicas? | 1 (dev), 2 (HA minimum), 3+ (production) |
| Update strategy? | RollingUpdate (zero-downtime) or Recreate (full restart) |
| Need autoscaling? | Add HPA with CPU/memory/custom metrics |
| Any init containers? | DB migrations, config validation, wait-for-dependency |
| Health checks? | Always add liveness + readiness probes |

**Probe Templates:**

| Probe Type | Purpose | When to Use |
|---|---|---|
| Liveness | Is the container alive? Restart if dead | All containers |
| Readiness | Is the container ready for traffic? | All containers |
| Startup | Give slow-starting containers more time | Java apps, ML models |

---

### Section 3: Services and Their Types

**Service Type Decision Tree:**

```
Who needs to access this component?
  │
  ├─ External users (internet) → LoadBalancer (production) or NodePort (dev)
  │
  ├─ Other pods inside cluster → ClusterIP (default, most common)
  │
  ├─ StatefulSet pods (need stable DNS) → Headless Service (ClusterIP: None)
  │
  └─ Specific node port needed → NodePort (30000-32767 range)
```

**Service Planning Table Template:**

| Service Name | Target | Type | Port | Justification |
|---|---|---|---|---|
| `<component>-service` | `<component>` pods | `<type>` | `<port>` | `<why>` |

**Common Patterns:**

| Scenario | Service Type |
|---|---|
| Public-facing web UI | LoadBalancer |
| REST API (internal use) | ClusterIP |
| REST API (dev/staging external) | NodePort |
| Database StatefulSet | Headless ClusterIP |
| Message queue StatefulSet | Headless ClusterIP |

---

### Section 4: Resource Requests and Limits

**The Golden Rule:**
> Always set both `requests` (guaranteed minimum) and `limits` (hard maximum).
> Never set limits without requests. Never leave both unset.

**Sizing Guidelines by Component Type:**

| Component Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| Lightweight UI (React/Next.js) | 100m | 500m | 128Mi | 512Mi |
| REST API (Node/Python/Go) | 250m | 1000m | 256Mi | 1Gi |
| AI/LLM Agent | 500m | 2000m | 512Mi | 2Gi |
| Background Worker | 250m | 1000m | 256Mi | 1Gi |
| Database (per pod) | 500m | 2000m | 1Gi | 4Gi |
| Cache (Redis per pod) | 250m | 1000m | 512Mi | 2Gi |
| Message Queue (per pod) | 250m | 1000m | 512Mi | 2Gi |
| Vector DB (per pod) | 1000m | 4000m | 2Gi | 8Gi |

> Adjust based on profiling. These are safe starting points.

**HPA Template:**

| Setting | Value |
|---|---|
| Min replicas | 2 |
| Max replicas | 10 (adjust per scale needs) |
| CPU threshold | 70% |
| Custom metric | Queue depth, active sessions, etc. |

---

### Section 5: ConfigMaps

**What Goes in ConfigMaps:**
- URLs and hostnames of other services
- Feature flags
- Timeout values
- Log levels
- Non-sensitive environment variables
- Application settings

**What Does NOT Go in ConfigMaps:**
- Passwords, API keys, tokens → Use Secrets
- TLS certificates → Use Secrets
- Per-user data → Use a database

**ConfigMap Per Service Pattern:**

Each service gets its own ConfigMap named `<service-name>-config`.

**Typical ConfigMap Keys:**

```
# Service discovery
<DEPENDENCY>_SERVICE_URL=http://<service-name>:<port>

# App configuration
LOG_LEVEL=info
MAX_WORKERS=4
TIMEOUT_SECONDS=30

# Feature flags
FEATURE_<NAME>=true|false
```

---

### Section 6: Secrets Management

**Secrets Classification:**

| Secret Type | Examples | Rotation Frequency |
|---|---|---|
| Database credentials | DB_USER, DB_PASSWORD | Every 30 days |
| API keys | ANTHROPIC_API_KEY, STRIPE_KEY | Every 90 days |
| JWT signing keys | JWT_SECRET, PRIVATE_KEY | Every 7-30 days |
| OAuth client secrets | CLIENT_ID, CLIENT_SECRET | Every 90 days |
| User OAuth tokens | access_token, refresh_token | Auto-refresh before expiry |
| TLS certificates | cert.pem, key.pem | 30 days before expiry |

**Secrets Architecture:**

```
HashiCorp Vault / AWS Secrets Manager / GCP Secret Manager
              │
              │ (External Secrets Operator syncs)
              ▼
    Kubernetes Secrets (encrypted at rest)
              │
              │ (mounted as env vars or volume)
              ▼
         Pod containers
```

**Security Best Practices Checklist:**

| Practice | Description |
|---|---|
| ✅ Encrypt at rest | Enable EncryptionConfiguration in K8s API server |
| ✅ External secret store | Never manage secrets manually — use Vault or cloud SM |
| ✅ Least privilege | Each SA only accesses its own secrets |
| ✅ Never in images | No secrets in Dockerfiles or env files committed to git |
| ✅ Never in logs | Mask secrets in all application log output |
| ✅ Immutable secrets | Mark production secrets as `immutable: true` |
| ✅ Audit secret access | Enable audit logging for Secret reads in K8s |
| ✅ Auto-rotation | Automate rotation — never rotate manually in production |

---

### Section 7: RBAC

**RBAC Planning Process:**

**Step 1:** Create a ServiceAccount per component
**Step 2:** Define minimal Role with only needed permissions
**Step 3:** Bind Role to ServiceAccount via RoleBinding
**Step 4:** Audit and remove any unused permissions

**Minimal Permissions by Component Type:**

| Component | ConfigMaps | Secrets | Pods | Jobs | Services |
|---|---|---|---|---|---|
| Frontend/UI | get, list | none | none | none | get, list |
| Backend API | get, list | get | get, list | none | get, list |
| AI Agent/Worker | get, list | get | get, list, watch | create, delete | get |
| Auth Service | get | get, update | get | none | get |
| Background Job | get | get | none | create, get | none |
| Monitoring | get, list | none | get, list, watch | none | get, list |

**RBAC Template:**

```yaml
# Conceptual structure (no code — planning only)

ServiceAccount: sa-<component>
  namespace: <app-namespace>

Role: <component>-role
  namespace: <app-namespace>
  rules:
    - resource: ConfigMaps → verbs: get, list
    - resource: Secrets → verbs: get (only specific secrets)

RoleBinding: <component>-rolebinding
  role: <component>-role
  subject: sa-<component>
```

**When to use ClusterRole vs Role:**

| Use Case | Type |
|---|---|
| Permissions scoped to one namespace | Role |
| Monitoring that spans all namespaces | ClusterRole |
| Admin access across cluster | ClusterRole |
| Normal microservice | Role (always prefer this) |

---

### Section 8: Inter-Service Communication

**Communication Pattern Decision:**

| Pattern | When to Use | Tool |
|---|---|---|
| Synchronous HTTP/REST | Request needs immediate response | ClusterIP Service + DNS |
| Asynchronous messaging | Fire-and-forget, task queues | RabbitMQ, Redis Streams, Kafka |
| Event streaming | Real-time events, fan-out | Kafka, NATS |
| gRPC | High performance, typed contracts | ClusterIP Service |

**Internal DNS Format:**
```
<service-name>.<namespace>.svc.cluster.local:<port>
```

**Communication Security Levels:**

| Level | Requirement | Implementation |
|---|---|---|
| Basic | Service-to-service HTTP | ClusterIP + Network Policies |
| Standard | Authenticated API calls | JWT validation at each service |
| High | Encrypted + authenticated | Service mesh with mTLS (Istio/Linkerd) |
| Critical | Zero-trust | mTLS + OPA/Gatekeeper policies |

**Network Policy Template:**

```
Default policy: DENY ALL ingress and egress
Then explicitly ALLOW:
  - <service-A> → <service-B> on port <X>
  - <service-B> → <service-C> on port <Y>
  - <service-X> egress to internet on port 443 (HTTPS only)
```

---

## How to Use This Skill: Step-by-Step

**Step 1:** Gather application inputs (component list, types, traffic, security level)

**Step 2:** Go through each of the 8 sections in order

**Step 3:** For each section, use the decision trees and tables to make choices

**Step 4:** Fill in the templates with your application's specific values

**Step 5:** Review the Summary Checklist (see below)

**Step 6:** Share the completed plan for team review before any deployment

---

## Summary Checklist (Use for Every Plan)

| # | Item | Check |
|---|---|---|
| 1 | Namespaces defined (prod, staging, monitoring minimum) | ☐ |
| 2 | Every stateless component has a Deployment | ☐ |
| 3 | Every stateful component has a StatefulSet + PVC | ☐ |
| 4 | Every component has a Service with correct type | ☐ |
| 5 | Resource requests AND limits set for all containers | ☐ |
| 6 | HPA configured for components that need autoscaling | ☐ |
| 7 | Liveness and readiness probes on all containers | ☐ |
| 8 | ConfigMap created per service (non-sensitive config only) | ☐ |
| 9 | All secrets listed with rotation schedule | ☐ |
| 10 | External secrets operator or Vault integration planned | ☐ |
| 11 | ServiceAccount created per component | ☐ |
| 12 | Role with minimal permissions per component | ☐ |
| 13 | RoleBinding linking SA to Role | ☐ |
| 14 | Communication flow diagram drawn | ☐ |
| 15 | Network policies: default deny + explicit allow | ☐ |
| 16 | TLS planned for all external-facing services | ☐ |
| 17 | No secrets committed to git or baked into images | ☐ |
| 18 | Audit logging planned (especially for AI agents) | ☐ |

---

## Quick Reference: K8s Object Selection Guide

| You Need | K8s Object |
|---|---|
| Run a stateless app | `Deployment` |
| Run a stateful app with stable identity | `StatefulSet` |
| Run a one-time job | `Job` |
| Run a scheduled job | `CronJob` |
| Expose pods inside cluster | `Service (ClusterIP)` |
| Expose pods to internet | `Service (LoadBalancer)` |
| Store non-sensitive config | `ConfigMap` |
| Store sensitive credentials | `Secret` |
| Persist data across pod restarts | `PersistentVolumeClaim` |
| Control who can do what | `Role` + `RoleBinding` |
| Scale pods automatically | `HorizontalPodAutoscaler` |
| Route external HTTP/HTTPS traffic | `Ingress` |
| Isolate workloads | `Namespace` |
| Control network traffic between pods | `NetworkPolicy` |
| Give a pod K8s API permissions | `ServiceAccount` |

---

## Example: Applying This Skill to a New Project

**Input:** "I need to deploy a simple e-commerce app with a React frontend, Node.js API, and PostgreSQL database."

**Skill Output Summary:**

| Section | Decision |
|---|---|
| Namespaces | `ecommerce-prod`, `ecommerce-staging`, `ecommerce-monitoring` |
| React frontend | Deployment, 2 replicas, RollingUpdate |
| Node.js API | Deployment, 3 replicas, HPA on CPU > 70% |
| PostgreSQL | StatefulSet, 3 replicas, 50Gi PVC each |
| Frontend service | LoadBalancer (public) |
| API service | ClusterIP (internal) |
| DB service | Headless ClusterIP |
| API ConfigMap | `API_DB_URL`, `LOG_LEVEL`, `PORT` |
| API Secrets | `DB_PASSWORD`, `JWT_SECRET`, `STRIPE_KEY` |
| API RBAC | Role: get ConfigMaps, get Secrets |
| Communication | Frontend → API (ClusterIP), API → DB (Headless DNS) |
| Network Policy | Default deny, allow frontend→API:3001, API→DB:5432 |

---

*This skill was created as part of a Kubernetes Deployment Planning project. It can be extended with additional sections for Ingress controllers, service meshes, GitOps (ArgoCD/Flux), and multi-cluster planning.*
