# System Design: Large-Scale Data Ingestion, Labeling & Training-Data Pipeline

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/responsible-ai-sdlc-governance]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/bulkhead]]

> Interview-prep scenario for a Semantic Understanding Platform-style NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This page is the **upstream, offline** counterpart to [[solution-arch/scenarios/design-semantic-signal-feature-platform]] (the online signal-serving path). They are deliberately separate systems: this pipeline produces labeled training data that eventually becomes a model artifact; that page serves live inference traffic. The whole point of the design below is that a failure here must never be able to reach there.

---

## Step 1: Requirements

**Functional:**
- Ingest raw media/text at petabyte scale from many upstream sources (product logs, crawled/licensed content, user-contributed content where permitted)
- Route ingested items to human and/or automated labeling with a quality-control gate before anything is marked "labeled"
- Version and snapshot labeled datasets so a training run can always reproduce exactly what it trained on
- Support incremental updates — new labeled data flows in continuously, not only via periodic full-dataset rebuilds
- Redact or otherwise handle PII per compliance policy before raw data reaches human labelers or lands in a durable training set

**Non-Functional:**
- **Labeling quality bar:** measurable and enforced — e.g. an inter-annotator agreement (IAA) threshold below which a labeled item is routed for adjudication, plus ongoing spot-audit sampling (not exhaustive re-review) of already-approved labels
- **Throughput vs. scale:** the pipeline must sustain petabyte-scale ingest without an unbounded backlog; labeling capacity (human or automated) is the natural bottleneck and must be explicitly modeled and provisioned for
- **Isolation from production serving:** a bad ingestion batch, a labeling-quality regression, or a schema change in this pipeline must be structurally unable to affect live serving traffic — see Step 4
- **Compliance/retention SLAs:** PII handling and data retention windows are policy-driven, auditable, and enforced by the pipeline itself, not by manual process

---

## Step 2: High-Level Architecture

```
Raw sources (product logs, licensed/crawled content, ...)
  |
  v
Ingestion + PII redaction/scrub  (before anything downstream sees raw PII)
  |
  v
Object storage (raw, versioned)  -->  Data Lake (petabyte-scale, partitioned)
  |
  v
Labeling router
  |-- automated labeling (heuristics/weak models, high-confidence cases)
  |-- human labeling queue (low-confidence / high-value cases)
        |
        v
   Quality control gate (inter-annotator agreement check,
                          spot-audit sampling of approved labels)
        |
        v
Versioned, labeled dataset snapshot  --(pull, one-way)-->  Offline training
                                                             pipeline
```

The arrow from the labeled dataset to offline training is deliberately one-way and pull-based — the training pipeline reads a versioned snapshot when it wants to, and nothing in this pipeline pushes anything toward live serving infrastructure.

---

## Step 3: Quality Control Without 100% Review

```
Problem: petabyte-scale, continuous labeling makes exhaustive human
         review of every label economically and operationally infeasible.

Fix, layered:
  1. Inter-annotator agreement (IAA): route a sample of items to
     2+ independent labelers; disagreement above a threshold routes
     to adjudication (a senior/expert labeler resolves it).
  2. Automated pre-labeling + human review-of-disagreement only,
     for high-volume/low-ambiguity categories — human effort spent
     where the automated labeler is least confident, not uniformly.
  3. Post-hoc spot-audit sampling: a small, statistically sized random
     sample of ALREADY-APPROVED labels is periodically re-reviewed.
     A rising error rate in the audit sample is the trigger to widen
     review scope or roll back a batch — not a per-item guarantee,
     a population-level quality signal.
```

---

## Step 4: Keeping Pipeline Issues From Cascading Into Serving

```
This is the operational-reliability crux of the prompt: a bad
ingestion batch or a labeling-quality regression must not become
a production incident.

Structural isolation, not just process discipline:
  - Separate infrastructure and failure domain from the serving path
    ([[solution-arch/scenarios/design-semantic-signal-feature-platform]]
    lives in a different service, different on-call, different SLOs)
  - Training only ever consumes an immutable, versioned dataset
    SNAPSHOT — a bad labeling batch corrupts a future snapshot
    candidate, not data already in use
  - A new model trained on a bad snapshot is caught by the SAME
    canary/shadow-evaluation gate any model goes through before
    reaching production (see
    [[solution-arch/scenarios/design-continuous-model-deployment-rollout]]) —
    the training-data pipeline is one more input to that gate,
    not a bypass around it
  - Schema changes to the labeled dataset are versioned and
    backward-compatible by default; a breaking schema change requires
    an explicit migration, not an implicit one that could silently
    corrupt an in-flight training run
```

The guiding principle: nothing produced by this pipeline reaches production serving directly. It always passes through the same rollout/canary gate as any other model change, which is what actually contains the blast radius — not a promise that this pipeline itself never has a bad day.

---

## Step 5: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Review coverage | Sampled IAA + spot-audit, not exhaustive | Exhaustive human review doesn't scale to petabyte-scale continuous ingestion |
| Dataset consumption model | Pull-based, versioned snapshots | Removes any push path from this pipeline into production, and makes every training run reproducible |
| PII handling | Redact/scrub at ingestion, before labeling | Keeps PII out of the labeling workforce's view entirely, simplifying compliance |
| Schema evolution | Versioned, backward-compatible by default | Prevents a schema change from silently corrupting an in-flight training run |
| Blast-radius containment | Route every trained model through the standard canary gate | The pipeline doesn't need to be perfect — the downstream gate is the actual safety net |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #data-pipeline #ml-training #responsible-ai #system-design #nalsd*
