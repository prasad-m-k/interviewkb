# System Design: Continuous Model Deployment & Progressive Rollout for Production Understanding Models

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/rate-limiting]]
**Patterns:** [[solution-arch/patterns/blue-green-canary]], [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

---

## Step 1: Requirements

**Functional:**
- Register a new model version (artifact + metadata: training data snapshot, eval metrics, owner)
- Run a new version in **shadow mode**: score live traffic in parallel with production, log results, affect nothing user-visible
- Canary the version to a small traffic percentage with automated quality gates before wider rollout
- Roll forward automatically when gates pass; roll back automatically when they fail — no required human-in-the-loop for the common case
- Support A/B and interleaving experiments for ranking-style models, distinct from a safety canary (an experiment measures a metric difference; a canary decides go/no-go)
- Expose "what version is live where" as a queryable fleet state, for on-call and for debugging a bad output

**Non-Functional:**
- Rollout must not breach the platform's existing serving SLOs at any point during the process — a canary is not allowed to be "a little bit down"
- Rollback must complete within a bounded time window (e.g. minutes, not the deploy pipeline's normal cadence) once a gate fails
- Fleet-wide model-version consistency: no indefinite split-brain state where some replicas silently keep serving a version rollback was supposed to remove
- Gate evaluation itself must be fast enough not to become the rollout bottleneck at fleet scale (thousands of serving replicas)
- Auditability: every rollout decision (promote/rollback) must be traceable to the specific gate signal that triggered it

---

## Step 2: Why This Differs From a Generic Blue-Green/Canary Rollout

[[solution-arch/patterns/blue-green-canary]] already covers the general deployment mechanics — traffic splitting, health-check-gated promotion, instant rollback via traffic shift. A model rollout reuses all of that infrastructure but adds a layer the generic pattern doesn't have: **the model can be "healthy" by every infra signal (200s, normal latency, normal CPU) and still be wrong.** A bad model doesn't crash — it quietly returns worse embeddings, worse rankings, or subtly shifted classifications. The rollout system therefore needs a second gate class beyond error-rate/latency: a **quality gate**.

```
Generic canary gate:     error rate, latency, CPU/memory  →  infra health
Model rollout adds:      quality-metric delta vs. baseline →  output health

Both must pass. Infra health alone is necessary but not sufficient.
```

---

## Step 3: Pipeline Stages

```
1. Register version
   → artifact stored, eval metrics attached, owner recorded

2. Shadow mode (no user impact)
   → duplicate live requests to the new version in parallel
   → log predictions, do NOT return them to the caller
   → compare shadow output distribution against production baseline
   → gate: output-distribution divergence below threshold → proceed

3. Canary (small % of real traffic, real user impact)
   → traffic split, e.g. 1% → 5% → 25% → 100%, each step time-boxed
   → gate at each step: error rate, latency (infra) AND
     quality-metric delta (output health) both within bounds
   → any gate failure at any step → automatic rollback to prior version

4. Full rollout
   → 100% traffic, previous version kept warm for N minutes for fast
     rollback, then decommissioned
```

---

## Step 4: Quality Gates — Detecting a "Healthy but Wrong" Model

```
Signals used, since ground-truth labels aren't available at serving time:

  Embedding/output-distribution drift
    compare shadow-version output distribution to production's
    (e.g. KL divergence on a score histogram, or embedding-norm drift)

  Downstream proxy metrics
    click-through / engagement on canary traffic vs. control
    (works for ranking-style models; slower signal, needs a
    time-boxed canary window long enough to accumulate it)

  Known-answer regression suite
    a held-out set of inputs with expected outputs/ranges,
    scored on every candidate version before shadow even starts

A rollback trigger fires on EITHER an infra-health breach OR a
quality-gate breach — whichever fires first wins.
```

---

## Step 5: Fleet-Wide Consistency During Rollout/Rollback

```
Problem: at fleet scale, "roll back to v1" is not instantaneous —
         propagating a version change across thousands of serving
         replicas takes time, and a slow or partitioned replica can
         be stuck on a version well after the fleet controller
         believes rollback is complete.

Fix: rollout state is the source of truth, not replica-reported state.
     - Controller tracks target version + rollout state
       (using the same distributed-consensus mechanics fleet
       config systems already rely on — see
       [[solution-arch/concepts/distributed-consensus]])
     - Replicas poll/pull their target version rather than being
       pushed to synchronously
     - Rollback completion is defined by "no replica reports serving
       the bad version," not by "rollback command issued" — and that
       completion signal is what the rollback-time SLO measures against
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Gate types | Both infra health and quality-metric delta | Infra-only gates miss "healthy but wrong" model failures entirely |
| Canary steps | Multiple time-boxed stages (1%→5%→25%→100%) | A single-step canary either under-samples rare failure modes or over-exposes users if it fails late |
| Rollback trigger | Automatic on gate failure, not human-gated | Bounded rollback-time SLO is incompatible with waiting for a human page-and-decide loop |
| Version propagation | Pull-based (replica polls target state) | Push-based fan-out to thousands of replicas has no natural "rollback complete" signal; pull gives one |
| Shadow before canary | Always, for any model with real user impact | Shadow mode catches gross output regressions with zero user exposure before spending canary budget |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #ml-rollout #canary #model-deployment #system-design #nalsd*
