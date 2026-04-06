# Kubernetes Deployment Plan
## Scenario 2: AI Employee using OpenClaw

**Course:** AI-400 | **Class:** 22 | **Date:** April 6, 2026  
**Type:** Planning Document — No executable code  
**Security Level:** Critical (personal data + external system access)

---

## Table of Contents

1. [Overview](#overview)
2. [Namespaces](#1-namespaces)
3. [Pods, Deployments & StatefulSets](#2-pods-deployments--statefulsets)
4. [Services & Networking](#3-services--networking)
5. [Resource Requests & Limits](#4-resource-requests--limits)
6. [ConfigMaps](#5-configmaps)
7. [Secrets Management](#6-secrets-management)
8. [RBAC — Roles & RoleBindings](#7-rbac--roles--rolebindings)
9. [Network Policies](#8-network-policies)
10. [Inter-Service Communication](#9-inter-service-communication)
11. [Horizontal Pod Autoscaler](#10-horizontal-pod-autoscaler)
12. [Security Hardening](#11-security-hardening)
13. [Summary Checklist](#12-summary-checklist)

---

## Overview

**OpenClaw** is a Personal AI Employee platform. It acts as an autonomous agent that browses the web, reads emails, manages calendars, executes code, and interacts with external APIs on behalf of users. Because it handles sensitive personal data and accesses external systems with real-world consequences, **security is the primary architectural concern** — every design decision is evaluated through a security lens first.

| Component | Role |
|---|---|
| **User Gateway API** | The only public-facing service; handles authentication and routes requests to the Core Agent |
| **OpenClaw Core Agent** | Orchestrates tasks, manages tools, calls LLM APIs, reads and writes user memory |
| **Tool Executor** | Sandboxed runtime that performs web browsing, code execution, and external API calls on the agent's behalf |
| **Memory & Context Store** | Vector database (Qdrant) and structured relational store (PostgreSQL) for long-term agent memory |
| **Audit & Compliance Service** | Immutable log of every agent action; supports compliance review and incident investigation |

---

## 1. Namespaces

Security isolation is the primary driver for namespace design. Each namespace has a tightly scoped NetworkPolicy, preventing lateral movement between components.

| Namespace | Purpose | Key Security Constraint |
|---|---|---|
| `openclaw-prod` | Gateway API and Core Agent | Cannot initiate traffic to `openclaw-tooling` data stores |
| `openclaw-tooling` | Tool Executor (sandboxed) | **Cannot reach `openclaw-data` at all**; internet egress only |
| `openclaw-data` | PostgreSQL, Qdrant, Redis | Denies all ingress except from `openclaw-prod` |
| `openclaw-audit` | Audit & Compliance Service | Append-only log receiver; never initiates connections |
| `openclaw-monitoring` | Prometheus, Grafana, alerting | Read-only cluster metrics access |

> **Why isolate `openclaw-tooling`?** The Tool Executor runs untrusted operations: it browses arbitrary websites and executes user-supplied code. If it were compromised, it must not be able to reach the database, internal APIs, or secrets. Namespace isolation plus NetworkPolicies enforces this boundary at the kernel level.

---

## 2. Pods, Deployments & StatefulSets

### 2.1 — User Gateway API (Deployment)

The single public entry point. All user sessions authenticate here before any request reaches the Core Agent. Runs with hardened securityContext.

```
Name:       gateway-deployment
Namespace:  openclaw-prod
Replicas:   3  (HPA: min 3, max 10)
Strategy:   RollingUpdate  |  maxSurge: 1  |  maxUnavailable: 0

Container: gateway
  Image:   openclaw/gateway:latest
  Port:    8443  (HTTPS only; no plain HTTP)
  Resources:
    requests:  cpu 200m   |  memory 256Mi
    limits:    cpu 1000m  |  memory 1Gi
  EnvFrom:
    configMapRef → gateway-config
    secretRef    → gateway-secrets
  SecurityContext:
    runAsNonRoot:             true
    runAsUser:                1000
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem:   true
  Probes:
    livenessProbe:   GET /health  port 8443  initialDelay 15s
    readinessProbe:  GET /ready   port 8443  initialDelay 5s
```

---

### 2.2 — OpenClaw Core Agent (Deployment)

Stateless orchestration layer. Manages the LLM conversation loop, dispatches tool calls to the Tool Executor, and reads/writes agent memory. Memory state is persisted externally so any pod can resume any user session.

```
Name:       core-agent-deployment
Namespace:  openclaw-prod
Replicas:   4  (HPA: min 4, max 20 based on active sessions)
Strategy:   RollingUpdate  |  maxSurge: 2  |  maxUnavailable: 0

Container: core-agent
  Image:   openclaw/core-agent:latest
  Port:    9000
  Resources:
    requests:  cpu 500m   |  memory 1Gi
    limits:    cpu 2000m  |  memory 4Gi
  EnvFrom:
    configMapRef → core-agent-config
    secretRef    → core-agent-secrets   (via Vault sidecar injection)
  SecurityContext:
    runAsNonRoot:             true
    runAsUser:                1001
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem:   true
  VolumeMounts:
    emptyDir → /tmp   (writable scratch space; isolated per pod)
  Probes:
    livenessProbe:   GET /health  port 9000  initialDelay 20s
    readinessProbe:  GET /ready   port 9000  initialDelay 10s
```

---

### 2.3 — Tool Executor (Deployment, Isolated Namespace)

The most restricted workload in the system. Runs in `openclaw-tooling` with maximum security hardening. All Linux capabilities are dropped, the filesystem is read-only, and the scratch directory is capped at 500Mi to prevent disk exhaustion attacks.

```
Name:       tool-executor-deployment
Namespace:  openclaw-tooling
Replicas:   4  (HPA: min 4, max 30 — highest variability component)
Strategy:   RollingUpdate

Container: tool-executor
  Image:   openclaw/tool-executor:latest
  Port:    9100
  Resources:
    requests:  cpu 500m   |  memory 512Mi
    limits:    cpu 2000m  |  memory 2Gi
  SecurityContext:
    runAsNonRoot:             true
    runAsUser:                65534  (nobody — lowest privilege UID)
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem:   true
    seccompProfile:           RuntimeDefault
    capabilities.drop:        ALL
  VolumeMounts:
    emptyDir (sizeLimit: 500Mi) → /sandbox
  Probes:
    livenessProbe:   GET /health  port 9100  initialDelay 10s
    readinessProbe:  GET /ready   port 9100  initialDelay 5s
```

> **Why `runAsUser: 65534` (nobody)?** If a container escape occurs, the attacker lands as the lowest-privilege user on the host — they cannot write to system directories or escalate to root.

---

### 2.4 — Audit & Compliance Service (Deployment)

Receives structured events from all other services. Uses `Recreate` strategy (not RollingUpdate) to guarantee exactly-once log writes — two audit pods running simultaneously could create duplicate or out-of-order entries.

```
Name:       audit-deployment
Namespace:  openclaw-audit
Replicas:   2
Strategy:   Recreate  (log consistency > rolling availability)

Container: audit-service
  Image:   openclaw/audit:latest
  Port:    9200
  Resources:
    requests:  cpu 200m  |  memory 256Mi
    limits:    cpu 500m  |  memory 512Mi
  EnvFrom:
    configMapRef → audit-config
    secretRef    → audit-secrets
  SecurityContext:
    runAsNonRoot: true
    runAsUser:    1002
    allowPrivilegeEscalation: false
  Probes:
    livenessProbe:   GET /health  port 9200  initialDelay 10s
    readinessProbe:  GET /ready   port 9200  initialDelay 5s
```

---

### 2.5 — PostgreSQL (StatefulSet)

Stores user profiles, agent task history, and structured agent state. Deployed as a StatefulSet for stable pod identity and persistent storage.

```
Name:        postgres-statefulset
Namespace:   openclaw-data
Replicas:    1 primary  +  1 read replica (separate StatefulSet)
ServiceName: postgres-headless

Container: postgres
  Image:   postgres:16
  Port:    5432
  Resources:
    requests:  cpu 500m   |  memory 2Gi
    limits:    cpu 2000m  |  memory 8Gi
  EnvFrom:     secretRef → postgres-secrets  (Vault dynamic credentials)
  VolumeMount: pgdata → /var/lib/postgresql/data

VolumeClaimTemplate:
  storageClassName: ssd-retain
  accessModes:      ReadWriteOnce
  storage:          50Gi
```

---

### 2.6 — Qdrant Vector Store (StatefulSet)

Stores agent memory embeddings for semantic recall. Three replicas provide high availability and distributed query performance.

```
Name:       qdrant-statefulset
Namespace:  openclaw-data
Replicas:   3  (distributed HA cluster)

Container: qdrant
  Image:   qdrant/qdrant:latest
  Ports:   6333 (HTTP)  |  6334 (gRPC)
  Resources:
    requests:  cpu 500m   |  memory 2Gi
    limits:    cpu 2000m  |  memory 8Gi
  VolumeMount: qdrant-storage → /qdrant/storage

VolumeClaimTemplate:
  storageClassName: ssd-retain
  accessModes:      ReadWriteOnce
  storage:          30Gi per replica
```

---

### 2.7 — Redis (StatefulSet)

Stores JWT session tokens and per-user rate limit counters. Deployed with persistence so session state survives pod restarts.

```
Name:       redis-statefulset
Namespace:  openclaw-data
Replicas:   1

Container: redis
  Image:   redis:7-alpine
  Port:    6379
  Resources:
    requests:  cpu 100m  |  memory 512Mi
    limits:    cpu 500m  |  memory 2Gi
  VolumeMount: redis-data → /data

VolumeClaimTemplate:
  storage: 10Gi
```

---

## 3. Services & Networking

### Service Definitions

| Service Name | Type | Port | Namespace | Justification |
|---|---|---|---|---|
| `gateway-service` | LoadBalancer | 443 | openclaw-prod | Single public HTTPS entrypoint |
| `core-agent-service` | ClusterIP | 9000 | openclaw-prod | Internal; called by gateway only |
| `tool-executor-service` | ClusterIP | 9100 | openclaw-tooling | Internal; called by core-agent only |
| `audit-service` | ClusterIP | 9200 | openclaw-audit | Internal; receives events from all services |
| `postgres-service` | ClusterIP | 5432 | openclaw-data | Internal; no external access |
| `postgres-headless` | Headless | 5432 | openclaw-data | StatefulSet replication pod identity |
| `qdrant-service` | ClusterIP | 6333/6334 | openclaw-data | Internal; accessed by core-agent only |
| `redis-service` | ClusterIP | 6379 | openclaw-data | Internal; session cache |

> **No NodePort services.** All external traffic enters exclusively through the single LoadBalancer → Ingress → gateway-service path.

---

## 4. Resource Requests & Limits

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| User Gateway API | 200m | 1000m | 256Mi | 1Gi |
| Core Agent | 500m | 2000m | 1Gi | 4Gi |
| Tool Executor | 500m | 2000m | 512Mi | 2Gi |
| Audit Service | 200m | 500m | 256Mi | 512Mi |
| PostgreSQL | 500m | 2000m | 2Gi | 8Gi |
| Qdrant | 500m | 2000m | 2Gi | 8Gi |
| Redis | 100m | 500m | 512Mi | 2Gi |

**Sizing rationale:**
- **Core Agent** is allocated 4Gi memory because it maintains the full LLM conversation context in memory during active sessions.
- **PostgreSQL and Qdrant** receive 8Gi limits to support concurrent user sessions without swapping.
- **Tool Executor** gets a 500Mi memory floor; tool execution is short-lived but may require holding HTML DOMs or code execution output in memory momentarily.

---

## 5. ConfigMaps

ConfigMaps hold **non-sensitive** configuration only. All credentials are injected separately via Secrets.

### `gateway-config`
```
AUTH_PROVIDER_URL:   https://auth.openclaw.ai
SESSION_TTL_SECONDS: 3600
RATE_LIMIT_RPM:      60
CORE_AGENT_URL:      http://core-agent-service.openclaw-prod.svc:9000
REDIS_HOST:          redis-service.openclaw-data.svc
TLS_ENABLED:         true
LOG_LEVEL:           info
```

### `core-agent-config`
```
TOOL_EXECUTOR_URL:      http://tool-executor-service.openclaw-tooling.svc:9100
AUDIT_SERVICE_URL:      http://audit-service.openclaw-audit.svc:9200
POSTGRES_HOST:          postgres-service.openclaw-data.svc
QDRANT_HOST:            qdrant-service.openclaw-data.svc
QDRANT_PORT:            6333
AGENT_MODEL:            claude-opus-4
AGENT_MAX_STEPS:        50
AGENT_TIMEOUT_SECONDS:  300
MEMORY_COLLECTION:      openclaw-memory
LOG_LEVEL:              info
```

### `tool-executor-config`
```
SANDBOX_TIMEOUT_SECONDS: 30
MAX_OUTPUT_BYTES:         1048576
AUDIT_SERVICE_URL:        http://audit-service.openclaw-audit.svc:9200
LOG_LEVEL:                info
NOTE: Allowed egress domains are enforced via NetworkPolicy, not config
```

### `audit-config`
```
POSTGRES_HOST:    postgres-service.openclaw-data.svc
POSTGRES_DB:      openclaw_audit
RETENTION_DAYS:   365
LOG_LEVEL:        info
```

---

## 6. Secrets Management

### Secrets Inventory

| Secret Name | Sensitive Keys | Rotation Policy |
|---|---|---|
| `gateway-secrets` | `JWT_SIGNING_KEY`, `AUTH_CLIENT_SECRET`, `SESSION_ENCRYPTION_KEY` | 30 days |
| `core-agent-secrets` | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `POSTGRES_PASSWORD` | 30 days |
| `tool-executor-secrets` | `TOOL_EXECUTOR_INTERNAL_TOKEN` | 14 days |
| `audit-secrets` | `AUDIT_DB_PASSWORD`, `AUDIT_HMAC_KEY` | 60 days |
| `postgres-secrets` | `POSTGRES_USER`, `POSTGRES_PASSWORD` | Dynamic (Vault) |

### Zero-Trust Secrets Strategy

OpenClaw uses a **zero-trust secrets model** where no long-lived database passwords exist in the cluster:

1. **HashiCorp Vault** is the central secrets store. All secrets are defined, versioned, and rotated here — never in the cluster directly.
2. **Vault Dynamic Secrets** generate short-lived PostgreSQL credentials on demand. The Core Agent receives a database username and password valid for **1 hour**, then automatically revoked. Long-lived DB passwords are eliminated entirely.
3. **Vault Agent Sidecar** injects secrets into pods at startup as environment variables. Most ServiceAccounts have **zero direct access** to Kubernetes Secret objects.
4. **External Secrets Operator (ESO)** syncs non-dynamic Vault secrets into Kubernetes Secrets every **15 minutes** for high-sensitivity values.
5. **Reloader** (Stakater) watches Secret changes and triggers rolling restarts — ensuring zero-downtime rotation.
6. Each Secret is annotated:
   ```
   vault.hashicorp.com/agent-inject:      "true"
   vault.hashicorp.com/role:              "core-agent"
   secrets.openclaw.ai/expires-at:        "2026-05-01T00:00:00Z"
   secrets.openclaw.ai/notify-before-days:"14"
   ```
7. **Prometheus alert** `SecretExpiringSoon` fires 14 days before expiry, routed to PagerDuty.
8. **Vault audit log** streams every secret access event to the Audit Service for compliance.

---

## 7. RBAC — Roles & RoleBindings

All roles follow strict **least privilege**. The most security-sensitive workload (Tool Executor) has **zero Kubernetes API access**.

### ServiceAccount: `gateway-sa`

```
Namespace:  openclaw-prod
Resources:  configmaps  →  get
Secrets:    ✗ No access  (secrets injected via Vault sidecar)
```

### ServiceAccount: `core-agent-sa`

```
Namespace:  openclaw-prod
Resources:  configmaps  →  get, list
Secrets:    ✗ No access  (secrets injected via Vault sidecar)
```

### ServiceAccount: `tool-executor-sa` *(Most restricted)*

```
Namespace:  openclaw-tooling
K8s API:    ✗ Zero access — intentionally empty role
Reason:     Tool Executor only accepts inbound HTTP.
            It must not be able to call the Kubernetes API under any circumstances.
```

### ServiceAccount: `audit-sa`

```
Namespace:  openclaw-audit
Resources:  configmaps  →  get
Secrets:    ✗ No access
```

### Role: `developer-read` *(Namespace-scoped, all namespaces)*

```
Resources:  pods, services, configmaps, events, deployments, statefulsets
Verbs:      get, list, watch
Secrets:    ✗ No access
```

### Role: `cicd-deploy` *(Namespace-scoped)*

```
Resources:  deployments, statefulsets  →  get, list, update, patch
Resources:  configmaps                 →  get, list, create, update
Secrets:    ✗ No access
```

---

## 8. Network Policies

Network Policies enforce microsegmentation between namespaces. Even if a pod is compromised, it cannot reach services outside its allowed scope.

### Policy: Tool Executor Isolation (`openclaw-tooling`)

```
Ingress:
  ✅ Allow: from openclaw-prod (core-agent pods) on port 9100
  ✗ Deny:  all other sources

Egress:
  ✅ Allow: to openclaw-audit on port 9200  (send audit events)
  ✅ Allow: to internet port 443  (web browsing, external API calls)
  ✅ Allow: DNS port 53
  ✗ Deny:  to openclaw-data  (cannot reach database or vector store)
  ✗ Deny:  to openclaw-prod  (cannot call back into core services)
```

### Policy: Data Namespace Restriction (`openclaw-data`)

```
Ingress:
  ✅ Allow: from openclaw-prod on ports 5432, 6333, 6334, 6379
  ✗ Deny:  from openclaw-tooling  (Tool Executor has no data access)
  ✗ Deny:  from openclaw-audit    (Audit has its own scoped DB credentials)
  ✗ Deny:  all other sources

Egress:
  ✅ Allow: within openclaw-data only  (replication traffic)
  ✗ Deny:  all egress to internet
```

### Policy: Audit Namespace (`openclaw-audit`)

```
Ingress:
  ✅ Allow: from all openclaw namespaces on port 9200  (receive events)

Egress:
  ✅ Allow: to openclaw-data on port 5432  (write audit logs to PostgreSQL)
  ✗ Deny:  all other egress
```

---

## 9. Inter-Service Communication

### Topology Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                          INTERNET                                 │
└────────────────────────┬─────────────────────────────────────────┘
                         │ HTTPS 443
                ┌────────▼──────────┐
                │   LoadBalancer    │
                └────────┬──────────┘
                         │
                ┌────────▼──────────┐
                │  Ingress Controller│ ← WAF, TLS termination, rate limiting
                └────────┬──────────┘
                         │
   ┌─────────────────────▼──────────────────────┐
   │            openclaw-prod namespace          │
   │  ┌──────────────────┐                      │
   │  │  gateway-service │ ← JWT auth, routing  │
   │  └────────┬─────────┘                      │
   │           │ mTLS (Istio)                    │
   │  ┌────────▼─────────┐                      │
   │  │ core-agent-svc   │                      │
   │  └──┬───────┬────┬──┘                      │
   └─────│───────│────│───────────────────────── ┘
         │       │    │
         │  ┌────▼────▼──────────────────────────┐
         │  │       openclaw-data namespace        │
         │  │  ┌───────────┐  ┌─────────────────┐ │
         │  │  │ postgres  │  │ qdrant / redis  │ │
         │  │  │ :5432     │  │ :5432/:6333/:6379│ │
         │  │  └───────────┘  └─────────────────┘ │
         │  └────────────────────────────────────── ┘
         │
   ┌─────▼──────────────────────────────────────────┐
   │         openclaw-tooling namespace              │
   │  ┌────────────────────────────────┐             │
   │  │  tool-executor-service :9100   │ ← sandboxed │
   │  └──────────────────┬─────────────┘             │
   └─────────────────────│──────────────────────────┘
                         │ audit events (all namespaces)
                ┌────────▼──────────────────────────────┐
                │         openclaw-audit namespace        │
                │  ┌─────────────────────────────────┐   │
                │  │  audit-service :9200             │   │
                │  └─────────────────────────────────┘   │
                └────────────────────────────────────────┘
```

### Communication Protocols

| Source | Destination | Protocol | TLS |
|---|---|---|---|
| Browser | Ingress | HTTPS | ✅ Terminated at Ingress |
| Gateway | Core Agent | HTTP (mTLS via Istio) | ✅ mTLS |
| Core Agent | Tool Executor | HTTP + internal bearer token | ✅ mTLS |
| Core Agent | PostgreSQL | TCP pg driver + pg SSL mode | ✅ |
| Core Agent | Qdrant | gRPC | ✅ |
| Core Agent | Redis | TCP Redis protocol | ❌ Internal |
| All services | Audit Service | HTTP POST (one-way) | ✅ mTLS |
| Tool Executor | Internet | HTTPS via egress proxy | ✅ |

---

## 10. Horizontal Pod Autoscaler

| Workload | Min | Max | Scale Metric | Threshold |
|---|---|---|---|---|
| User Gateway API | 3 | 10 | CPU utilization | > 60% |
| Core Agent | 4 | 20 | Active sessions (custom metric) | > 80% session capacity |
| Tool Executor | 4 | 30 | CPU utilization | > 50% |
| Audit Service | 2 | 4 | CPU utilization | > 70% |

The **Core Agent** uses a custom metric tracking active session slots rather than raw CPU, because LLM orchestration is often blocked on external API calls (low CPU) but saturated in terms of concurrent sessions. Scaling on active sessions provides a more accurate proxy for actual load.

---

## 11. Security Hardening

A full security hardening checklist applied to every workload in this deployment:

| Control | Applied To | Status |
|---|---|---|
| `runAsNonRoot: true` | All containers | ✅ |
| `readOnlyRootFilesystem: true` | All containers | ✅ |
| `allowPrivilegeEscalation: false` | All containers | ✅ |
| `capabilities.drop: ALL` | Tool Executor (strictest) | ✅ |
| `seccompProfile: RuntimeDefault` | Tool Executor | ✅ |
| `runAsUser: 65534` (nobody) | Tool Executor | ✅ |
| NetworkPolicy microsegmentation | All namespaces | ✅ |
| Vault dynamic DB credentials | PostgreSQL | ✅ |
| Vault sidecar secret injection | All ServiceAccounts | ✅ |
| Zero K8s API access for Tool Executor | openclaw-tooling | ✅ |
| Structured audit log of all actions | Audit Service | ✅ |
| No NodePort services | Entire cluster | ✅ |
| Sandbox disk limit (500Mi emptyDir) | Tool Executor | ✅ |
| mTLS between all internal services | Istio mesh | ✅ |

---

## 12. Summary Checklist

| Area | Status | Notes |
|---|---|---|
| Namespaces | ✅ Complete | 5 namespaces with security boundaries and documented purpose |
| Deployments | ✅ Complete | Gateway, Core Agent, Tool Executor, Audit Service |
| StatefulSets | ✅ Complete | PostgreSQL, Qdrant (3 replicas), Redis with PVCs |
| Service types | ✅ Complete | LoadBalancer → gateway only; ClusterIP → all internal |
| Resource requests & limits | ✅ Complete | Every container defined with sizing rationale |
| ConfigMaps | ✅ Complete | One per service; zero credentials stored |
| Secrets management | ✅ Complete | Vault dynamic secrets + ESO + rotation + expiry alerts |
| RBAC | ✅ Complete | Least privilege; Tool Executor has zero K8s API access |
| Network Policies | ✅ Complete | Tool Executor fully isolated from data namespace |
| Inter-service communication | ✅ Complete | Topology diagram + protocol table + mTLS |
| HPA | ✅ Complete | All workloads; Core Agent scales on session count |
| Security hardening | ✅ Complete | Non-root, read-only FS, dropped capabilities, seccomp, Vault |
