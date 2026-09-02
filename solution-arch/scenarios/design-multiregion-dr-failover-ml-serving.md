# System Design: Multi-Region Disaster Recovery & Failover for a Semantic Understanding Serving Path

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

**Scope note:** this page covers failover for a **live serving path** — detecting region degradation, shifting real traffic, RTO/RPO for the thing users are hitting right now. It is deliberately distinct from a one-time **bulk data migration** (bandwidth physics, parallel transfer strategy for moving petabytes between regions) — see the dedicated migration scenario for that problem. It also builds directly on [[solution-arch/scenarios/high-availability-platform]], the general (non-ML) version of multi-region failover; read that first for the baseline mechanics. This page calls out what changes when the thing failing over is a **model-serving path** rather than a general stateful service.

---

## Step 1: Requirements

**Functional:**
- Detect region-level degradation (not just a hard outage — elevated error rate or latency counts too) fast enough to act before it becomes user-visible at scale
- Shift traffic away from a degraded region to a healthy one within the target RTO
- Maintain model-artifact and cached-signal freshness parity across regions, so the failover target can serve **correct** results immediately, not just *any* result
- Run scheduled failover drills that exercise the real failover path, not a simulated/dry-run substitute

**Non-Functional:**
- Explicit RTO (time to restore service) and RPO (acceptable staleness of state) targets, stated as numbers, not "as fast as possible"
- Standby-region capacity headroom sized as a percentage of primary-region peak load, pre-provisioned — not autoscaled reactively during the incident itself
- Failover validated on a regular cadence (game days), with near-zero user-visible error rate during a drill — a DR plan that's only ever been read, never exercised, is not a validated plan
- Traffic-shift mechanism itself must not become a new single point of failure (e.g. a DNS/GSLB layer that is itself region-scoped)

---

## Step 2: What's Different for a Model-Serving Path vs. a General Stateful Service

```
General HA service (per high-availability-platform):
  primary state = a database — replication strategy is about
  transactional consistency and data loss bounds (RPO)

Model-serving path adds two things a general service doesn't have:

  1. Model ARTIFACT replication
     the standby region needs the exact live model version already
     loaded and warm — pulling and loading a multi-GB+ model artifact
     during the incident blows the RTO budget

  2. Accelerator capacity state (warm vs. cold)
     if serving depends on GPU/TPU capacity, "headroom" isn't just
     spare compute quota — it's spare compute quota with the model
     ALREADY loaded and warmed up. A cold accelerator that still
     needs model load + warm-up on failover is not real headroom
     for RTO purposes, even though it shows up as "available capacity"
     on a naive capacity dashboard.
```

This is the single biggest gap between a textbook HA design and an ML-serving HA design: **standby capacity that isn't warm doesn't count**, and warming it costs time that eats directly into RTO.

---

## Step 3: Replication & Freshness Strategy

```
Model artifacts:
  push new model versions to all regions as part of the SAME
  rollout pipeline that promotes them in the primary region — never
  a two-step "promote in primary, replicate to standby later" flow,
  which would leave the standby region silently stale and unable
  to serve equivalent results if failover happens mid-window

Cached signals/embeddings:
  cross-region replication is asynchronous and eventually consistent
  (per CAP — see [[solution-arch/concepts/cap-theorem]]) — this is
  acceptable for cached derived signals (a slightly stale embedding
  cache degrades quality marginally, doesn't break correctness), but
  NOT acceptable for the model artifact itself, which must be
  synchronously part of rollout, not lazily replicated
```

---

## Step 4: Traffic Shifting Mechanics

```
Detection:
  region-level health signal (aggregate error rate / latency,
  not a single instance) crosses threshold

Traffic shift, layered (fastest, coarsest first):
  1. Load-balancer-level traffic weight shift (fast, seconds —
     preferred first response if the LB layer itself is healthy)
  2. DNS/GSLB-level shift (slower, subject to client-side TTL/caching
     — a necessary fallback if the degraded region's LB tier is
     what's actually failing, but not the first lever)

Partial shift, not always all-or-nothing:
  shift traffic proportionally to the standby region's actual
  warm capacity — dumping 100% of primary's load onto a standby
  sized for 50% headroom just converts one region's outage into
  two regions' outage
```

---

## Step 5: Validating Failover Actually Works

```
The part interviewers push hardest on: a runbook that has never
been executed is a hypothesis, not a plan.

Game-day drill cadence:
  - Scheduled, regular (e.g. quarterly) full failover drills against
    production traffic, not a shadow/staging replica of it
  - Traffic-shift mechanism, capacity headroom, and model/signal
    freshness are ALL exercised together — testing them independently
    doesn't catch integration gaps (e.g. headroom that's numerically
    correct but not actually warm)
  - Success criterion: near-zero user-visible error-rate delta during
    the drill window, measured the same way a real incident would be
  - Any drill failure feeds back into capacity/replication design —
    a drill's purpose is finding the gap before an actual outage does
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Standby capacity | Pre-provisioned and kept warm, not autoscaled during incident | Autoscale-during-incident adds accelerator provisioning lead time directly onto RTO |
| Model artifact replication | Synchronous, part of the same rollout pipeline as primary | A lazily-replicated standby can be stale exactly when failover needs it most |
| Cached-signal replication | Asynchronous, eventually consistent | Acceptable quality degradation for derived signals; full synchronous replication here isn't worth the cost |
| Traffic-shift layering | LB-level first, DNS/GSLB as fallback | LB-level is faster and avoids client-side DNS caching delay |
| Validation | Regular game-day drills on real production traffic | An unexercised runbook is unverified; drills are what convert RTO from a target into a measured, trusted number |

## Sources
- [[solution-arch/scenarios/high-availability-platform]] — general multi-region HA baseline this scenario extends
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #disaster-recovery #multi-region #ml-serving #system-design #nalsd*
