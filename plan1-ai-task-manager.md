# Kubernetes Deployment Plan
## Scenario 1: AI Native Task Manager

**Course:** AI-400 | **Class:** 22 | **Date:** April 6, 2026  
**Type:** Planning Document — No executable code

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
9. [Inter-Service Communication](#8-inter-service-communication)
10. [Horizontal Pod Autoscaler](#9-horizontal-pod-autoscaler)
11. [Summary Checklist](#10-summary-checklist)

---

## Overview

The **AI Native Task Manager** is a cloud-native application composed of four independently deployable services. External traffic enters exclusively through a managed Ingress controller. All internal services communicate over the private cluster network.

| Service | Role |
|---|---|
| **UI Interface** | React frontend served to end users |
| **Backend APIs** | Core business logic, REST/GraphQL endpoints, WebSocket support |
| **Task Agent** | AI-powered autonomous task execution engine (LLM-backed) |
| **Notification Service** | Email, push, and in-app notification delivery |

Supporting infrastructure: **PostgreSQL** (relational database) and **Redis** (cache), both deployed as StatefulSets with persistent storage.

---

## 1. Namespaces

Namespaces provide logical isolation, simplified RBAC scoping, and independent resource quotas per environment.

| Namespace | Purpose |
|---|---|
| `task-manager-prod` | All production application workloads |
| `task-manager-staging` | Pre-production testing and QA environment |
| `task-manager-monitoring` | Prometheus, Grafana, and alerting stack |

> **Design note:** The monitoring namespace is intentionally separate from the application namespace. This prevents a misbehaving workload from consuming resources that the observability stack needs, and limits RBAC so monitoring agents can read metrics without accessing application secrets.

---

## 2. Pods, Deployments & StatefulSets

### 2.1 — UI Interface (Deployment)

The frontend is stateless and optimized for fast, zero-downtime rolling deployments.

```
Name:       ui-deployment
Namespace:  task-manager-prod
Replicas:   2  (HPA: min 2, max 6)
Strategy:   RollingUpdate  |  maxSurge: 1  |  maxUnavailable: 0

Container: ui
  Image:   task-manager/ui:latest
  Port:    3000
  Resources:
    requests:  cpu 100m  |  memory 128Mi
    limits:    cpu 500m  |  memory 512Mi
  EnvFrom:     configMapRef → ui-config
  Probes:
    livenessProbe:   GET /health  port 3000  initialDelay 15s
    readinessProbe:  GET /ready   port 3000  initialDelay 5s
```

---

### 2.2 — Backend APIs (Deployment)

The API layer is stateless with a higher replica floor of 3 to handle concurrent user traffic reliably.

```
Name:       backend-deployment
Namespace:  task-manager-prod
Replicas:   3  (HPA: min 3, max 10)
Strategy:   RollingUpdate  |  maxSurge: 1  |  maxUnavailable: 0

Container: backend-api
  Image:   task-manager/backend:latest
  Port:    8080
  Resources:
    requests:  cpu 250m   |  memory 256Mi
    limits:    cpu 1000m  |  memory 1Gi
  EnvFrom:
    configMapRef → backend-config
    secretRef    → backend-secrets
  Probes:
    livenessProbe:   GET /api/health  port 8080  initialDelay 15s
    readinessProbe:  GET /api/ready   port 8080  initialDelay 5s
```

---

### 2.3 — Task Agent (Deployment)

The Task Agent runs LLM inference workloads. It is stateless (task state persisted in PostgreSQL) but CPU and memory intensive. HPA is configured on both CPU utilization and a custom queue-depth metric so the cluster scales ahead of demand.

```
Name:       task-agent-deployment
Namespace:  task-manager-prod
Replicas:   2  (HPA: min 2, max 8)
Strategy:   RollingUpdate  |  maxSurge: 2  |  maxUnavailable: 0

Container: task-agent
  Image:   task-manager/task-agent:latest
  Port:    9090
  Resources:
    requests:  cpu 500m   |  memory 512Mi
    limits:    cpu 2000m  |  memory 2Gi
  EnvFrom:
    configMapRef → task-agent-config
    secretRef    → task-agent-secrets
  Probes:
    livenessProbe:   GET /agent/health  port 9090  initialDelay 30s
    readinessProbe:  GET /agent/ready   port 9090  initialDelay 10s
```

> **Why higher CPU limits?** The Task Agent makes real-time calls to external LLM APIs and performs local prompt assembly, token counting, and result parsing. Throttling it causes timeouts under load.

---

### 2.4 — Notification Service (Deployment)

Handles outbound email, push, and in-app notifications. Lightweight and I/O-bound.

```
Name:       notification-deployment
Namespace:  task-manager-prod
Replicas:   2  (HPA: min 2, max 5)
Strategy:   RollingUpdate

Container: notification-service
  Image:   task-manager/notifications:latest
  Port:    8081
  Resources:
    requests:  cpu 100m  |  memory 128Mi
    limits:    cpu 500m  |  memory 512Mi
  EnvFrom:
    configMapRef → notification-config
    secretRef    → notification-secrets
  Probes:
    livenessProbe:   GET /health  port 8081  initialDelay 10s
    readinessProbe:  GET /ready   port 8081  initialDelay 5s
```

---

### 2.5 — PostgreSQL (StatefulSet)

PostgreSQL requires a StatefulSet to guarantee stable network identity and persistent volume binding across pod restarts. A separate read-replica StatefulSet handles read-heavy query offloading.

```
Name:        postgres-statefulset
Namespace:   task-manager-prod
Replicas:    1 primary  +  1 read replica (separate StatefulSet)
ServiceName: postgres-headless

Container: postgres
  Image:   postgres:16
  Port:    5432
  Resources:
    requests:  cpu 500m   |  memory 1Gi
    limits:    cpu 2000m  |  memory 4Gi
  EnvFrom:     secretRef → postgres-secrets
  VolumeMount: postgres-data → /var/lib/postgresql/data

VolumeClaimTemplate:
  storageClassName: ssd-retain
  accessModes:      ReadWriteOnce
  storage:          20Gi
```

---

### 2.6 — Redis Cache (StatefulSet)

Redis provides low-latency caching for session tokens, API rate limits, and frequently accessed task state. Deployed as a StatefulSet with persistence to survive pod restarts.

```
Name:       redis-statefulset
Namespace:  task-manager-prod
Replicas:   1

Container: redis
  Image:   redis:7-alpine
  Port:    6379
  Resources:
    requests:  cpu 100m  |  memory 256Mi
    limits:    cpu 500m  |  memory 1Gi
  VolumeMount: redis-data → /data

VolumeClaimTemplate:
  storage: 5Gi
```

---

## 3. Services & Networking

### Service Definitions

| Service Name | Type | Port | Routes To | Justification |
|---|---|---|---|---|
| `ui-service` | LoadBalancer | 80 / 443 | ui-deployment : 3000 | Public-facing; requires external IP |
| `backend-service` | ClusterIP | 8080 | backend-deployment | Internal only; reached via Ingress |
| `task-agent-service` | ClusterIP | 9090 | task-agent-deployment | Internal HTTP from backend |
| `notification-service` | ClusterIP | 8081 | notification-deployment | Internal; triggered by backend/agent |
| `postgres-service` | ClusterIP | 5432 | postgres-statefulset | Internal database access only |
| `postgres-headless` | Headless | 5432 | postgres-statefulset | Pod DNS identity for replication |
| `redis-service` | ClusterIP | 6379 | redis-statefulset | Internal cache access only |

> **No NodePort services** in production. All external traffic routes through the LoadBalancer and Ingress controller only.

### Ingress Routing Rules

```
HTTPS 443 → Ingress Controller (nginx / AWS ALB)
  /          → ui-service : 3000         (React SPA)
  /api/*     → backend-service : 8080    (REST API)
  /ws/*      → backend-service : 8080    (WebSocket upgrade)

TLS:          Managed by cert-manager (Let's Encrypt or AWS ACM)
Rate Limit:   60 requests / minute / IP (enforced at Ingress)
```

---

## 4. Resource Requests & Limits

All containers define both `requests` (guaranteed allocation) and `limits` (hard ceiling) to prevent resource starvation across pods.

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| UI Interface | 100m | 500m | 128Mi | 512Mi |
| Backend APIs | 250m | 1000m | 256Mi | 1Gi |
| Task Agent | 500m | 2000m | 512Mi | 2Gi |
| Notification Service | 100m | 500m | 128Mi | 512Mi |
| PostgreSQL | 500m | 2000m | 1Gi | 4Gi |
| Redis | 100m | 500m | 256Mi | 1Gi |

**Sizing rationale:**
- **Task Agent** receives the highest CPU ceiling because LLM prompt assembly and response parsing are CPU-intensive at burst time.
- **PostgreSQL** is given 4Gi memory to accommodate its shared buffer pool (recommended: 25% of available RAM for optimal query performance).
- **UI and Notification Service** are deliberately light — they serve static assets and forward messages respectively.

---

## 5. ConfigMaps

ConfigMaps hold all **non-sensitive** configuration. No credentials, tokens, or API keys are stored here.

### `ui-config`
```
UI_API_BASE_URL:       http://backend-service.task-manager-prod.svc:8080/api
UI_WEBSOCKET_URL:      ws://backend-service.task-manager-prod.svc:8080/ws
UI_FEATURE_AI_TASKS:   true
UI_FEATURE_DARK_MODE:  true
NODE_ENV:              production
LOG_LEVEL:             info
```

### `backend-config`
```
DATABASE_HOST:         postgres-service.task-manager-prod.svc
DATABASE_PORT:         5432
DATABASE_NAME:         taskmanager
DATABASE_MAX_POOL:     20
REDIS_HOST:            redis-service.task-manager-prod.svc
REDIS_PORT:            6379
TASK_AGENT_URL:        http://task-agent-service.task-manager-prod.svc:9090
NOTIFICATION_URL:      http://notification-service.task-manager-prod.svc:8081
REQUEST_TIMEOUT_MS:    30000
LOG_LEVEL:             info
```

### `task-agent-config`
```
AGENT_MODEL:            claude-opus-4
AGENT_MAX_STEPS:        20
AGENT_TIMEOUT_SECONDS:  120
BACKEND_CALLBACK_URL:   http://backend-service.task-manager-prod.svc:8080/api/agent/callback
QUEUE_POLL_INTERVAL_MS: 500
LOG_LEVEL:              info
```

### `notification-config`
```
EMAIL_FROM:           noreply@taskmanager.app
EMAIL_PROVIDER:       sendgrid
PUSH_PROVIDER:        firebase
RETRY_MAX_ATTEMPTS:   3
RETRY_BACKOFF_MS:     1000
LOG_LEVEL:            info
```

---

## 6. Secrets Management

### Secrets Inventory

| Secret Name | Sensitive Keys | Rotation Schedule |
|---|---|---|
| `backend-secrets` | `DATABASE_PASSWORD`, `JWT_SECRET`, `REDIS_PASSWORD` | 90 days |
| `task-agent-secrets` | `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` | 30 days |
| `notification-secrets` | `SENDGRID_API_KEY`, `FIREBASE_SERVER_KEY` | 60 days |
| `postgres-secrets` | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | 90 days |

### Rotation & Expiry Strategy

All secrets are managed via **AWS Secrets Manager** (or HashiCorp Vault) and synced into the cluster by the **External Secrets Operator (ESO)**:

1. Each external secret carries a `rotation-schedule` tag (e.g., `rotate-every: 30d`).
2. ESO polls every **60 minutes**. When a new version is detected, the Kubernetes Secret is updated automatically — no manual intervention required.
3. **Reloader** (Stakater controller) watches Secret changes and triggers a rolling restart of affected Deployments, ensuring **zero-downtime rotation**.
4. Each Kubernetes Secret is annotated with its expiry date:
   ```
   secrets.manager/expires-at:        "2026-07-01T00:00:00Z"
   secrets.manager/rotate-alert-days: "7"
   ```
5. A **Prometheus alert** fires `SecretExpiringSoon` when the expiry window is within 7 days, routed to the on-call Slack channel via Alertmanager.

---

## 7. RBAC — Roles & RoleBindings

All roles follow the **principle of least privilege**: each identity receives only the permissions required to function.

### Role: `developer-read` *(Namespace-scoped)*

Grants developers read-only visibility into running workloads. No access to Secrets.

```
Namespace:  task-manager-prod
Resources:  pods, services, configmaps, events, deployments, replicasets
Verbs:      get, list, watch
Secrets:    ✗ No access
```

### Role: `cicd-deploy` *(Namespace-scoped)*

Used by CI/CD pipelines (GitHub Actions, GitLab CI) to update image versions and configuration.

```
Namespace:  task-manager-prod
Resources:  deployments, statefulsets   →  get, list, update, patch
Resources:  configmaps                  →  get, list, create, update, patch
Secrets:    ✗ No access
```

### ServiceAccount: `task-agent-sa`

The Task Agent reads only its own ConfigMap. Secrets are injected at pod startup via `secretRef` in the Deployment spec — no direct Secret API access needed.

```
Namespace:  task-manager-prod
Resources:  configmaps  →  get, list
Secrets:    ✗ No access
```

### ClusterRole: `monitoring-reader`

Prometheus requires cluster-wide read access to scrape metrics across all namespaces. This is the only ClusterRole defined in the system.

```
Resources:  pods, nodes, services, endpoints   →  get, list, watch
Resources:  metrics.k8s.io/pods, nodes         →  get, list
```

---

## 8. Inter-Service Communication

### Topology Diagram

```
                          ┌─────────────────┐
                          │    INTERNET      │
                          └────────┬─────────┘
                                   │ HTTPS 443
                          ┌────────▼─────────┐
                          │  LoadBalancer /  │
                          │ Ingress Controller│  ← TLS, rate limiting
                          └────────┬─────────┘
                                   │
                   ┌───────────────┴───────────────┐
                   │ /                             │ /api/*  /ws/*
          ┌────────▼────────┐             ┌────────▼────────┐
          │   ui-service    │             │ backend-service  │
          │ (UI Pods :3000) │             │ (API Pods :8080) │
          └─────────────────┘             └──┬───────┬───┬───┘
                                             │       │   │
                                   ┌─────────▼─┐  ┌──▼──┐  ┌──────────────────┐
                                   │ postgres  │  │redis│  │ task-agent-svc   │
                                   │ :5432     │  │:6379│  │ :9090            │
                                   └───────────┘  └─────┘  └────────┬─────────┘
                                                                     │ HTTP callback
                                                            ┌────────▼─────────┐
                                                            │ notification-svc  │
                                                            │ :8081             │
                                                            └──────────────────┘
```

### Communication Protocols

| Source | Destination | Protocol | TLS Encrypted |
|---|---|---|---|
| Browser | Ingress | HTTPS | ✅ Terminated at Ingress |
| UI Pod | Backend | HTTP via Ingress | ✅ via Ingress |
| UI Pod | Backend | WebSocket via Ingress | ✅ via Ingress |
| Backend | Task Agent | HTTP POST | ❌ Internal plain HTTP |
| Task Agent | Backend (callback) | HTTP POST | ❌ Internal plain HTTP |
| Backend | Notification Service | HTTP POST (async) | ❌ Internal plain HTTP |
| Backend | PostgreSQL | TCP (pg driver) | Optional (pg `sslmode`) |
| Backend | Redis | TCP (Redis protocol) | ❌ Internal plain TCP |

> **Upgrade path:** Deploy **Istio** as a service mesh to automatically upgrade all internal pod-to-pod connections to **mTLS** with zero application code changes, achieving a zero-trust network posture.

---

## 9. Horizontal Pod Autoscaler

| Workload | Min Replicas | Max Replicas | Scale Metric | Threshold |
|---|---|---|---|---|
| UI Interface | 2 | 6 | CPU utilization | > 70% |
| Backend APIs | 3 | 10 | CPU utilization | > 60% |
| Task Agent | 2 | 8 | CPU **or** queue depth | > 50% CPU / > 100 tasks |
| Notification Service | 2 | 5 | CPU utilization | > 70% |

The **Task Agent** uses a custom Prometheus metric (task queue depth) exposed via the Kubernetes Custom Metrics API. This allows the cluster to scale out proactively before CPU saturation occurs, reducing user-perceived latency for AI task execution.

---

## 10. Summary Checklist

| Area | Status | Notes |
|---|---|---|
| Namespaces | ✅ Complete | prod, staging, monitoring with documented purpose |
| Deployments | ✅ Complete | UI, Backend, Task Agent, Notification Service |
| StatefulSets | ✅ Complete | PostgreSQL (+ read replica), Redis with PVCs |
| Service types | ✅ Complete | LoadBalancer → UI; ClusterIP → all internal; Headless → DB |
| Resource requests & limits | ✅ Complete | Every container defined with rationale |
| ConfigMaps | ✅ Complete | One per service; zero credentials stored |
| Secrets management | ✅ Complete | ESO + rotation schedules + Prometheus expiry alerts |
| RBAC | ✅ Complete | developer-read, cicd-deploy, per-component ServiceAccounts |
| Inter-service communication | ✅ Complete | Topology diagram + protocol table + mTLS upgrade path |
| HPA | ✅ Complete | All stateless workloads; Task Agent uses custom queue metric |
| Ingress / TLS | ✅ Complete | cert-manager managed; rate limiting enforced |
