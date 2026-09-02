# System Design: Global Rate Limiting, Admission Control & Load Shedding

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/event-driven-architecture]]

> Interview-prep scenario answering two related Google NALSD prompts (a Semantic Understanding Platform-specific fairness question and a general high-QPS-service-hardening question) — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This scenario answers two closely related NALSD prompts with one design: (a) "design a fair, global rate-limiting / admission-control / load-shedding system for a shared understanding platform" — protecting a multi-tenant ML platform and its most critical customers (e.g. Search or Lens) under overload; and (b) "design a globally distributed rate limiter / admission-control / load-shedding system, or harden an existing high-QPS service against overload" — the general infra version of the same problem. The mechanics are identical; only the tenant framing differs, so one page covers both. This is the companion piece to [[solution-arch/scenarios/design-multitenant-ml-serving-platform]] (SLO isolation between tenants at the serving layer) — that page is about resource isolation, this one is about what happens the moment demand exceeds capacity anyway.

For the underlying algorithm mechanics (token bucket, leaky bucket, sliding window), see [[solution-arch/concepts/rate-limiting]] — this page is the Google-scale, multi-region systems-design application of those algorithms, not a re-derivation of them.

---

## Step 1: Requirements

**Functional:**
- Classify incoming requests into priority tiers (e.g. critical — Search/Lens interactive traffic; standard — Photos; best-effort — internal batch/offline jobs)
- Admit or shed requests to keep the service within capacity, per tenant and in aggregate
- Signal backpressure to callers explicitly (not a silent drop) so well-behaved clients can back off and retry
- Allow a critical tenant's traffic to preempt best-effort tenants under contention, without one misbehaving tenant starving another
- Support both a per-tenant quota (fairness) and a global capacity ceiling (protect the shared platform itself)

**Non-Functional:**
- **Fairness:** a protected tenant is never starved by a lower-priority tenant, even during a traffic spike from that lower-priority tenant
- **Graceful degradation under region loss:** losing one region must shed load in priority order, not fail uniformly across all tenants
- **Low added latency:** the admission-control check itself must add single-digit milliseconds, not become the bottleneck it's protecting against
- **Correctness under approximate global state:** counters that are eventually-consistent across regions must not let aggregate admitted traffic run meaningfully over the intended global ceiling
- **Availability:** the rate limiter itself must be more available than the service it protects — a rate limiter outage must fail toward "protect the backend" (shed), not "let everything through"

---

## Step 2: High-Level Architecture

```
Region A                    Region B                    Region C
  callers                     callers                     callers
    |                           |                           |
    v                           v                           v
 local admission ctrl      local admission ctrl      local admission ctrl
 (per-tenant token bucket, (per-tenant token bucket, (per-tenant token bucket,
  local capacity slice)     local capacity slice)     local capacity slice)
    |                           |                           |
    v                           v                           v
  backend fleet (Region A)   backend fleet (Region B)   backend fleet (Region C)

Cross-region: async, low-frequency capacity-share updates
  (each region reports its local admit/reject rate; a global
  coordinator periodically rebalances per-region capacity slices
  — NOT a synchronous cross-region check on every request)
```

Each region enforces its own slice of the global limit locally, using in-memory counters, and only exchanges aggregate usage numbers with other regions asynchronously and infrequently. No request is ever gated on a cross-region round trip.

---

## Step 3: Why Not Exact Global Coordination

```
Naive design: every request checks a single global counter
  before being admitted.

Problem: that counter must be consistent across regions, which means
  either (a) one region owns it and every other region pays a
  cross-region round trip per request — adds 50-150ms and makes the
  rate limiter's own availability the ceiling for the whole platform,
  or (b) a consensus protocol (Paxos/Raft) coordinates writes to it,
  which is built for correctness under contention, not for the
  sub-millisecond latency a rate-limiting check needs.

Fix: give up exact global correctness. Each region gets a
  capacity slice (e.g. total_limit / num_regions, adjusted by
  observed regional traffic share) and enforces it LOCALLY with
  no per-request cross-region call. Global totals can briefly
  overshoot the nominal limit by a bounded amount — that's an
  accepted trade-off, not a bug.
```

This is the same CAP-style trade-off as [[solution-arch/concepts/cap-theorem]] applied to a control-plane counter instead of application data: a rate limiter chooses availability and latency over perfect global consistency, because the cost of being wrong (briefly over-admitting) is far lower than the cost of being slow (defeating the purpose of the limiter).

---

## Step 4: Priority Classes and Load-Shedding Order

```
Priority tiers (highest to lowest):
  1. Critical  — Search, Lens interactive path        (never shed while any
                                                         capacity remains)
  2. Standard  — Photos, other product-facing traffic  (shed second)
  3. Best-effort — internal batch, offline recompute,  (shed first, and
                    non-time-sensitive backfills        cheapest to retry)

Shedding order under overload:
  shed best-effort first → shed standard next → critical last

[[solution-arch/patterns/bulkhead]] gives each tier its own resource
pool (thread pool / connection pool / capacity slice) so a flood
of best-effort traffic physically cannot consume the pool reserved
for critical traffic, even before the load-shedding logic engages.
```

---

## Step 5: Signaling Backpressure to Callers

```
A shed request must never look like a silent failure or timeout —
that just causes the caller to retry blindly and make the overload worse.

Response on shed:
  HTTP 429 Too Many Requests
  Retry-After: <seconds>            ← tells caller how long to back off
  X-RateLimit-Tier: best-effort     ← tells caller WHY, for its own logic

Well-behaved internal callers implement exponential backoff + jitter
on 429/Retry-After. This is the caller-side half of the contract —
[[solution-arch/patterns/circuit-breaker]] is what a caller uses to stop
hammering a service that's shedding it, so the two patterns are
complementary: admission control is the callee protecting itself,
the circuit breaker is the caller protecting the callee (and itself)
from a service it has learned is unhealthy.
```

---

## Step 6: Capacity Math Under Region Failure

```
4-region deployment, each region provisioned for 25% of global
peak load plus some headroom. Region fails:

  Remaining regions must each absorb load = failed_region_share / 3
  If regions were provisioned with e.g. 10% headroom over their own
  25% share (so ~27.5% capacity each), the extra ~8.3% of global
  load per remaining region (25% / 3) EXCEEDS that headroom
  → automatic load-shedding of best-effort/standard tiers kicks in
    for the remaining regions until traffic is rebalanced or the
    failed region recovers.

3-region deployment (each ~33% of load): losing one region means
  remaining two must each absorb an extra 16.5% — a larger relative
  jump than the 4-region case. Fewer, bigger regions make each
  single-region failure a proportionally larger shock — this is
  the standard argument for more, smaller regions over few, large
  ones, traded off against per-region fixed operating cost.
```

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Global coordination | Async, infrequent capacity-share rebalancing, not per-request | A synchronous global counter check would make cross-region latency and the coordinator's own availability the bottleneck |
| Shedding granularity | Tiered (critical/standard/best-effort), not uniform | Uniform shedding treats a Search outage the same as dropping a backfill job — unacceptable |
| Backpressure signal | Explicit 429 + Retry-After | Silent drops or timeouts cause blind retries that worsen overload |
| Resource isolation | Bulkhead per tier | Prevents a best-effort flood from physically starving critical traffic even before shedding logic runs |
| Consistency of global limit | Eventually consistent, bounded overshoot accepted | Exact consistency isn't worth the latency and availability cost for this use case |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #rate-limiting #load-shedding #multi-tenant #system-design #nalsd*
