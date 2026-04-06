# Kubernetes Deployment Plans
## AI-400 Class 22 - Project 2

This project covers Kubernetes deployment planning for two real-world application scenarios.
No code is written - the focus is on architecture, security, and infrastructure design decisions.

---

## Files

| File | Description |
|---|---|
| `plan1-ai-task-manager.md` | Scenario 1: AI Native Task Manager |
| `plan2-ai-employee-openclaw.md` | Scenario 2: AI Employee using OpenClaw |
| `k8-planning-skill.md` | Reusable K8 Planning Skill for future projects |

---

## What Each Plan Covers

- Namespaces and environment isolation
- Deployments and StatefulSets with replica strategy
- Service types (ClusterIP, LoadBalancer, Headless)
- Resource requests and limits for every container
- ConfigMaps for non-sensitive configuration
- Secrets management with rotation and expiry handling
- RBAC roles and ServiceAccounts
- Inter-service communication and topology diagrams
