# System Design: Multi-Tenant ML Model Serving / Inference Platform

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/service-mesh]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/blue-green-canary]], [[solution-arch/patterns/database-per-service]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

Design a shared inference platform that many internal products — Lens, Photos, Search, Cloud Vertex AI, YouTube — deploy models to and call. The central tension: shared infrastructure is more efficient, but any tenant's misbehavior (traffic spike, oversized model, bad query pattern) must not degrade another tenant's SLO.

---

## Step 1: Requirements

**Functional:**
- Onboard a new tenant/model with a declared SLO (latency target, availability target) and a resource request (expected QPS, model size/accelerator type)
- Route inference requests to the correct model version for the calling tenant
- Per-tenant usage and cost visibility (who is consuming what, for chargeback and capacity planning)
- Canary and gradually roll out a new model version for one tenant without affecting any other tenant's traffic
- Support heterogeneous model shapes on shared infrastructure: small classifiers alongside large multimodal models

**Non-Functional:**
- **Isolation guarantee:** one tenant's overload must not breach another tenant's SLO — this is the platform's core promise, not a nice-to-have
- **Fairness under contention:** when aggregate demand exceeds capacity, allocate the shortfall according to declared priority/quota, not first-come-first-served
- **SLO differentiation:** Search's latency SLO is stricter than Photos' background-tagging SLO — the platform must serve both without forcing one-size-fits-all provisioning
- **Capacity headroom:** enough spare capacity to absorb the largest single tenant's traffic spike without borrowing from others' reserved capacity
- **Rollout safety:** a bad model version for tenant A must be detectable and rollback-able before it affects tenant A's SLO, and must never be visible to tenant B at all

---

## Step 2: High-Level Architecture

```
  Tenant requests (Lens, Photos, Search, Vertex AI, YouTube)
        │
        ▼
  API Gateway — authenticates tenant, attaches tenant_id + declared quota
        │
        ▼
  Admission Controller — per-tenant token bucket (see rate-limiting concept);
        rejects/queues requests beyond the tenant's provisioned quota
        │
        ▼
  Router — resolves (tenant_id, model_name) → correct model version + resource pool
        │
        ▼
  Resource Pools (bulkhead per tenant or per tenant-tier)
        │
        ├── Pool A: Search  (dedicated GPU/TPU reservation, strict SLO)
        ├── Pool B: Lens    (dedicated reservation, strict SLO)
        ├── Pool C: Photos, Vertex AI (shared reservation, relaxed SLO,
        │           can absorb queueing under contention)
        └── Shared burst pool: overflow capacity, best-effort, first pool
                    to hit its ceiling can borrow here if headroom exists
        │
        ▼
  Model serving instances (per model version, per pool)
```

---

## Step 3: Isolation — Why Bulkheads, Not Just Quotas

```
Quotas alone (a token-bucket limit per tenant) prevent a tenant from
SENDING more than its share, but don't prevent a tenant's requests from
consuming a disproportionate share of a SHARED resource pool once
admitted — e.g. tenant A's large-model requests each hold a GPU slot
for 200ms while tenant B's small-model requests only need 20ms; under a
shared pool, A can starve B's queue even while both are within quota.

Fix: dedicated resource pools ([[solution-arch/patterns/bulkhead]]) per
tenant or per tenant class, sized to each tenant's declared and
provisioned capacity. A tenant can only exhaust ITS OWN pool. The shared
burst pool exists purely as opportunistic overflow — any tenant using it
must expect best-effort treatment, and burst-pool exhaustion never
degrades a tenant's dedicated pool.
```

---

## Step 4: Fair Sharing and SLO Differentiation

```
Not all tenants get equal treatment by design — that's the point.

Priority tiers (example):
  Tier 0 (Search, Lens):  dedicated capacity, strict P99 SLO, never queued
                          behind lower tiers, first to get provisioned
                          headroom during capacity planning
  Tier 1 (Photos, YouTube background jobs): dedicated but smaller
                          reservation, relaxed SLO, can queue briefly
  Tier 2 (Vertex AI ad-hoc/experimental workloads): burst-pool only,
                          best-effort, first to be shed under overload

Weighted fair queuing inside a shared pool ensures that even within a
tier, no single tenant can monopolize the pool — each tenant's queue
gets serviced proportional to its declared weight, not its request rate.
```

---

## Step 5: Canarying Without Cross-Tenant Blast Radius

```
New model version v2 for tenant A:
  1. Deploy v2 into tenant A's OWN resource pool alongside v1 (never a
     shared pool — a bad v2 must not be reachable by any other tenant)
  2. Route 1% of tenant A's traffic to v2, rest stays on v1
  3. Compare v2 vs v1 on tenant A's own SLO metrics (latency, error rate,
     and — for a model — an output-quality proxy metric, not just
     "did it respond")
  4. Ramp 1% → 10% → 50% → 100% per [[solution-arch/patterns/blue-green-canary]],
     auto-rollback to v1 if any stage regresses tenant A's SLO
  5. Only after v2 is fully promoted for tenant A does it become eligible
     as the "current" version — other tenants pin their own versions
     independently and are never auto-migrated
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Isolation mechanism | Per-tenant/tier bulkhead pools, not shared pool + quota alone | Quota limits admission rate, not resource hold time — bulkheads are needed to stop starvation from heterogeneous request costs |
| Capacity allocation | Tiered (dedicated Tier 0/1 + shared burst) | Matches provisioning cost to actual SLO strictness instead of over-provisioning every tenant to the strictest bar |
| Rollout scope | Canary confined to the owning tenant's own pool | A bad model version must have zero blast radius outside the tenant that deployed it |
| Fairness inside a pool | Weighted fair queuing by declared tenant weight | Prevents one tenant within a shared tier from starving peers even while individually within quota |
| Overload response | Shed lowest-tier/burst-pool traffic first | Protects the SLOs that matter most (Search, Lens) at the cost of best-effort workloads |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #ml-serving #multi-tenancy #vertex-ai #system-design #nalsd*
