# System Design: Hardening a Global Video/Image Understanding Pipeline Under Volume Growth

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/rate-limiting]]
**Patterns:** [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/event-driven-architecture]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

Scope note — this page is specifically about pipeline **throughput and reliability under volume growth** (queueing, priority, backpressure, failure recovery). It is not the video upload/transcode/CDN-delivery pipeline (see [[solution-arch/scenarios/design-youtube-video-pipeline-reliability]] for that) and not the signal storage/serving layer (see [[solution-arch/scenarios/design-semantic-signal-feature-platform]] for that) — this page is the understanding-pipeline stage that sits between raw content arriving and signals being written to that storage layer.

---

## Step 1: Requirements

**Functional:**
- Process incoming video/image volume through understanding pipelines at differentiated priority (interactive/Lens-triggered work vs. bulk Photos-library backfill)
- Support full-corpus batch reprocessing (e.g. after a model version upgrade) without disrupting online priority traffic
- Signal to downstream consumers when data is stale/lagging rather than silently serving old signals as if fresh

**Non-Functional:**
- **Throughput headroom:** sustain N-x traffic growth (e.g. 5-10x a current baseline) without redesign, via horizontal scaling of stateless pipeline stages
- **Bounded staleness:** even under backpressure, downstream consumers must know HOW stale the signals they're reading are, not just eventually receive fresh ones
- **Failure isolation:** one pipeline stage's outage or slowdown must not cascade into upstream stages backing up indefinitely or downstream consumers silently reading garbage
- **Recovery time:** after a stage failure, catch-up processing of the backlog must complete within a bounded window, prioritized by content priority

---

## Step 2: High-Level Architecture

```
  Content sources (uploads, batch reprocessing triggers)
        │
        ▼
  Ingest queue — partitioned by priority class, not a single FIFO:
        ├── Priority queue: interactive (Lens-triggered on-demand)
        ├── Priority queue: online (near-real-time, e.g. new Photos upload)
        └── Priority queue: batch (bulk backfill, model-upgrade reprocessing)
        │
        ▼
  Understanding pipeline stages (parallel workers per stage, per
  priority queue — resource-isolated via bulkhead pools so batch
  workers can't starve interactive workers even at 100% utilization)
        │
        ▼
  Stage output → Signal store (design-semantic-signal-feature-platform)
        │
        ▼
  Staleness signal published alongside each write — consumers can see
  "this content's signal is N minutes/hours old" rather than assuming
  freshness
```

---

## Step 3: Priority Isolation, Not Just Priority Ordering

```
A single priority QUEUE with different priority labels is not enough —
under sustained batch load, priority ordering alone still lets batch
work consume 100% of a shared worker pool between interactive requests
arriving, causing latency spikes for interactive traffic whenever a
batch job is mid-flight.

Fix: separate resource pools per priority class ([[solution-arch/patterns/bulkhead]]),
sized so interactive has GUARANTEED capacity independent of batch
volume. Batch workers scale into whatever capacity is left over (or a
separate autoscaled pool), never borrowing from the interactive pool's
reservation.
```

---

## Step 4: Backpressure and Bounded Staleness

```
Problem: a downstream stage (e.g. the ML inference stage) slows down or
partially fails. Without backpressure, the upstream ingest queue grows
unboundedly, and — worse — downstream CONSUMERS of the final signal
keep reading whatever was last written, with no indication that new
content isn't being processed.

Fix, two parts:
  1. Backpressure propagation: each stage exposes its queue depth/lag
     as a signal upstream. When a stage's queue exceeds a threshold,
     upstream producers slow admission for LOWER-priority work first
     (batch), preserving headroom for interactive work as long as
     possible before slowing that too.
  2. Explicit staleness metadata: every signal write carries a
     computed_at timestamp and the pipeline publishes a per-content-class
     "max processing lag" metric. A downstream consumer (e.g. Search)
     can check this lag and decide to degrade its own behavior (e.g.
     fall back to a cached ranking signal) rather than blindly trusting
     understanding output that might be hours out of date during an
     incident.
```

---

## Step 5: Failure Recovery and Batch Reprocessing Without Disruption

```
Stage failure:
  Failed work items go to a dead-letter queue with the reason for
  failure attached, not silently dropped. A recovery job retries DLQ
  items at LOW priority once the stage recovers, so recovery doesn't
  itself create a new backpressure event on top of already-degraded
  capacity.

Full-corpus reprocessing (e.g. new model version needs to re-score
every existing video):
  Injected as a distinct, lowest-priority, RATE-LIMITED batch stream
  (see [[solution-arch/concepts/rate-limiting]]) — capped to consume only
  spare capacity, explicitly measured against current interactive/online
  load, and backed off automatically if it starts to eat into the
  interactive pool's headroom. This is the same "don't let bulk work
  starve interactive work" principle as Step 3, applied over a much
  longer time horizon (a reprocessing job can run for days).
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Queueing | Separate priority-class queues + bulkhead pools, not one FIFO with priority labels | Priority labels alone don't prevent a shared pool from being monopolized by low-priority work between high-priority arrivals |
| Backpressure target | Slow lowest-priority admission first | Preserves interactive SLO for as long as possible under sustained overload |
| Staleness handling | Explicit lag metadata published to consumers | Lets consumers make an informed degrade decision instead of silently trusting stale data |
| Failure recovery | DLQ + rate-limited low-priority recovery replay | Prevents recovery itself from re-triggering the backpressure event it's recovering from |
| Full reprocessing | Rate-limited background stream, not a burst job | A model-upgrade backfill running at full speed would itself be an unplanned capacity event |

## Sources
- [[solution-arch/scenarios/design-semantic-signal-feature-platform]] — where this pipeline's output lands
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #pipeline-reliability #backpressure #semantic-understanding #system-design #nalsd*
