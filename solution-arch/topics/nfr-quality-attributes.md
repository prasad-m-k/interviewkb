# Non-Functional Requirements & Quality Attributes

**Related:** [[solution-arch/overview]], [[solution-arch/concepts/cap-theorem]], [[solution-arch/topics/scalability-and-reliability]]

NFRs define **how well** a system works — not what it does. They are often the harder and more interesting part of system design interviews. Interviewers expect you to ask about NFRs before proposing a design.

---

## The NFR Interview Checklist

Before designing anything, ask:

```
Scale
  ├── How many users / requests per second?
  ├── Read-heavy or write-heavy? (ratio)
  └── Data volume today vs. in 3 years?

Availability
  ├── What is the uptime SLA? (99.9% vs 99.99%)
  └── What are the consequences of downtime?

Consistency
  ├── Strong (bank transfer) or eventual (social feed)?
  └── Can users see stale data? For how long?

Latency
  ├── P50 / P99 target?
  └── Is it user-facing (< 200ms) or background (seconds)?

Durability
  └── If a write is acknowledged, can it ever be lost?

Security
  ├── PII / regulated data?
  └── Who needs to access what?
```

---

## Availability

Availability = uptime / (uptime + downtime)

| SLA | Downtime / year | Downtime / month |
|-----|----------------|-----------------|
| 99% ("two nines") | 87.6 hours | 7.3 hours |
| 99.9% ("three nines") | 8.7 hours | 43.8 minutes |
| 99.99% ("four nines") | 52 minutes | 4.4 minutes |
| 99.999% ("five nines") | 5.3 minutes | 26 seconds |

**Techniques:**
- Redundancy: eliminate single points of failure (SPOF)
- Health checks + auto failover
- Multi-AZ / multi-region deployment
- Graceful degradation (return cached / partial data rather than error)
- Chaos engineering: intentionally inject failures to validate resilience

**Availability in series vs parallel:**

```
Series:   A=99.9%, B=99.9%  →  99.9% × 99.9% = 99.8%
          (each adds failure probability)

Parallel: A=99.9%, B=99.9%  →  1 - (0.1% × 0.1%) ≈ 99.9999%
          (both must fail simultaneously)
```

---

## Scalability

Ability to handle increasing load.

```
Vertical Scaling (Scale Up)          Horizontal Scaling (Scale Out)
┌───────────────────────────┐        ┌────────────────────────────────────┐
│  One big server           │        │  Many smaller servers + LB         │
│  ┌──────────────────────┐ │        │  ┌──────┐  ┌──────┐  ┌──────┐    │
│  │ 64 cores, 512 GB RAM │ │        │  │ Svc  │  │ Svc  │  │ Svc  │    │
│  └──────────────────────┘ │        │  └──────┘  └──────┘  └──────┘    │
│  Ceiling exists           │        │  Theoretically unlimited           │
│  No code changes needed   │        │  Requires stateless design         │
└───────────────────────────┘        └────────────────────────────────────┘
```

**Scalability types:**
- **Stateless services:** trivial to scale horizontally (add instances)
- **Stateful services:** require session affinity or external state store
- **Database:** read replicas for reads; sharding for writes; caching reduces pressure

---

## Reliability vs Availability

- **Availability:** is the system responding?
- **Reliability:** is it giving correct results?

A system can be available (returns 200 OK) but unreliable (returns wrong data).

Techniques: checksums, idempotency, write-ahead logs, saga patterns for distributed txns.

---

## Performance

Two dimensions:
- **Latency:** time to complete one request (ms)
- **Throughput:** requests handled per second (RPS/TPS)

They can conflict: batching improves throughput at the cost of latency.

**Latency targets (rule of thumb):**

| Operation | Expected latency |
|-----------|-----------------|
| L1 cache read | ~1 ns |
| L2 cache read | ~4 ns |
| RAM read | ~100 ns |
| SSD read | ~100 µs |
| Disk read | ~1–10 ms |
| Same-DC network round trip | ~0.5 ms |
| Cross-region network | ~100–200 ms |

Design implication: a query that hits disk 10 times sequentially = ~100 ms before any network.

---

## Security

Key dimensions:
- **Authentication (AuthN):** who are you? (JWT, OAuth2, mTLS)
- **Authorization (AuthZ):** what can you do? (RBAC, ABAC, OPA)
- **Encryption in transit:** TLS 1.3 everywhere
- **Encryption at rest:** AES-256, key management (HSM, KMS)
- **Data minimisation:** only store what you need
- **Audit logging:** immutable log of who did what when

---

## Maintainability

- **Modifiability:** can you change one thing without breaking others?
- **Testability:** can you verify correctness?
- **Deployability:** can you ship fast and safely?

Key patterns: clean interfaces, bounded contexts, IaC, CI/CD, feature flags.

---

## Observability

```
Logs      → What happened? (structured, queryable events)
Metrics   → How much? How often? (counters, gauges, histograms)
Traces    → Why? (end-to-end request path across services)
Alerts    → Who to wake up? (threshold or anomaly-based)
```

The "three pillars" + alerting. Can't debug a distributed system without all three.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
