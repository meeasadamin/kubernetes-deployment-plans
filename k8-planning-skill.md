# K8 Planning Skill
## Reusable Agent Skill for Kubernetes Deployment Planning

**Skill Name:** K8 Planning Skill  
**Version:** 1.0.0  
**Author:** AI-400 Class 22  
**Date:** April 6, 2026  
**Type:** Agent Skill Document

---

## Table of Contents

1. [Skill Identity & Purpose](#1-skill-identity--purpose)
2. [Trigger Conditions](#2-trigger-conditions)
3. [Input Schema](#3-input-schema)
4. [Output Format](#4-output-format)
5. [Generation Rules](#5-generation-rules)
6. [Example Invocation](#6-example-invocation)
7. [Quality Self-Review Checklist](#7-quality-self-review-checklist)
8. [Evaluation Criteria](#8-evaluation-criteria)
9. [Skill Limitations](#9-skill-limitations)
10. [Changelog](#10-changelog)

---

## 1. Skill Identity & Purpose

This skill enables an AI agent to produce **comprehensive, production-ready Kubernetes deployment plans** in Markdown format for any given application — no executable YAML or code is generated, only structured planning documents.

Given a project name, a list of services, and optional configuration parameters, the agent will produce a complete deployment plan covering all standard Kubernetes concerns:

- Namespace isolation strategy
- Workload definitions (Deployments and StatefulSets)
- Service types and networking
- Resource requests and limits
- ConfigMap design
- Secrets management and rotation
- RBAC roles and ServiceAccounts
- Inter-service communication topology
- Horizontal Pod Autoscaler configuration
- Security hardening (scaled to the requested security level)

**Design principles:**
- **Reusable** — works for any project type: web apps, AI agents, microservices, data pipelines
- **Security-aware** — built-in escalation from standard to high to critical security posture
- **Consistent** — every output follows the same structure so plans are comparable and reviewable
- **Explainable** — every decision includes a rationale comment, not just the configuration value

---

## 2. Trigger Conditions

Activate this skill when the user:

- Asks to "plan a Kubernetes deployment" for any system
- Provides a list of services and asks how to deploy them on K8s
- Asks for a K8s architecture document, deployment spec, or infrastructure plan
- Mentions any of these phrases: `deployment plan`, `k8s plan`, `kubernetes architecture`, `deploy to k8s`, `K8 planning`, `cluster design`, `k8s setup`

Do **not** activate this skill for requests to generate actual YAML manifests, Helm charts, or Terraform — those are code generation tasks, not planning tasks.

---

## 3. Input Schema

Collect all required inputs before generating the plan. If any required field is missing, ask the user before proceeding.

### Required Inputs

```
project_name:     string
  Description:    Short name of the project (used in namespace names, labels, etc.)
  Example:        "ai-task-manager", "openclaw", "ecommerce-platform"

services:         list of objects
  Each service must include:
    name:         string   — Human-readable service name
    type:         string   — One of: frontend | api | agent | worker | database | cache | queue | other
    description:  string   — What this service does (1–2 sentences)
```

### Optional Inputs (with Defaults)

```
environment:
  Type:     string
  Options:  prod | staging | dev
  Default:  prod
  Effect:   Affects replica counts, resource sizing, and alert thresholds

security_level:
  Type:     string
  Options:  standard | high | critical
  Default:  standard
  Effects:
    standard  → Kubernetes Secrets, basic RBAC, no NetworkPolicies
    high      → Adds: External Secrets Operator, Reloader, expiry alerts, mTLS recommendation
    critical  → Adds all of "high" plus: Vault dynamic secrets, Vault sidecar injection,
                seccomp profiles, dropped Linux capabilities, read-only filesystems,
                NetworkPolicies, dedicated audit namespace

cloud_provider:
  Type:     string
  Options:  aws | gcp | azure | on-prem | generic
  Default:  generic
  Effect:   Adjusts LoadBalancer annotations and StorageClass names

has_stateful_services:
  Type:     bool
  Default:  inferred from service types
  Effect:   If true, generates StatefulSets + PersistentVolumeClaims for databases/caches
```

---

## 4. Output Format

The agent must produce a **single Markdown document** with sections in the following order. No section may be skipped. Each section heading must be an H2 (`##`) with a numbered prefix.

```
## Overview
## 1. Namespaces
## 2. Pods, Deployments & StatefulSets
## 3. Services & Networking
## 4. Resource Requests & Limits
## 5. ConfigMaps
## 6. Secrets Management
## 7. RBAC — Roles & RoleBindings
## 8. [Network Policies]         ← include only if security_level: high or critical
## 9. Inter-Service Communication
## 10. Horizontal Pod Autoscaler
## 11. [Security Hardening]      ← include only if security_level: critical
## 12. Summary Checklist
```

The document must begin with a metadata header:
```
# Kubernetes Deployment Plan
## [Project Name]

**Course / Team:** [value]  |  **Date:** [value]  |  **Security Level:** [value]
**Type:** Planning Document — No executable code
```

---

## 5. Generation Rules

### Rule 1: Namespace Strategy

Determine namespaces based on the service list and security level:

| Condition | Namespaces to Create |
|---|---|
| Always | `{project}-prod`, `{project}-monitoring` |
| Any staging/dev services exist | `{project}-staging` |
| `security_level: high or critical` | Add `{project}-data` (databases/caches only), `{project}-audit` |
| Worker or agent services making external HTTP calls | Add `{project}-workers` or `{project}-tooling` |

Every namespace entry must include a one-sentence justification in the plan.

---

### Rule 2: Workload Type Selection

Use this decision table to assign a workload kind to each service:

| Service Type | Workload Kind | Reason |
|---|---|---|
| frontend | Deployment | Stateless; fast rollout needed |
| api | Deployment | Stateless; horizontal scaling |
| agent / worker | Deployment | Stateless; HPA on custom metrics |
| database (relational) | StatefulSet | Stable network identity; persistent storage |
| database (vector) | StatefulSet | Distributed; persistent storage |
| cache (Redis, Memcached) | StatefulSet | Persistence recommended in production |
| queue (Kafka, RabbitMQ) | StatefulSet | Ordered, persistent; stable pod names required |
| other | Ask the user | Cannot safely assume stateful vs stateless |

Every workload definition must include:
- `replicas` with an HPA range (min–max)
- `strategy` type with rationale
- `livenessProbe` and `readinessProbe`
- `securityContext` appropriate to the security level (see Rule 8)

---

### Rule 3: Service Type Selection

| Exposure Requirement | Service Type |
|---|---|
| Public internet access required | LoadBalancer |
| StatefulSet pod DNS identity | Headless |
| Internal cluster communication | ClusterIP |
| Temporary dev/debug external access | NodePort |

Additional rules:
- **Never use NodePort in production** unless explicitly requested by the user.
- Every LoadBalancer service must be paired with an Ingress controller for TLS termination, routing rules, and rate limiting.
- Ingress routing rules must be documented in the plan.

---

### Rule 4: Resource Sizing

Use these baseline tiers. Adjust upward for compute-intensive services (LLM inference, image processing, vector search).

| Service Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---|---|---|---|---|
| frontend | 100m | 500m | 128Mi | 512Mi |
| api (light) | 200m | 1000m | 256Mi | 1Gi |
| api (heavy) | 500m | 2000m | 512Mi | 2Gi |
| agent / AI worker | 500m | 2000m | 1Gi | 4Gi |
| database (small) | 250m | 1000m | 512Mi | 2Gi |
| database (large) | 500m | 2000m | 2Gi | 8Gi |
| cache | 100m | 500m | 256Mi | 2Gi |

Every container must have both `requests` and `limits` defined. If a limit differs significantly from the baseline, include a one-sentence rationale.

---

### Rule 5: ConfigMap Design

For each service, create a ConfigMap containing:
- All non-sensitive environment variables
- Service discovery URLs using full Kubernetes DNS format: `{service}.{namespace}.svc`
- Feature flags (if mentioned in the project description)
- Connection pool sizes and timeout values
- Log level

**Hard rule:** Never put any of the following in a ConfigMap: passwords, API keys, tokens, signing keys, or any credentials. If the user's description implies a credential should go in a ConfigMap, flag it as a security issue and move it to a Secret.

---

### Rule 6: Secrets Management

Apply the tier that matches the requested `security_level`:

**Standard:**
- Use Kubernetes Secrets (base64 encoded in etcd)
- Document all keys stored in each Secret
- Assign a rotation policy per secret type:
  - LLM / AI API keys: 30 days
  - Database passwords: 90 days
  - JWT / session signing keys: 30 days
  - Internal service tokens: 14 days
- Annotate each Secret with `expires-at` and `rotate-alert-days`

**High (adds to Standard):**
- Sync secrets from AWS Secrets Manager or HashiCorp Vault via **External Secrets Operator (ESO)**
- Use **Reloader** (Stakater) for zero-downtime secret rotation
- Add a Prometheus alert rule `SecretExpiringSoon` triggering 14 days before expiry
- ESO poll interval: 60 minutes (15 minutes for high-sensitivity secrets)

**Critical (adds to High):**
- Use **Vault Dynamic Secrets** for database credentials (short-lived, auto-revoked — no long-lived DB passwords)
- Use **Vault Agent Sidecar** to inject secrets into pods at startup; most ServiceAccounts have zero direct K8s Secret access
- Stream Vault audit logs to the Audit Service for compliance
- Log every secret access event

---

### Rule 7: RBAC Design

Always define a minimum of three identities:

| Identity | Permissions |
|---|---|
| `developer-read` role | get, list, watch on pods, services, configmaps, events, deployments (no Secrets) |
| `cicd-deploy` role | get/list/update/patch on deployments and configmaps (no Secrets) |
| One ServiceAccount per application component | Scoped to only what that component needs |

For `security_level: high or critical`:
- ServiceAccounts for sensitive services receive no K8s Secrets access (Vault sidecar injection only)
- Worker / Tool Executor ServiceAccounts receive **zero K8s API access** — document this explicitly
- All roles are namespace-scoped (no ClusterRole) unless monitoring requires cross-namespace metric scraping

---

### Rule 8: Security Context (Scaled by Security Level)

| Field | Standard | High | Critical |
|---|---|---|---|
| `runAsNonRoot` | Recommended | Required | Required |
| `allowPrivilegeEscalation: false` | Recommended | Required | Required |
| `readOnlyRootFilesystem` | Optional | Required | Required |
| `runAsUser` | Not specified | Non-zero UID | 65534 (nobody) for sandboxed workers |
| `capabilities.drop: ALL` | Not required | Recommended | Required for sandboxed workers |
| `seccompProfile: RuntimeDefault` | Not required | Optional | Required for sandboxed workers |

---

### Rule 9: Inter-Service Communication

Every plan must include:
1. An ASCII topology diagram showing all services, their namespace boundaries, and the direction of every connection
2. A protocol table listing: Source → Destination, Protocol (HTTP/gRPC/TCP/etc.), and whether TLS is applied
3. An mTLS upgrade path note if the security level is `high` or `critical` (recommend Istio or Linkerd)

For `security_level: critical`:
- Add a Network Policies section with ingress and egress rules for each namespace
- Explicitly document which namespaces are **denied** from reaching the data namespace

---

### Rule 10: HPA Configuration

For every Deployment, define an HPA entry with:
- `minReplicas` and `maxReplicas`
- Primary scaling metric (CPU by default)
- Custom metric if the service is I/O-bound or queue-driven (e.g., task queue depth, active session count)
- Target utilization threshold

StatefulSets generally do not use HPA. Note this in the plan with a brief explanation.

---

### Rule 11: Summary Checklist

End every plan with a table-format checklist. Use ✅ for completed items and include a brief note column:

```
| Area | Status | Notes |
|---|---|---|
| Namespaces | ✅ Complete | [namespace names and count] |
| Deployments | ✅ Complete | [service names] |
| StatefulSets | ✅ Complete | [service names + PVC sizes] |
| Service types | ✅ Complete | [types used] |
| Resource limits | ✅ Complete | All containers defined |
| ConfigMaps | ✅ Complete | Zero credentials stored |
| Secrets management | ✅ Complete | [rotation strategy summary] |
| RBAC | ✅ Complete | [roles defined] |
| Inter-service communication | ✅ Complete | Topology diagram included |
| HPA | ✅ Complete | All stateless workloads covered |
| [Network Policies] | ✅ Complete | [if applicable] |
| [Security Hardening] | ✅ Complete | [if applicable] |
```

---

## 6. Example Invocation

### User Prompt

> "Plan a Kubernetes deployment for a SaaS analytics platform with the following services: a React dashboard (frontend), a Python FastAPI backend, a Celery worker for report generation, and a PostgreSQL database. Security level: high. Cloud provider: AWS."

### Agent Reasoning

1. **Map service types:**
   - React dashboard → `frontend` → Deployment
   - FastAPI backend → `api` → Deployment
   - Celery worker → `worker` → Deployment (HPA on task queue depth)
   - PostgreSQL → `database` → StatefulSet + PVC

2. **Determine namespaces** (`security_level: high`):
   - `analytics-prod` — frontend, backend, worker
   - `analytics-data` — PostgreSQL only
   - `analytics-audit` — Audit service
   - `analytics-monitoring` — observability

3. **Apply security level "high":**
   - External Secrets Operator from AWS Secrets Manager
   - Reloader for zero-downtime rotation
   - Prometheus expiry alert
   - mTLS recommendation via Istio
   - NetworkPolicy: deny worker namespace from reaching analytics-data directly

4. **AWS-specific adjustments:**
   - LoadBalancer annotations for AWS ALB
   - StorageClass: `gp3` (AWS EBS)

5. **Output:** Single Markdown file following the 12-section structure defined in §4.

---

## 7. Quality Self-Review Checklist

Before finalizing any plan, the agent must verify each item below internally. Do not output the plan if any item is unchecked.

- [ ] Every service has a named Deployment or StatefulSet
- [ ] Every container has both `resources.requests` and `resources.limits`
- [ ] Every container has `livenessProbe` and `readinessProbe`
- [ ] No credentials, passwords, or tokens appear in any ConfigMap
- [ ] Every Secret has a documented rotation policy and `expires-at` annotation
- [ ] Every namespace has a one-sentence justification
- [ ] The topology diagram includes every service and shows namespace boundaries
- [ ] RBAC roles follow least privilege; Secrets access is not granted unnecessarily
- [ ] HPA is defined for every Deployment
- [ ] `security_level: high/critical` has triggered all required additional controls
- [ ] The Summary Checklist covers all major sections
- [ ] All Kubernetes DNS URLs use the full format: `{service}.{namespace}.svc`

---

## 8. Evaluation Criteria

When evaluating the quality of a plan generated by this skill, score it against the following rubric. A score of **80% or above is required** to pass.

| Criterion | Weight | Pass Condition |
|---|---|---|
| All required sections present and in correct order | 20% | All sections exist; none skipped |
| Resources defined on every container with rationale | 15% | Both requests and limits on every container |
| Zero credentials in any ConfigMap | 15% | No passwords, tokens, or API keys found |
| Secrets have rotation policies and expiry annotations | 10% | Every secret has a schedule and `expires-at` |
| RBAC follows least privilege throughout | 15% | No over-permissioned role; Secrets restricted |
| Topology diagram is present and accurate | 10% | Diagram matches the service list in the plan |
| Security level controls correctly applied | 15% | Standard/high/critical controls match the input |

**Scoring:**
- 80–100%: Pass — plan is production-ready
- 60–79%: Revise — one or more sections need improvement
- Below 60%: Fail — regenerate with corrected inputs

---

## 9. Skill Limitations

- This skill produces **planning documents only** — it does not generate executable YAML manifests, Helm charts, or Terraform modules.
- Resource sizing is based on established heuristics. Actual values must be validated through load testing against real traffic patterns.
- Network Policy rules are illustrative. Exact pod label selectors and CIDR blocks must be filled in for a real cluster.
- Vault and ESO configuration details are architectural recommendations. Exact setup depends on the target cloud provider and existing infrastructure.
- This skill does not plan for multi-region or multi-cluster deployments. Those require additional design work beyond the scope of a single plan.

---

## 10. Changelog

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0.0 | 2026-04-06 | AI-400 Class 22 | Initial release — supports standard, high, and critical security levels; covers all 10 core K8s planning concerns |
