# System Design: Multi-Tier Caching Platform for Tier-0 Streaming Metadata

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/caching]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/database-sharding]], [[solution-arch/topics/cost-architecture-finops]], [[solution-arch/topics/enterprise-architecture-frameworks]]

> **Role context:** this scenario is built around the kind of problem an "Online Data Stores" team owns at a large streaming company — a platform team building the in-memory and distributed caching layer that sits in front of Tier-0 services (services where an outage stops playback for every customer), such as the video **Catalog** and **CDN metadata**. The team's stated goals: extreme scale, low latency, high throughput, cost efficiency, and an opinionated abstraction layer over the underlying storage systems, built from a mix of in-house/open-source solutions and public cloud offerings. Treat this as a platform-team system design, not a feature-team one — the "customer" is every other service at the company, not an end user directly.

---

## Step 1: Requirements

**Functional:**
- Serve read-heavy metadata (catalog entries, CDN/edge routing metadata) with a simple key-based access pattern
- Provide a single abstraction layer so calling services don't need to know whether data is served from memory, a distributed cache, or the origin store
- Support both **in-memory** (single-node, sub-millisecond) and **distributed** (cross-node, cross-zone) tiers behind that abstraction
- Allow calling teams to onboard a new dataset without operating their own cache cluster

**Non-Functional (Tier-0 specific):**
- P99 read latency in the low single-digit milliseconds — this is inline in the playback-start critical path
- Survive an availability-zone failure and a region failure without a customer-visible outage (Tier-0 = "if this is down, nobody streams")
- Throughput: millions of reads/sec in aggregate across a fleet this large; must scale horizontally, not vertically
- Cost efficiency at this scale is a first-class requirement, not an afterthought — a few cents per GB-month compounds into a large line item at petabyte scale
- Build-vs-buy: some layers are in-house (and open-sourced back to the community), others are cloud-managed — this is a deliberate, per-layer decision, not all-or-nothing

---

## Step 2: Two-Tier Model — In-Memory vs Distributed

```
Tier 1 — In-process / local cache (per app instance)
  Sub-microsecond access, no network hop
  Scope: single JVM/process; lost on restart; small (fits in app heap)
  Use for: extremely hot, small working sets (e.g., feature flags,
  routing tables) where even a network hop to Tier 2 is too slow

Tier 2 — Distributed cache (cross-node, cross-zone)
  Sub-millisecond access, one network hop
  Scope: shared across the fleet; survives individual app restarts
  Use for: the bulk of Tier-0 read traffic — catalog rows, CDN
  metadata — where the working set exceeds what fits in-process

Tier 3 — Origin store (source of truth)
  Single-digit to double-digit ms access
  Only hit on a distributed-cache miss
```

This is the same hit/miss and cache-aside mechanics as [[solution-arch/concepts/caching]] — what's new at this scale is that Tier 2 is not one Redis box, it is itself a sharded, replicated cluster. See [[solution-arch/concepts/distributed-caching]] for the consistent-hashing, replication, and gutter-pool mechanics that Tier 2 depends on.

---

## Step 3: Architecture — the Abstraction Layer

```
                        Calling Service (Catalog API, CDN routing, etc.)
                                      │
                                      ▼
                    ┌─────────────────────────────────┐
                    │   Data Access Abstraction Layer  │   ◀── this is the
                    │  (secure, opinionated client SDK  │      team's core
                    │   — same interface regardless of  │      product
                    │   which storage backend is used)  │
                    └────────────────┬──────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
      Tier 1: in-process     Tier 2: distributed        Tier 3: origin
      local cache (Caffeine-  cache cluster (Memcached-  store (wide-column
      style, per instance)    based, e.g. EVCache-style   DB, e.g. Cassandra-
                               pattern; zone-aware)        style, or managed
                                                            cloud equivalent)
```

**Why an abstraction layer at all?** Two reasons this team would care about it specifically:
1. **Developer productivity** — calling teams get a single, secure, intuitive interface and never hand-roll cache-aside logic, retry policy, or serialization per team. Consistency across hundreds of internal consumers matters more than any one team's cleverness.
2. **Optionality for the platform team** — because callers depend on the abstraction, not the backend, the platform team can swap Memcached for a different engine, change sharding strategy, or migrate a dataset from self-hosted to a cloud-managed offering, without every calling service changing code. This is the same boundary-hiding rationale as the API Gateway pattern (see [[solution-arch/concepts/api-gateway]]), applied to storage instead of HTTP routing.

---

## Step 4: Zone-Aware Replication (the real-world precedent)

Netflix has publicly documented a caching system of exactly this shape — **EVCache**, an open-sourced, Memcached-based distributed cache designed for AWS multi-AZ deployment (public reference: Netflix Technology Blog, "EVCache"). The pattern is worth knowing regardless of employer, because it is the standard answer to "how do you keep a cache available across zone failures":

```
Problem: a single Memcached cluster lives in one AZ. If that AZ has
a network partition or goes down, every cache read for that data
becomes a miss — simultaneously, for the whole fleet — at exactly
the moment the origin store is also under stress from the same
event.

Zone-aware replication fix:
  - Each cache cluster is replicated to multiple AZs
  - The client library (part of the abstraction layer from Step 3)
    is AZ-aware: it prefers reading from the LOCAL AZ's replica
    (avoids cross-AZ data transfer cost and latency) but transparently
    fails over to another AZ's replica if the local one is unhealthy
  - Writes fan out to all AZ replicas (client-side fan-out, or via
    a replication proxy)

  App (AZ-a) ──▶ prefers Cache replica (AZ-a) ──HIT──▶ return
                        │ (AZ-a unhealthy)
                        ▼
                 falls back to Cache replica (AZ-b)
```

This is a direct extension of the replication-topology and cross-region-consistency material in [[solution-arch/concepts/distributed-caching]] — same idea, but the "regions" are AWS AZs, and the driver is availability (survive a zone failure) rather than latency (serve users from the nearest region).

---

## Step 5: Cost Efficiency at Extreme Scale

This is the requirement that most system-design answers skip, and exactly the one a platform/infra team is measured on. At petabyte-scale working sets and billions of daily requests, small per-GB or per-request costs compound fast.

```
Levers, roughly in order of impact:

1. Right-size the tiers — don't put everything in the
   (expensive, RAM-bound) distributed cache. Push truly hot, small
   keys to Tier 1 (in-process, free); reserve Tier 2 for what
   actually needs cross-instance sharing.

2. TTL and eviction tuning per dataset — catalog metadata that
   changes rarely can carry a much longer TTL (fewer refills) than
   CDN routing metadata that changes by the minute. One global TTL
   wastes either freshness or cache capacity.

3. Compression of cached values — trades a small amount of CPU for
   a large reduction in RAM footprint, which is usually the
   dominant cost driver for a large in-memory tier.

4. Build vs. buy per layer, not once — the in-house/open-source
   Tier 2 cache may be cheaper at this scale than an equivalent
   fully-managed cloud offering, but a smaller or newer dataset may
   be cheaper to run on a managed service until it justifies
   dedicated infra. This is the same build-vs-buy framing as
   [[solution-arch/topics/enterprise-architecture-frameworks]],
   applied per dataset rather than as a single platform-wide choice.

5. Cross-AZ / cross-region data transfer — the zone-aware routing
   in Step 4 exists partly for cost, not just availability: reading
   from a remote AZ's replica by default (instead of preferring
   local) would silently add data-transfer cost to every single
   cache read at this volume.
```

This maps directly onto [[solution-arch/topics/cost-architecture-finops]]'s FinOps framing: cost is a design input evaluated continuously, not a review gate applied after the architecture is fixed.

---

## Step 6: Failure Modes

```
Scenario: a distributed cache node fails
  → without protection: its keys become simultaneous misses,
    stampeding the origin store right as the cluster is degraded
  → fix: gutter pool absorbs the miss load (see
    [[solution-arch/concepts/distributed-caching]])

Scenario: a hot key (e.g. a trending title's catalog entry)
  → single-node overload even inside a healthy cluster
  → fix: local (Tier 1) caching of the hottest keys, plus key
    replication/sharding suffixes, as covered in
    [[solution-arch/concepts/caching]]'s "hot key" interview angle

Scenario: origin store write, cache not yet invalidated
  → Tier 2 across multiple AZs must all be invalidated, not just
    the local replica — an incomplete fan-out on invalidation
    reintroduces exactly the stale-read problem covered in
    [[solution-arch/concepts/distributed-caching]]'s cross-region
    consistency section (same failure mode, cross-AZ instead of
    cross-region)
```

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Cache topology | Two-tier (in-process + distributed) | Local tier removes network hops for the hottest keys; distributed tier gives shared, larger capacity |
| Distributed engine | Memcached-based (or equivalent) with AZ-aware client | Simplicity and raw throughput over the richer feature set of something like Redis, when the access pattern is pure key-value |
| Replication | Multi-AZ, client-preferred-local | Availability against AZ failure, without paying cross-AZ transfer cost on every read |
| Interface to callers | Opinionated abstraction/SDK, not raw client access | Centralizes correctness (retries, serialization, invalidation) and preserves the platform team's ability to change backends later |
| Cost strategy | Per-dataset TTL/tiering/build-vs-buy, not one global policy | A single blanket policy either wastes capacity on cold data or under-caches hot data |

---

## Common interview angles

- "Why not just run one big Redis cluster for everything?" — mixing datasets with very different hotness/TTL/size profiles behind one policy wastes capacity; a platform team typically offers tiers and lets each dataset owner tune within them.
- "How do you decide what belongs in Tier 1 vs Tier 2?" — working-set size and sharing need: if it must be consistent/shared across instances, it can't live only in-process.
- "This is a Tier-0 dependency — how do you avoid becoming the outage?" — zone-aware replication with local-preferred reads, gutter pools for node failure, and a fail-open policy on the abstraction layer itself (serve slightly stale or fall through to origin rather than error) so the cache is never a harder dependency than the thing it's protecting.
- "How do you justify the cost of an in-house cache platform vs. buying a managed one?" — this is a recurring senior-level framing: the in-house build amortizes across every internal consumer and can be open-sourced back (reducing external maintenance burden via community contribution), whereas a managed service is billed per team, per dataset — the crossover point is a real build-vs-buy calculation, not a default.
- "What's the case for open-sourcing this kind of internal infrastructure?" — a platform this central benefits from external adoption: bug fixes and hardening come from a wider user base than one company's traffic could ever surface alone, and it raises the team's credibility and hiring pipeline in that ecosystem.

## Sources
- [[solution-arch/concepts/caching]]
- [[solution-arch/concepts/distributed-caching]]
- [[solution-arch/topics/cost-architecture-finops]]
- Netflix Technology Blog — EVCache (public; zone-aware Memcached-based distributed cache) — used here as a real-world precedent for the replication pattern in Step 4, not as insider/proprietary detail
