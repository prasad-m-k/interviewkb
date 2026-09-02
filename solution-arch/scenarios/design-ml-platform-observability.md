# System Design: Observability, Monitoring & Alerting for a Multi-Stage ML Understanding Platform

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/llm-observability-and-evals]], [[solution-arch/topics/nfr-quality-attributes]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

**Scope note:** this page is about observability for a general **multi-stage ML-serving platform** — per-stage SLOs, silent quality-regression detection, cross-stage tracing, and alert design at scale. It is deliberately distinct from [[solution-arch/concepts/llm-observability-and-evals]], which covers LLM-specific observability (LLM-as-judge scoring, trajectory evals for agentic systems). The techniques here (proxy-signal drift detection, SLO burn-rate alerting) generalize to any multi-stage understanding pipeline — image, text, video — not just LLM applications; cross-link both ways where a reader needs the LLM-specific variant.

---

## Step 1: Requirements

**Functional:**
- Define and track per-stage SLIs across a multi-stage pipeline (ingest → feature extraction → model inference → post-processing → serving), not just one end-to-end number
- Detect quality regression at a stage that returns a structurally valid, non-erroring output that is nonetheless wrong or degraded — the platform's hardest observability problem
- Trace a single request across every pipeline stage, including stages owned by different teams
- Route alerts by severity and ownership so the right on-call sees the right signal

**Non-Functional:**
- Alerting must have a low false-positive rate — noisy alerts are toil and erode on-call trust in the system, defeating the point of automated alerting
- Tracing overhead must stay a small fraction of the serving latency budget — observability must not become the thing that breaks the SLO it's meant to protect
- Observability data volume/cardinality must be bounded and cost-aware at this scale — full-fidelity logging of every field on every request across a multi-stage pipeline at Google-scale QPS is not economically viable
- Detection latency for a quality regression must be short enough to page before a bad model/pipeline change causes broad user-visible harm, not just short enough to eventually show up in a weekly report

---

## Step 2: Per-Stage SLIs, Not Just End-to-End

```
End-to-end latency/error rate alone hides WHERE a problem is and
often hides WHETHER there's a quality problem at all (a stage can
be fast and return 200 OK while silently returning garbage).

Per-stage SLI set (repeated per stage in the pipeline):
  - latency (p50/p99)
  - error rate
  - throughput / queue depth (is this stage falling behind upstream?)
  - output-quality proxy signal (stage-specific — see Step 3)

Aggregating per-stage SLIs into an end-to-end SLO, rather than only
measuring end-to-end, is what lets an on-call go straight to the
failing stage instead of bisecting a multi-team pipeline during an incident.
```

---

## Step 3: Detecting Silent Quality Regressions

```
The hard problem: at serving time there is no ground-truth label to
compare against — you can't compute "accuracy" on live traffic
directly. The system instead watches PROXY signals for drift:

  Output-distribution drift
    compare live output distribution (e.g. embedding-norm histogram,
    classification-confidence histogram) against a rolling baseline;
    alert on statistically significant divergence

  Known-answer canary requests
    periodically inject inputs with known expected output ranges
    into the live pipeline; alert if the response falls outside range
    (this catches an entire stage silently returning defaults/nulls
    dressed up as a valid response)

  Downstream engagement/feedback signal
    where available, a delayed but real ground-truth proxy
    (click-through, thumbs-down rate, correction rate) — slower to
    detect with, but closes the loop the other two signals only infer

None of these alone is sufficient — output-distribution drift can be
a legitimate traffic-mix shift, not a bug; known-answer canaries only
cover the inputs you thought to test. Running all three in combination
is what makes silent-regression detection tractable.
```

---

## Step 4: Cross-Stage Tracing at Scale

```
Request enters pipeline with trace_id
  │
  ├─▶ Stage 1 (ingest)         span: trace_id.stage1
  ├─▶ Stage 2 (feature extract) span: trace_id.stage2
  ├─▶ Stage 3 (inference)      span: trace_id.stage3
  └─▶ Stage 4 (serving)        span: trace_id.stage4

Sampling strategy (cost control):
  - 100% sampling of ERROR and SLO-breaching requests
    (you always want the trace for the cases that matter)
  - Low-rate sampling (e.g. 0.1%) of normal-path requests
    (enough for baseline latency/distribution tracking without
    paying full-fidelity cost on every request)
  - Head-based sampling decision made at ingest and propagated
    via trace context, so all stages agree on whether this
    particular request is being fully traced
```

The overhead-vs-fidelity trade-off is resolved by sampling asymmetrically: full fidelity exactly where it's needed (errors, SLO breaches), thin sampling everywhere else.

---

## Step 5: Alert Design — Symptom-Based, Not Component-Based

```
Anti-pattern: one alert per low-level metric per component
  (CPU high, queue depth high, cache hit rate low, ...)
  → alert fatigue, most of these don't mean user impact on their own

Better: alert on SLO burn rate — the rate at which the pipeline is
consuming its error budget — which is a SYMPTOM (user impact),
not a component-level cause.

  fast burn (would exhaust the budget in hours) → page immediately
  slow burn (would exhaust the budget over days)  → ticket, not a page

Low-level component metrics remain available for DEBUGGING once an
SLO-burn alert has already told you something matters — they just
aren't the thing that pages someone at 3am.
```

This burn-rate framing is the same error-budget mechanism used broadly in SRE practice; the ML-specific extension is that "error budget" here includes the quality-proxy signals from Step 3, not just hard errors.

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| SLI granularity | Per-stage, aggregated to end-to-end | Localizes incidents instantly instead of requiring cross-team bisection |
| Quality-regression detection | Combine distribution drift + known-answer canaries + delayed feedback | No single signal is sufficient alone; each has a distinct blind spot |
| Trace sampling | Asymmetric: 100% on errors/breaches, thin on normal path | Keeps tracing overhead bounded while preserving full fidelity where it matters |
| Alerting basis | SLO burn rate (symptom), not per-component metric (cause) | Cuts alert-fatigue toil; component metrics stay available for debugging, not paging |
| Regression detection latency | Sized to catch a bad rollout before broad exposure | A regression only caught in a weekly report is too slow to prevent user-visible harm |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #observability #sre #ml-platform #system-design #nalsd*
