# DevOps — Overview

## What DevOps Is (Interview-Ready Definition)

DevOps is a **culture + practice + toolchain** that breaks the wall between development and operations by:
1. **Automating** the path from code commit to production
2. **Shifting left** — catching problems (security, quality, performance) earlier in the lifecycle
3. **Creating fast feedback loops** — deploy often, observe everything, fix fast
4. **Sharing ownership** — developers own reliability; ops teams understand code

> A common interview opener: *"What does DevOps mean to you?"*  
> Answer: culture first, then process, then tools. Candidates who lead with "it's Jenkins + Docker" miss the point.

---

## The DevOps Lifecycle

```
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor
  ↑___________________________________________________|
               Continuous Feedback Loop
```

| Phase | Key practices | Key tools |
|---|---|---|
| Plan | Agile, sprint planning, feature flags | Jira, Linear |
| Code | Trunk-based dev, code review, pre-commit hooks | Git, GitHub/GitLab |
| Build | CI, dependency management, artifact creation | GitHub Actions, Jenkins, Bazel |
| Test | Unit, integration, SAST, DAST, contract tests | pytest, Trivy, Snyk, Pact |
| Release | Semantic versioning, changelog, approval gates | Helm, semantic-release |
| Deploy | Deployment strategies, GitOps, progressive delivery | Argo CD, Spinnaker, Flux |
| Operate | IaC, config management, capacity planning | Terraform, Ansible, K8s |
| Monitor | Metrics, logs, traces, alerting, SLOs | Prometheus, Grafana, Datadog |

---

## DevOps vs SRE vs Platform Engineering

| | DevOps | SRE | Platform Engineering |
|---|---|---|---|
| **Focus** | Unify dev + ops | Reliability engineering via SLOs | Build internal developer platform |
| **Key metric** | Deployment frequency, lead time | Error budget, MTTR | Developer productivity (DORA) |
| **Owns** | CI/CD culture, automation | On-call, incident response, capacity | IDP, golden paths, self-service |
| **Google's view** | Culture | "What happens when you take a software engineer and give them ops problems" | SRE + product mindset |

---

## DORA Metrics (What Elite Teams Look Like)

The four DORA metrics are the universal benchmark for DevOps performance. Know them cold.

| Metric | Elite | High | Medium | Low |
|---|---|---|---|---|
| **Deployment Frequency** | On-demand (multiple/day) | Weekly | Monthly | < Monthly |
| **Lead Time for Changes** | < 1 hour | 1 day–1 week | 1–6 months | > 6 months |
| **Change Failure Rate** | 0–15% | 16–30% | 16–30% | 46–60% |
| **Time to Restore Service** | < 1 hour | < 1 day | 1–7 days | > 6 months |

> Interview angle: *"How do you measure DevOps success?"* → DORA metrics. Know what moves each one.

---

## The Three Ways (Gene Kim's Framework)

1. **Flow (left to right)** — optimize for fast, smooth delivery from dev to ops to customer
2. **Feedback (right to left)** — fast feedback at every stage; amplify problems to fix them fast
3. **Continual Learning** — culture of experimentation, blameless postmortems, shared knowledge

---

## Interview Strategy

### What FAANG DevOps interviews test

| Round type | What's tested |
|---|---|
| **Coding** | Scripting (Bash/Python), automation, sometimes DSA |
| **System Design** | CI/CD pipeline design, Kubernetes architecture, observability stack |
| **Scenario/Ops** | Incident response, debugging production issues, trade-off analysis |
| **Behavioral** | Postmortems, cross-team collaboration, driving change |

### High-yield topics to master (in order)

1. Kubernetes architecture + debugging (appears in ~80% of senior DevOps rounds)
2. CI/CD pipeline design end-to-end
3. Deployment strategies + rollback
4. Observability: what to instrument, how to alert, how to debug
5. Infrastructure as Code: Terraform state, drift, modules
6. Secrets management
7. GitOps model
8. Incident response framework
9. Security in the pipeline (DevSecOps basics)
10. Scenario-based questions (see `scenarios/`)

---

## Coverage Map

| Topic | Status | Key gaps |
|---|---|---|
| CI/CD | In progress | GitHub Actions deep-dive, multi-stage pipelines |
| Kubernetes | In progress | CRDs, operators, advanced networking |
| IaC | In progress | Terraform modules, Terragrunt, drift remediation |
| Observability | In progress | OpenTelemetry, distributed tracing, cardinality |
| Security | In progress | Supply chain (SLSA), OPA/Gatekeeper |
| Scenarios | In progress | Multi-region failover, database migration |

## Sources
- [[devops/index]]
