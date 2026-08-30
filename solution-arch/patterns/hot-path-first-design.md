# Hot-Path-First Design

**Topic:** [[solution-arch/topics/scalability-and-reliability]], [[solution-arch/topics/nfr-quality-attributes]]
**Related concepts:** [[solution-arch/concepts/caching]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/rest-api-design-principles]], [[solution-arch/concepts/database-sharding]]
**Related patterns:** [[solution-arch/patterns/outbox]], [[solution-arch/patterns/cqrs-event-sourcing]]
**Source:** [[solution-arch/sources/hot-path-first-system-design]]

## What it solves

The default instinct in most system design interviews (and real projects) is to reach for a toolbox — "we should add Redis, read replicas, and a load balancer" — and apply it uniformly across the whole system. Hot-path-first design inverts that: **before choosing any component, find out where the system is actually hot**, and scale only that. Everything else stays simple. This reframes the entire system design exercise around one question asked first: *where is the system actually hot, and does that part need speed or correctness?*

```
Tool-collecting mindset:                Architecture-first mindset:
"We should add Redis, read              "Product browsing tolerates
 replicas, and a load balancer."          staleness, so it uses cache +
                                          replicas. Checkout cannot
 → applies uniformly, can't explain       trust stale inventory, so it
   WHY each piece is there                reads from the consistency path."

                                         → every component earns its
                                           place by naming the specific
                                           requirement it satisfies
```

## The Core Distinction: Reads Want Speed, Writes Want Correctness

```
READS                              WRITES
  Optimize for: volume, latency      Optimize for: correctness,
  Tolerate: staleness (often)        durability
  Scale via: caching, replicas       Tolerate: staleness NEVER
                                     Scale via: partitioning,
                                     careful transaction design —
                                     NOT by relaxing correctness
```

This single split — decided PER ENDPOINT, not per system — is the lens every other decision in this pattern flows through.

### Display Reads vs Decision Reads

Not all reads are equal, and conflating them is the most common design mistake this pattern targets:

```
Display reads                        Decision reads
  Product page, search results,        Checkout, inventory
  recommendations                      confirmation, payment
                                        authorization

  Tolerates eventual consistency       Requires fresh, authoritative
  (a few seconds/minutes of            data — a stale read here
  staleness is invisible/harmless      causes a real-money mistake
  to the user)                         (oversold inventory, wrong
                                        price charged)

  → served from cache + read           → served from the primary /
    replicas                             a dedicated consistency path
```

**The failure mode this prevents:** a price update commits to the primary DB. The product page (cache) still shows the old price for a few minutes — acceptable. But if checkout ALSO reads from that same stale cache instead of the primary, a customer can see one price while browsing and get charged a different amount at checkout — a trust and revenue problem, not just a UX quirk. Classifying reads correctly is what prevents this exact bug.

## Diagram 1 — Basic Read/Write Split

```
                    ┌─────────────────────┐
                    │  Application Instance │
                    └──────────┬───────────┘
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                      ▼
      READ PATH                              WRITE PATH
      Cache ──▶ Read Replicas ──▶ Primary DB   Direct ──▶ Primary DB
      (fast, tolerant of                       (correctness-first,
       staleness)                               no shortcuts)
```

## Diagram 2 — Read/Write Split at Scale (Load-Balanced)

```
                         ┌────────────────┐
                Client ──▶  Load Balancer  │
                         └────────┬────────┘
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              App Instance 1  App Instance 2  App Instance 3
                    │             │             │
          ┌─────────┴─────────────┴─────────────┴─────────┐
          ▼                                                ▼
     READ PATH (shared)                              WRITE PATH (shared)
     Cache + Read Replicas                            Primary DB
```

Every app instance follows the SAME read/write split — this is a property of the architecture, not something each instance decides independently.

## Diagram 3 — Write Path with Reliable Cache Invalidation

The hardest part of hot-path design isn't the read side — it's making sure a write reliably tells the cache it's now stale. This pattern uses the transactional [[solution-arch/patterns/outbox]] pattern, specifically applied to cache invalidation events rather than generic downstream event publishing:

```
Write Request
     │
     ▼
BEGIN TRANSACTION
  UPDATE product SET price = ...          ← the actual write
  INSERT INTO outbox (event: "ProductInvalidated", id: 123)
                                           ← SAME transaction —
                                             both succeed or both fail
COMMIT
     │
     ▼
Background worker polls/streams the outbox (see the Polling vs CDC
approaches in [[solution-arch/patterns/outbox]])
     │
     ▼
Publishes "ProductInvalidated" event
     │
     ▼
Cache consumer (subscribed to the event) deletes the stale key
     │                                         │
     ▼                                         ▼
Next read is a cache MISS,              Read replicas eventually
repopulates from primary                replicate the new value too
(now-correct) data                      (separate, slower path — the
                                         cache invalidation and the
                                         replica catch-up are two
                                         DIFFERENT staleness windows,
                                         don't assume they close at
                                         the same time)
```

**TTL is the safety net, not the primary mechanism.** Even with a reliable outbox-driven invalidation pipeline, treat it as best-effort: the cache consumer can be down, the event can be delayed, a bug can exist. A TTL on every cached entry (even a generous one — minutes, not seconds) bounds the WORST CASE staleness window if the event-driven path fails silently, so the system self-heals instead of serving stale data indefinitely. Never rely on event-driven invalidation alone with no TTL backstop.

## Diagram 4 — Full Architecture (E-Commerce Example)

```
                              ┌─────────────────┐
                    Users ───▶│  Load Balancer   │
                              └────────┬─────────┘
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
             DISPLAY READS      CRITICAL READS         WRITES
             (search, product   (checkout inventory   (product create/
              detail, category,  check, payment auth)  update, price
              recommendations)                          change, stock
                    │                  │                 adjustment)
                    ▼                  ▼                 ▼
              ┌──────────┐      ┌─────────────┐    ┌──────────────┐
              │  Cache    │      │ Consistency  │    │  Primary DB   │
              │  (miss?)  │      │ Path (reads  │    │  (source of   │
              └────┬─────┘      │ from Primary,│    │  truth)       │
                   ▼             │ or a strongly│    └──────┬───────┘
              ┌──────────┐      │ consistent   │           │
              │  Read     │      │ replica)     │           ▼
              │ Replicas  │      └─────────────┘    ┌──────────────┐
              └────┬─────┘             ▲              │  Outbox Table │
                   ▼                   │              └──────┬───────┘
              ┌──────────┐             └──────same DB─────────┘
              │ Primary   │                                   │
              │ DB (miss  │                                   ▼
              │ fallback) │                          Event Publishing
              └──────────┘                                   │
                                                               ▼
                                                     Cache Invalidation
                                                     (deletes stale keys)
```

## Diagram 5 — Endpoint Classification Matrix

The concrete artifact this pattern produces — before any infrastructure decision, classify every endpoint:

```
┌──────────────────────┬──────────┬───────────┬──────────────────┬─────────┐
│ Endpoint               │ Traffic   │ Freshness  │ Scaling strategy   │ Revenue │
├──────────────────────┼──────────┼───────────┼──────────────────┼─────────┤
│ GET /search/products    │ High      │ Stale-OK   │ Cache + replicas   │ Direct   │
│ GET /products/{id}       │ High      │ Stale-OK   │ Cache + replicas   │ Direct   │
│ GET /categories/{id}/... │ High      │ Stale-OK   │ Cache + replicas   │ Indirect │
│ GET /recommendations     │ Medium    │ Stale-OK   │ Cache + replicas   │ Indirect │
│ GET /checkout/inventory  │ Medium    │ Must-fresh │ Consistency path   │ Direct   │
│ POST /payment/authorize  │ Low       │ Must-fresh │ Primary (txn)      │ Direct   │
│ PUT  /products/{id}      │ Low       │ n/a (write)│ Primary            │ None     │
│ PATCH /products/{id}/... │ Low       │ n/a (write)│ Primary            │ None     │
│ POST /products           │ Low       │ n/a (write)│ Primary            │ None     │
└──────────────────────┴──────────┴───────────┴──────────────────┴─────────┘
```

**Why the matrix matters more than any single diagram:** this table IS the design. Everything else (which components to build, where caching goes, which reads get a consistency path) is derived mechanically from filling in these four columns honestly, using real production signals — not guessed from intuition.

## The 5-Step Methodology

```
Step 1 — Measure the hot path
  Get real production signals: request volume ranking per endpoint,
  DB query time consumed per endpoint, spike/burst behavior,
  staleness tolerance, revenue/trust sensitivity. Guessing which
  endpoints are hot without data is how systems get over- or
  under-engineered in the wrong places.

Step 2 — Classify endpoints
  Build the matrix above: traffic intensity × consistency
  requirement × assigned scaling strategy, for every endpoint that
  matters.

Step 3 — Separate access patterns
  Route display reads through the fast path (cache + replicas).
  Route decision reads through a correctness path (primary, or a
  strongly-consistent replica/consistency layer). Route writes
  through a protected, transactional path. This routing decision
  is made ONCE, architecturally — not re-decided ad hoc per request.

Step 4 — Connect invalidation
  Wire writes to cache invalidation reliably: include the
  invalidation event in the write's own transaction (outbox),
  publish asynchronously, make invalidation consumers idempotent
  (processing the same invalidation event twice must be harmless),
  and keep TTL as the safety net under all of it.

Step 5 — Avoid over-engineering
  "Scale what is hot" — not the whole system uniformly. Adding
  Redis in front of a low-traffic admin endpoint (PUT /products/{id},
  called a handful of times a day) adds operational complexity
  (cache invalidation surface, another failure mode) for zero
  measurable benefit. The same Redis layer in front of
  GET /products/{id} (called thousands of times a second) is the
  entire point of the exercise. The infrastructure decision is
  IDENTICAL in kind — the justification is what differs, and that
  justification must come from Step 1's data, not from habit.
```

## Key Trade-offs This Pattern Forces You to Name Explicitly

```
| Trade-off                          | What you're choosing between |
|-------------------------------------|-------------------------------|
| Replica lag vs read distribution     | More replicas spread read load, but widen the eventual-consistency window every display read is exposed to |
| Cache TTL vs DB pressure              | Shorter TTL shrinks the staleness window but increases cache-miss rate and DB load |
| Consistency path vs complexity        | Protecting decision reads requires extra routing logic (a second code path, not just "add a cache") |
| Invalidation failures                 | Even with an outbox, the invalidation consumer can be down — TTL bounds the damage, but silent invalidation failures need their own alerting, not just a bigger TTL |
```

## Signal phrases (recognize this pattern being asked for)

- "Design a system that's read-heavy / has a 99:1 read:write ratio"
- "How would you scale product browsing without breaking checkout correctness?"
- "Where would you actually add caching in this system, and where would you NOT?"
- "A user sees one price browsing and gets charged another at checkout — what's wrong with the architecture?"
- "How do you decide which endpoints need a cache versus which need to hit the database directly?"

## Complexity

Not algorithmic — the "cost" here is architectural discipline: the methodology requires production telemetry (Step 1) to exist before it can be applied honestly, and requires maintaining two distinct read code paths (display vs decision) rather than one, which is real ongoing complexity that must be justified by the classification data, not assumed.

## How This Complements the Standard SA Interview Framework

[[solution-arch/scenarios/interview-questions]]'s "How to Answer System Design Questions" gives the overall interview structure (clarify → estimate scale → high-level design → deep dive → trade-offs). Hot-path-first design is the substance that should fill the "high-level design" step: instead of presenting a generic three-tier diagram with a cache box glued on, present the endpoint classification matrix first, then derive the architecture from it. This is the difference an interviewer is listening for between a candidate who has memorized components and one who has an actual design methodology.

## Problems using this pattern
- [[solution-arch/scenarios/design-url-shortener]] — read-heavy redirect lookups vs infrequent short-URL creation
- [[solution-arch/scenarios/design-enterprise-rag-system]] — display-style Q&A reads vs document ingestion writes
- [[solution-arch/scenarios/high-availability-platform]] — applies the same read/write separation at the availability-tier level

## Sources
- [[solution-arch/sources/hot-path-first-system-design]]
- [[solution-arch/patterns/outbox]]
- [[solution-arch/concepts/distributed-caching]]
