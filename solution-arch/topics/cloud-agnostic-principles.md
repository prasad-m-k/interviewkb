# Cloud-Agnostic Principles

**Related:** [[solution-arch/topics/architectural-styles]], [[solution-arch/topics/integration-patterns]]

---

## The 12-Factor App

Methodology for building portable, scalable SaaS applications.

| Factor | Rule | Why |
|--------|------|-----|
| 1. Codebase | One repo, many deploys | Traceability |
| 2. Dependencies | Explicitly declare, isolate | Reproducible builds |
| 3. Config | Store in environment (not code) | Security; env portability |
| 4. Backing services | Treat as attached resources | Swap DB/queue without code change |
| 5. Build, release, run | Strict separation of stages | Rollback; auditability |
| 6. Processes | Stateless; share-nothing | Horizontal scale |
| 7. Port binding | Export services via port | Self-contained; composable |
| 8. Concurrency | Scale via process model | Horizontal scale |
| 9. Disposability | Fast startup, graceful shutdown | Resilience; elasticity |
| 10. Dev/prod parity | Keep environments similar | Eliminate "works on my machine" |
| 11. Logs | Treat as event streams | Centralised log aggregation |
| 12. Admin processes | Run as one-off processes | Database migrations, scripts |

---

## Infrastructure as Code (IaC)

All infrastructure defined in version-controlled code.

```
Git Repo (infra definitions)
       │ PR + review
       ▼
IaC Tool (Terraform / Pulumi / Ansible)
       │ plan → apply
       ▼
Cloud Provider API (cloud-agnostic via providers)
       │
       ▼
Actual infrastructure (VMs, networks, DBs, queues)
```

**Benefits:**
- Reproducible environments (no snowflake servers)
- Audit trail via git history
- Disaster recovery: rebuild infra from code
- Environment parity: same code for dev/staging/prod (parameterised)

**Tools comparison:**

| Tool | Approach | Language | State |
|------|---------|---------|-------|
| Terraform | Declarative | HCL | Remote state file |
| Pulumi | Declarative/Imperative | Python/TS/Go | Pulumi Cloud or S3 |
| Ansible | Procedural | YAML | Stateless (idempotent) |
| CloudFormation | Declarative | YAML/JSON | AWS-only |

---

## GitOps

Git as the single source of truth for **both application and infrastructure** state. A reconciliation agent ensures live state matches desired state in git.

```
Developer
    │ git push
    ▼
Git Repo (desired state)
    │ webhook / poll
    ▼
GitOps Agent (ArgoCD / Flux)
    │ diff: desired vs actual
    ▼
Cluster/Infra (actual state)
```

**Benefits:**
- Rollback = `git revert`
- All changes peer-reviewed via PR
- Full audit trail
- Self-healing (agent converges to git state)

---

## Container Strategy

Containers are the portability layer between code and infrastructure.

```
Code + Dockerfile
    │ docker build
    ▼
Container Image (immutable artifact)
    │ pushed to
    ▼
Container Registry (ECR / GCR / GHCR)
    │ pulled by
    ▼
Container Runtime (Docker / containerd)
    │ managed by
    ▼
Orchestrator (Kubernetes)
    │
    ▼
Any Cloud or On-Prem Hardware
```

**Cloud portability pattern:**
- Use Kubernetes as the abstraction layer
- Use managed Kubernetes (EKS/GKE/AKS) — same API, different control plane
- Avoid cloud-native services that don't have equivalents (vendor lock-in)

---

## Portability vs Productivity Trade-off

```
Highest Portability                       Highest Productivity
(cloud-agnostic)                          (cloud-native)
       │                                          │
   Kubernetes +                            Managed services
   open-source stack                       (DynamoDB, SQS,
   (Kafka, PostgreSQL,                      Cloud Run, etc.)
    Redis, etc.)
       │                                          │
   More: ops burden,                      More: vendor lock-in,
   infra expertise,                       less control,
   cost at scale                          faster to build
```

**Decision framework:**
- Startup / fast iteration → use managed cloud services
- Large-scale / multi-cloud requirement → invest in cloud-agnostic stack
- Most teams → hybrid: managed DB + Kubernetes for compute

---

## Twelve-Factor Config Example

Bad:
```python
DB_HOST = "prod-db.internal.company.com"  # hardcoded in code
API_KEY = "sk_live_abc123"                # secret in source
```

Good:
```python
import os
DB_HOST = os.environ["DB_HOST"]   # injected at runtime
API_KEY = os.environ["API_KEY"]   # from secrets manager → env var
```

Config should differ between environments; code should not.

## Sources
- [[solution-arch/sources/clean-architecture]]
- [[solution-arch/sources/designing-data-intensive-applications]]
