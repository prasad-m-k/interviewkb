# System Design: Low-Latency Multi-Region Image Understanding Service (Lens-style)

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/network-architecture-fundamentals]], [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/cap-theorem]]
**Patterns:** [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/hot-path-first-design]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This page is the **low-latency read path** for one product surface (Lens: point a camera, get an answer in under half a second). It is a *consumer* of the signal generation/storage system described in [[solution-arch/scenarios/design-semantic-signal-feature-platform]] — that page owns ingest, versioning, and durable storage of understanding signals; this page owns the interactive request/response path that must hit a hard latency budget.

---

## Step 1: Requirements

**Functional:**
- Accept an image (or camera frame) as input and return semantic feature vector(s) and/or labels (objects, text, landmarks) within the latency budget
- Serve multiple downstream consumers with different needs from the same underlying capability: Lens visual search (interactive, single image), Photos auto-tagging (background, batch-tolerant)
- Support versioned models — a client can pin a model version during a controlled rollout, or take the default "current" version
- Return a confidence/quality signal alongside features so callers can decide whether to trust a degraded (fallback-model) response

**Non-Functional:**
- **Latency:** P99 end-to-end < 300–500ms for the interactive (Lens) path; Photos batch path has no hard per-item latency requirement
- **Availability:** 99.99%+ for the interactive path; a region failure must not surface as a user-visible outage
- **Multi-region:** requests served from the nearest healthy region; failover to a secondary region must complete within the latency budget, not after it
- **Cost:** GPU/TPU inference is the dominant cost driver at Google-scale QPS — the design must minimize the fraction of requests that hit the most expensive model tier
- **Freshness:** model artifacts roll out region-by-region; a request must never be served by a model version older than N days without an operator override

---

## Step 2: Latency Budget (P99 ~350ms)

```
Client upload (image, compressed)             ~60ms   (network, varies by client bandwidth)
Edge routing to nearest healthy region         ~20ms
Preprocessing (decode, resize, normalize)      ~15ms
Cache lookup (perceptual hash → cached result) ~5ms    (cache hit short-circuits everything below)
Model inference (see Step 3 for tier routing)  ~150-220ms
Postprocessing + response serialization        ~10ms
Network back to client                         ~60ms
```

The single biggest lever on this budget is the cache-hit rate in step 4 — a hit skips the entire inference cost. Everything else is optimized around minimizing how often that lookup misses.

---

## Step 3: Model Serving Tiers (CPU / GPU / TPU)

```
Request arrives → cheap classifier decides which tier to route to:

  Tier 0 (cache):  perceptual-hash match on a recent/popular image → return cached
                    features directly, no model call. Highest hit rate on popular
                    landmarks, products, common objects.

  Tier 1 (CPU, small model): fast, low-cost, lower-accuracy feature extraction.
                    Used for simple/common cases (text-heavy images, barcodes)
                    where a lightweight model's accuracy is already sufficient.

  Tier 2 (GPU/TPU, large multimodal model): full-accuracy inference. Used when
                    Tier 0/1 don't produce a sufficiently confident result, or
                    the request explicitly needs high accuracy (e.g. product
                    identification for shopping).

Router picks the cheapest tier expected to meet the confidence bar for this
request type; escalates to the next tier only on a low-confidence result.
```

This tiered-escalation design is what keeps average cost per request low without capping worst-case accuracy — most requests resolve at Tier 0/1; only the genuinely hard cases pay for Tier 2.

---

## Step 4: Caching Strategy — What's Cacheable and What Isn't

```
Cacheable:  image → features, keyed by a perceptual hash (not exact byte hash,
            since the same landmark/product photographed slightly differently
            should still hit). High reuse for popular queries (landmarks,
            common products, viral images).

Not cacheable: personal photos (Photos-surface images are private and
            effectively unique — no cross-user cache reuse is possible or
            appropriate), and any image where personalized context changes
            the answer (e.g. "what is this, given my location/language").

Cache tier: distributed cache (see [[solution-arch/concepts/distributed-caching]]
            for consistent-hashing/replication mechanics) sized for the
            popular-query working set; TTL'd and invalidated when a model
            version rolls forward (cached features are tied to the model
            version that produced them).
```

---

## Step 5: Multi-Region Consistency vs. Availability, and Failure Handling

```
This is a read-heavy, cache-friendly workload with no cross-region write
consistency requirement — each region can operate independently. The
CAP-theorem trade-off here (see [[solution-arch/concepts/cap-theorem]]) is
resolved firmly toward AVAILABILITY: a region doesn't need to agree with
its peers on cache contents or in-flight requests, only on which model
version is "current" (an eventually-consistent config push is fine —
serving a model that's a few minutes stale is acceptable; serving nothing
is not).

Region failure:
  Health-checked failover routes new requests to the next-nearest healthy
  region. In-flight requests to the failing region are retried once at
  the new region (idempotent — same image, same model version, same
  result), not resumed.

Overload (traffic spike, not a region failure):
  1. Shed the LOWEST-value work first: Photos batch/background jobs are
     paused before Lens interactive traffic is touched — enforced via
     [[solution-arch/patterns/bulkhead]] resource pools so batch work can't
     starve the interactive pool even under contention.
  2. If interactive load itself exceeds capacity, degrade rather than fail:
     route more requests to the cheaper Tier 1 model instead of Tier 2,
     trading some accuracy for staying within the latency budget.
  3. [[solution-arch/patterns/circuit-breaker]] around the Tier 2 model call —
     if Tier 2 latency degrades, the breaker opens and traffic falls back
     to Tier 1 rather than queuing behind a slow dependency.
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Consistency model | Available over consistent (per-region autonomy) | No cross-region write path on this workload; forcing consistency would only add latency for no benefit |
| Cost control | Tiered model escalation (cache → CPU → GPU/TPU) | Keeps average cost low without capping worst-case accuracy |
| Overload response | Shed batch work before interactive; degrade model tier before failing | Interactive latency SLO is the thing users actually feel; batch work has slack |
| Cache key | Perceptual hash, not exact hash | Maximizes hit rate on near-duplicate popular queries |
| Personal photos | Never cached | Privacy — no legitimate cross-user reuse exists for this data |
| Region failover | Idempotent retry at new region, not request resumption | Simpler correctness story; the request is cheap enough to redo |

## Sources
- [[solution-arch/scenarios/design-semantic-signal-feature-platform]] — the signal generation/storage system this service's inference results ultimately feed
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #lens #image-understanding #multi-region #system-design #nalsd*
