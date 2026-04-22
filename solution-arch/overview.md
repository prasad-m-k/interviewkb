# Solution Architecture — Overview

A Solution Architect designs the technical blueprint that satisfies a business problem within constraints (cost, time, team skill, regulatory, existing systems). The job is not to pick the "best" technology — it is to make the right **trade-offs** explicitly.

---

## The Architect's Core Loop

```
Business Problem
      │
      ▼
┌─────────────────────────────────────────┐
│  1. Understand requirements             │
│     • Functional (what the system does) │
│     • Non-functional (how well it does) │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  2. Identify constraints                │
│     • Budget, timeline, team size       │
│     • Regulatory, compliance            │
│     • Existing tech stack               │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  3. Enumerate options                   │
│     • Architectural styles              │
│     • Build vs buy vs integrate         │
│     • Trade-off matrix per option       │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  4. Select & document decision          │
│     • ADR (Architecture Decision Record)│
│     • Rationale + rejected alternatives │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  5. Design system                       │
│     • C4 diagrams (Context → Component) │
│     • Data flows, API contracts         │
│     • Security & failure modes          │
└─────────────────────────────────────────┘
```

---

## Quality Attributes (NFRs) Cheat Sheet

| Attribute | Definition | Key techniques |
|-----------|-----------|----------------|
| **Availability** | % uptime | Redundancy, health checks, failover |
| **Reliability** | Correct results under failure | Retries, idempotency, checksums |
| **Scalability** | Handle growing load | Horizontal scale, sharding, async |
| **Performance** | Latency + throughput | Caching, CDN, async, indexes |
| **Security** | Protect data + access | Encryption, AuthN/Z, zero trust |
| **Maintainability** | Ease of change | Modularity, clean interfaces, IaC |
| **Observability** | Understand system state | Logs, metrics, traces |
| **Portability** | Run on different infra | Containers, cloud-agnostic APIs |
| **Resilience** | Recover from failure | Circuit breaker, bulkhead, chaos testing |
| **Consistency** | Data agreement across nodes | CAP trade-off, consensus protocols |

---

## The "Ilities" Trade-off Reality

No system can maximise all attributes simultaneously. Classic tensions:

```
Consistency  ←──────────────▶  Availability
   (strong guarantees)           (always respond)

Performance  ←──────────────▶  Security
   (minimal overhead)             (add auth + encryption)

Simplicity   ←──────────────▶  Flexibility
   (monolith)                     (microservices)

Consistency  ←──────────────▶  Scalability
   (single DB)                    (distributed DB)
```

**The architect's job:** make these trade-offs consciously and document why.

---

## Architecture Decision Records (ADRs)

Lightweight format for capturing decisions:

```markdown
# ADR-001: Use PostgreSQL as primary datastore

## Status: Accepted

## Context
Need a relational store with strong ACID guarantees for financial transactions.
Team has existing expertise. Volume < 1M rows/day for 2 years.

## Decision
PostgreSQL with read replicas. Revisit if write throughput exceeds 10k TPS.

## Consequences
+ ACID, mature tooling, team familiarity
- Harder to scale writes beyond single primary
- Vertical scaling has ceiling

## Alternatives considered
- CockroachDB: distributed SQL but team unfamiliar
- DynamoDB: scales infinitely but no joins, eventual consistency
```

---

## Related
- [[solution-arch/topics/nfr-quality-attributes]]
- [[solution-arch/topics/architectural-styles]]
- [[solution-arch/diagrams/c4-model]]
- [[solution-arch/scenarios/interview-questions]]
