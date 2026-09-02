# System Design: High-Scale Logs + Metrics / Telemetry Pipeline

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This page is the GENERAL cross-service telemetry pipeline: collecting logs/metrics from every service in a fleet. It is not [[solution-arch/scenarios/design-ml-platform-observability]], which is the SLO/quality-regression observability layer for one specific multi-stage ML platform — that page's tracing and SLI collection would be built as a *consumer* on top of a pipeline like this one, not a replacement for it.

---

## Step 1: Requirements

**Functional:**
- Ingest heterogeneous telemetry from every service: unstructured logs, structured metrics (counters/gauges/histograms), and traces
- Process near-real-time: windowed aggregation for metrics (e.g. 10s/1m rollups), log parsing/enrichment (add service, region, version labels)
- Store with tiered retention: hot (recent, full-resolution) vs. cold (archived, downsampled)
- Serve to three consumer classes: interactive dashboards, alerting rules, and downstream batch consumers (billing, capacity planning)
- Support ad-hoc query over recent logs/metrics without a schema migration for every new label a team adds

**Non-Functional:**
- **No silent data loss** under normal load spikes or a single-zone network partition — drops must be bounded and alertable, not undetected
- Ingestion throughput headroom for the peak fan-in of the largest single incident (an incident is exactly when telemetry volume spikes hardest — the pipeline must not fall over at the moment it's needed most)
- Dashboard query latency: seconds, not minutes, for the hot tier
- **Cost bounded despite cardinality growth** — the single largest cost driver at this scale is metric cardinality, not raw event volume
- Pipeline observability: the pipeline's own drop-rate, ingestion lag, and backlog depth must be first-class exported signals

---

## Step 2: High-Level Architecture

```
Service instances (millions)
   │ logs, metrics, traces
   ▼
Local agent (buffers + batches + compresses)
   │ bounded local disk buffer — absorbs brief network blips
   ▼
Shipping tier (regional collectors)
   │ shards by service/tenant
   ▼
Ingestion tier (partitioned by time + shard key)
   │
   ├──▶ Hot store (recent, full-resolution) ──▶ Dashboards, alerting
   │
   └──▶ Stream processor (windowed aggregation)
             │
             ▼
        Cold store (downsampled, long-retention) ──▶ Batch consumers
```

Backlog accumulates in three possible places, and each needs its own bound: the **local agent buffer** (bounded disk, oldest-drop-first once full — a single host being unreachable must not become unbounded memory growth), the **shipping tier** (backpressure signaled to agents — agents slow their send rate rather than the collector falling over), and the **ingestion tier** (partitioned so one noisy shard's backlog doesn't starve others — see [[solution-arch/patterns/bulkhead]]).

---

## Step 3: Cardinality — the Cost and Correctness Problem

```
Metric: request_latency{service, region, status_code}     ← ~1,000 series, fine
Metric: request_latency{service, region, request_id}      ← millions of series, NOT fine

A metric's cost scales with the CARDINALITY of its label set
(distinct combinations of label values), not event volume.
A single high-cardinality label (request_id, user_id, raw URL)
can turn a 1,000-series metric into a 10M-series metric overnight —
this is the single most common cause of an unplanned telemetry cost
or storage blowup at scale.

Logs and traces don't have this problem the same way — they're
naturally per-event, high-cardinality, and priced/retained
accordingly (shorter retention, sampled). The fix for metrics
is an enforced label allowlist / cardinality budget per metric
at the ingestion tier, rejecting or aggregating away label values
that would blow the budget, rather than trusting every team's
instrumentation to self-police.
```

---

## Step 4: Retention Tiers and Downsampling

```
Hot tier (0–7 days):   full resolution, fast query, most expensive per byte
Warm tier (7–30 days): downsampled (1m → 5m rollups), cheaper
Cold tier (30+ days):  downsampled further (1h rollups), object storage, cheapest

Downsampling happens ONCE, in the stream processor, not per-query —
recomputing rollups at query time for a dashboard is the mistake
that makes cold-tier queries unusably slow.
```

Logs get a parallel but shorter retention curve — most incident investigation happens within days, and raw log volume dwarfs metric volume, so cold-tier log retention is typically sampled rather than fully downsampled.

---

## Step 5: Observability of the Pipeline Itself

A telemetry pipeline that silently drops data under load is strictly worse than having no pipeline — engineers trust a dashboard that's quietly wrong more than they'd trust no dashboard at all. The pipeline must export, as first-class metrics of itself: ingestion lag (time from event generation to queryable), drop rate per stage, and backlog depth per shard. These get their own alerting, independent of the alerting the pipeline serves for other services — a [[solution-arch/patterns/circuit-breaker]] between the shipping tier and a degraded ingestion shard trades completeness for the rest of the fleet's data staying current, with the drop explicitly counted and alerted rather than silently absorbed.

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Buffering location | Bounded local-disk buffer at the agent | Absorbs brief network blips without unbounded memory growth on the host |
| Backpressure | Signaled upstream (agent slows send), not silent drop | A visible slowdown is debuggable; a silent drop is not |
| Cardinality control | Enforced label budget at ingestion, not instrumentation-time trust | Self-policing at hundreds of teams doesn't scale; a budget with rejection does |
| Downsampling | Computed once in the stream processor, stored per tier | Avoids recomputing rollups on every cold-tier dashboard query |
| Pipeline's own health | First-class exported drop-rate/lag metrics, separately alerted | A pipeline that fails silently is worse than no pipeline |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #observability #telemetry #system-design #nalsd*
