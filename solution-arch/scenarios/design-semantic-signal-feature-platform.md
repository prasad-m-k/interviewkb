# System Design: Semantic Signal / Feature Generation and Serving Platform

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/event-sourcing]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/cqrs-event-sourcing]], [[solution-arch/patterns/database-per-service]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This is the core system a Semantic Understanding Platform owns: ingest raw media/text, run understanding pipelines to produce signals (embeddings, labels, classifications), and store/serve those signals durably at high QPS with freshness and version guarantees. It is the **producer** side of the pipeline; [[solution-arch/scenarios/design-lens-image-understanding-service]] is one example of a low-latency **consumer** read path built on top of these signals, and [[solution-arch/scenarios/design-video-image-understanding-pipeline-reliability]] covers the throughput/reliability concerns of the ingest pipeline itself under volume growth — this page focuses on the storage/serving layer and its versioning contract.

---

## Step 1: Requirements

**Functional:**
- Ingest raw media/text at scale and run versioned understanding pipelines against it
- Store resulting signals (embeddings, labels) with an explicit schema version tied to the model version that produced them
- Serve signals via a high-QPS read API to multiple independent consumers (Lens, Search, Photos), each with different freshness/latency needs
- Support incremental/partial re-processing — e.g. only re-run understanding on content affected by a new model version, not the entire corpus

**Non-Functional:**
- **Freshness:** bounded staleness per consumer class — some consumers need a signal within seconds of the source content changing, others tolerate hours
- **Throughput:** ingest at petabyte-scale volume without falling behind production rate
- **Read latency:** P99 signal lookup in the low tens of milliseconds for high-QPS consumers
- **Durability:** no signal loss — a signal, once computed and acknowledged, must survive any single-node or single-AZ failure
- **Schema evolution:** backward-compatible versioning — a consumer pinned to schema v3 must keep working while the platform produces v4 signals for new content

---

## Step 2: High-Level Architecture

```
  Raw media/text sources
        │
        ▼
  Ingest layer (dedup, format normalization)
        │
        ▼
  Understanding pipeline (versioned models, batch + streaming — see
  design-video-image-understanding-pipeline-reliability for the
  throughput/backpressure mechanics of this stage)
        │
        ▼
  Signal write path
        ├── Signal store (durable, sharded by content ID — see
        │     database-sharding concept for shard-key mechanics)
        └── Change stream (event-sourced — every signal write emits an
              event; downstream caches and indexes subscribe to this
              stream rather than polling the store)
        │
        ▼
  Read-serving layer
        ├── Hot cache (recently-written / frequently-read signals)
        └── ANN index (for embedding similarity queries — see
              vector-databases concept)
        │
        ▼
  Consumers (Lens, Search, Photos, Vertex AI) — each reads at its own
  freshness tolerance; the change stream lets a consumer choose to read
  the store directly (freshest) or a lagging derived index (cheaper,
  slightly stale) depending on its own SLO
```

---

## Step 3: Signal Versioning and Schema Evolution

```
Every signal record carries:
  { content_id, signal_type, model_version, schema_version, payload,
    computed_at }

A new model version (e.g. embedding dimensionality changes from 512 to
768) does NOT overwrite existing signals in place. Instead:
  1. New signals are written under the new schema_version, alongside
     old ones (not replacing them)
  2. Consumers declare which schema_version(s) they can read
  3. A background backfill job re-computes signals for existing content
     under the new version, at low priority, without blocking new-content
     ingestion
  4. Once backfill coverage crosses a threshold AND all active consumers
     have migrated, old schema_version rows become eligible for cleanup

This is the same zero-downtime migration shape as the embedding-dimension
schema migration in [[solution-arch/scenarios/ai-data-platform-system-design]]
— see that page for the full backfill/dual-write mechanics; not
re-derived here.
```

---

## Step 4: Freshness Tiers

```
Not every consumer needs the same freshness, and serving everyone at the
strictest tier would be needlessly expensive:

  Tier "live" (seconds):   read directly from the signal store / change
                           stream. Used where a stale signal is visibly
                           wrong to the end user (e.g. a just-uploaded
                           photo not yet tagged).
  Tier "near-real-time" (minutes): read from a cache that's refreshed
                           off the change stream on a short interval.
                           Used for most interactive consumers.
  Tier "batch" (hours):   read from a periodically rebuilt index (e.g.
                           the ANN index, which is expensive to update
                           incrementally at full fidelity). Used for
                           bulk/offline consumers like large-scale
                           re-ranking or analytics.

Each consumer picks its tier based on its own SLO — the platform doesn't
force one uniform freshness bound onto every reader.
```

---

## Step 5: Durability and Read Scaling

```
Durability: signal writes are replicated synchronously to at least one
other AZ before being acknowledged (no signal is "computed" from the
producer's perspective until it's durable) — losing an understanding
pipeline's output would mean expensive re-computation, so this is a
harder durability bar than a typical cache.

Read scaling: the signal store is sharded by content_id (co-locating
all signal types for one piece of content on the same shard, avoiding
cross-shard joins for the common "give me everything known about this
content" read). Hot/popular content additionally sits in a distributed
cache in front of the store to absorb read QPS without hitting shard
limits.
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Schema migration | Dual-write + backfill, never in-place overwrite | Zero-downtime for existing consumers; a bad new model version doesn't destroy the previous good signal |
| Freshness | Tiered (live / near-real-time / batch) per consumer | Uniform strict freshness for every reader would be needlessly expensive; most consumers don't need it |
| Change propagation | Event stream (CQRS-style), not polling | Consumers subscribe to what changed rather than re-scanning the store; scales to many independent consumers |
| Sharding key | content_id (co-locate all signal types per item) | Avoids cross-shard joins for the dominant read pattern: "everything known about this content" |
| Durability bar | Synchronous cross-AZ replication before ack | Re-computing a lost signal is expensive (full pipeline re-run); worth the extra write cost |

## Sources
- [[solution-arch/scenarios/ai-data-platform-system-design]] — zero-downtime embedding-schema migration mechanics (referenced, not duplicated)
- [[solution-arch/concepts/vector-databases]]
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #semantic-understanding #embeddings #data-platform #system-design #nalsd*
