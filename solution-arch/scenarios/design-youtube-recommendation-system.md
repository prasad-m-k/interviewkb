# System Design: YouTube's Recommendation System

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/message-queues]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/cqrs-event-sourcing]]

This page covers the **systems/infrastructure view**: data pipelines, serving infra, SLAs, scaling, reliability. For the ML model architecture itself (two-tower retrieval, ranking model, embeddings, cold-start handling, evaluation metrics), see [[ml/scenarios/recommendation-system-design]] — that page owns the algorithmic depth; this page composes it into a production system.

This is the ranking/serving subsystem, distinct from [[solution-arch/scenarios/design-youtube-video-pipeline-reliability]] (upload → transcode → CDN delivery of the raw video catalog this page's recommendations serve results from).

---

## Step 1: Requirements

**Functional:**
- Serve personalized recommendations on 3 surfaces: homepage feed, watch-next/autoplay, search-adjacent
- Incorporate near-real-time signals — a video watched 30 seconds ago should influence the next recommendation in the same session
- Cold-start handling for new users (no history) and new videos (no engagement data yet)
- Exclude already-watched content; enforce diversity (no 5 consecutive videos from the same channel/genre)
- Respect content policy filters: age-restricted, demonetized, region-blocked, creator opt-outs
- Support online A/B testing of ranking model versions against the same traffic
- Explainability hooks ("Because you watched X") for product surfaces that need them

**Non-Functional:**
- Scale: ~100M+ DAU, catalog in the billions of videos → homepage request must select from an effectively unbounded candidate space
- Latency: end-to-end homepage load (retrieval + ranking + re-ranking) P99 < 200ms
- Freshness: new video recommendable within minutes of upload; feature store reflects a watch event within seconds, not the next batch cycle
- Availability: 99.99%; must **degrade gracefully** — if the ranking service is down, serve cached or trending results rather than fail the page
- Throughput: peak QPS in the millions across all surfaces combined; retrieval/ranking funnel exists specifically to keep the expensive (ranking) stage's per-request item count small
- Cost: serving cost per request must stay low at billions of requests/day — this is why a two-stage funnel (cheap broad retrieval → expensive narrow ranking) is load-bearing, not optional
- Consistency: eventual consistency is fine for embeddings/feature store; nothing here needs strong consistency
- Observability: score-distribution drift, catalog coverage (what % of videos ever get shown), feedback-loop/filter-bubble detection

---

## Step 2: High-Level Architecture

```
                        ┌─────────────────────────────────────────┐
                        │              Event Sources                │
                        │  Watch events, likes, skips, searches     │
                        └──────────────────┬──────────────────────┘
                                           │ publish
                                  ┌────────▼─────────┐
                                  │   Kafka / Pub-Sub  │
                                  └────────┬─────────┘
                     ┌──────────────────────┼──────────────────────┐
                     ▼                      ▼                      ▼
          ┌─────────────────┐   ┌─────────────────────┐  ┌──────────────────┐
          │ Online Feature   │   │  Data Lake            │  │  Trending/Counter  │
          │ Store (Redis)    │   │ (offline training set) │  │  Service (Redis)   │
          │ session signals  │   └──────────┬───────────┘  └────────┬──────────┘
          └────────┬────────┘              │                      │
                   │              ┌─────────▼──────────┐           │
                   │              │  Offline Training    │           │
                   │              │  Pipeline (batch)     │           │
                   │              │  → two-tower + ranker  │           │
                   │              └─────────┬──────────┘           │
                   │                        │ publish model          │
                   │                        ▼                      │
                   │              ┌──────────────────────┐         │
                   │              │  Embedding Index (ANN) │         │
                   │              │  ScaNN / FAISS / HNSW  │         │
                   │              └──────────┬───────────┘         │
                   ▼                        ▼                      ▼
          ┌──────────────────────────────────────────────────────────┐
          │                     Recommendation Service                  │
          │  Stage 1: Retrieval (ANN + collaborative + trending)        │
          │  Stage 2: Ranking (neural ranker, feature-store lookup)     │
          │  Stage 3: Re-rank/filter (diversity, policy, already-seen)  │
          └────────────────────────┬────────────────────────────────┘
                                   ▼
                             Client (homepage)
```

The online/offline split is the core infra decision: embeddings and the ranking model are trained **offline** on a data-lake snapshot and pushed as immutable artifacts to the serving layer; only session-level signals (last few watches, current query) are looked up **online** at request time from a low-latency feature store. This keeps the request-path hot path free of any training-time computation.

---

## Step 3: Serving Path Latency Budget

```
Total budget: 200ms P99

  Feature store lookup (session signals)      ~10ms
  Stage 1: ANN retrieval (→ ~500 candidates)  ~40ms   (parallel across shards)
  Stage 2: Ranking (500 → top 20)             ~80ms   (batched inference)
  Stage 3: Re-rank / policy filter            ~20ms
  Network + serialization overhead            ~50ms
```

Retrieval and ranking are sharded independently — retrieval fans out across ANN index shards (index doesn't fit on one host at billions of items), ranking runs as a batched inference call so per-item cost amortizes.

---

## Step 4: Handling Freshness — the Real-Time Feature Problem

```
Problem: a video watched 30 seconds ago should shift the NEXT recommendation.
         Batch-retrained embeddings update daily — too slow for this.

Fix: split "identity" (who is this user, slow-changing) from
     "session state" (what did they just do, fast-changing).

  Slow-changing → precomputed user embedding (updated in nightly batch)
  Fast-changing → session feature vector (last N watches) looked up live
                   from the online feature store at request time

  Ranking model consumes BOTH: batch user embedding + live session vector.
```

This is the same pattern as the retrieval/ranking split applied to *time*: don't retrain the whole model for a signal that decays in minutes — inject it at serving time instead.

---

## Step 5: Cold Start (Systems View)

New videos have zero engagement signal, so they can't win an ANN similarity search against watched-history embeddings. The system routes around this at the retrieval stage rather than trying to fix it in the model:

```
Retrieval candidate sources (run in parallel, results merged):
  1. ANN similarity search (personalized)     — needs engagement history
  2. Collaborative filter (personalized)      — needs engagement history
  3. Trending/fresh-content pool (unpersonalized) — always available
  4. Creator-subscription feed (unpersonalized)   — always available

New videos are injected into (3) and get exploration traffic
until enough engagement accrues for (1)/(2) to surface them.
```

---

## Step 6: Failure Modes and Graceful Degradation

| Failure | Fallback |
|---|---|
| Ranking service down | Serve Stage-1 retrieval results unranked, or fall back to trending pool |
| ANN index shard down | Skip that shard; merge remaining shards' candidates (reduced recall, not an outage) |
| Feature store unavailable | Serve with batch-only user embedding; drop session personalization for that request |
| Offline training pipeline stalled | Continue serving last-known-good model artifact; alert, don't block serving |

The guiding rule: **no single-stage failure should take the homepage down** — every stage has a cheaper fallback one level below it.

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Retrieval index | ANN (ScaNN/FAISS) over exact kNN | Billions of items make exact search infeasible at this latency budget |
| Model update cadence | Nightly batch for embeddings, live lookup for session state | Full retrain too slow for freshness; live-only too expensive to serve at scale |
| Ranking granularity | Two-stage funnel, not rank-all-candidates | Narrows the expensive stage from billions to ~500 candidates |
| A/B testing | Shadow + gradual rollout per model version | Ranking changes affect engagement metrics broadly — needs careful staged rollout |
| Degradation strategy | Fallback pool at every stage | Availability requirement (99.99%) rules out hard-failing on any single component |

## Sources
- [[ml/scenarios/recommendation-system-design]] — model architecture, feature design, evaluation metrics
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #recommendation-systems #youtube #video #system-design*
