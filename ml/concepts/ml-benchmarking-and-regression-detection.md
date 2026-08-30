# ML Benchmarking & Regression Detection

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/gpu-performance-engineering]], [[ml/concepts/ml-compilers-and-runtimes]], [[solution-arch/topics/cost-architecture-finops]]
**Tags:** #ResponsibleAI

## What it is

The discipline of measuring model/hardware performance reproducibly and catching regressions before they reach production, and — at the more senior end — the automated system that does this continuously rather than as a one-off exercise. This is the direct target of the JD language "build automation and observability that detect regressions" and the Principal-level ask to turn "one-off analyses into scalable platform capabilities."

## How it works

### Benchmarking methodology — why a single run is not a benchmark

```
A meaningful benchmark pins EVERY variable that affects the number:
  - Exact model + version
  - Batch size, sequence length, precision (FP16/BF16/FP8)
  - Hardware SKU, driver version, firmware version
  - Warm-up runs excluded (first call pays compile/cache-fill cost
    — see [[ml/concepts/ml-compilers-and-runtimes]])
  - Statistical treatment: run N times, report median/p90, not a
    single sample — GPU timing has real variance from thermal
    throttling, co-tenant noisy neighbors, and clock boost behavior
```

A number without all of the above pinned and disclosed is not comparable across runs, and "comparable across runs" is the entire point of a benchmark — an isolated fast number proves nothing if the next person can't reproduce the conditions that produced it.

### Metrics that matter, and where each is defined elsewhere in this wiki

```
Throughput      → tokens/sec, samples/sec — see the batching/
                   continuous-batching discussion in
                   [[ml/scenarios/llm-service-design]]
Latency          → TTFT, p50/p99 decomposition, also in
                   [[ml/scenarios/llm-service-design]]
Utilization      → achieved vs peak FLOPs/bandwidth — the roofline
                   framing in [[ml/concepts/gpu-performance-engineering]]
Cost              → cost per unit of work (per 1K tokens, per
                   training step) — ties to
                   [[solution-arch/topics/cost-architecture-finops]]
```

A benchmarking system's job is to make all four visible per change, not just whichever one happened to be the target of a given optimization — a kernel that improves throughput but regresses p99 latency (e.g., by increasing batch size) is not an unambiguous win, and a report that only shows throughput hides that trade-off from whoever reads it next.

### Regression detection — the automation half

```
Continuous benchmarking in CI:
  Every candidate change (model code, framework version, compiler
  version, driver/firmware update) runs the pinned benchmark suite
  and compares against a rolling baseline.

The hard part is the threshold, not the measurement:
  A FIXED percentage threshold (e.g., "fail if >5% slower") either
  misses real regressions in low-variance benchmarks or false-
  positives constantly on high-variance ones. A statistically-aware
  threshold — comparing the new distribution against the baseline's
  own historical variance, not a flat percentage — is what
  separates a benchmarking script from a production regression gate.

Bisection tooling:
  When a regression IS confirmed, localizing WHICH commit caused it
  across a large stack (model code → framework → compiler →
  driver → firmware) is its own engineering problem — automated
  bisection (binary search over the commit range, re-running the
  pinned benchmark at each candidate) turns a multi-day manual hunt
  into an automated one.
```

### One-off analysis vs durable platform capability

This is the distinction the JD draws explicitly between Senior and Principal scope. A Senior engineer profiling one slow kernel and fixing it delivers a point-in-time answer. A Principal engineer building the always-on benchmarking harness, the regression-alerting pipeline, and the bisection tooling ensures the NEXT fifty regressions like it get caught automatically, without anyone needing to notice by accident in production first. "Establish common measurement, automation, observability, and engineering mechanisms that turn one-off analyses into scalable platform capabilities" (direct JD language) is this idea stated as a responsibility.

## Complexity

Not algorithmic. The engineering cost is the harness/CI infrastructure; the harder cost is statistical — designing a regression threshold that has an acceptably low false-positive rate without missing real regressions, which is the same precision/recall trade-off reasoning applied elsewhere in this wiki to guardrail classifiers ([[solution-arch/concepts/ai-guardrails-and-safety]]), now applied to a performance-regression detector instead of a content classifier.

## When to use

```
- Any performance-critical platform, but especially GPU AI
  workloads: run-to-run variance is high and regressions are easy
  to introduce silently (a "correct" kernel that's 2× slower
  doesn't fail a normal correctness test suite)
- Any team supporting multiple hardware vendors (NVIDIA, AMD,
  in-house silicon): the SAME benchmark suite must run per-vendor,
  and a regression on one vendor but not another is a real, common
  failure mode worth explicitly designing detection for
- Model onboarding: a new model isn't "production ready" until its
  benchmark numbers are captured as the new baseline other changes
  will be measured against
```

## Common interview angles

```
Q: "A model's inference latency regressed 15% after a routine
    dependency bump, and nobody caught it for two weeks. How do you
    redesign the process so this doesn't happen again?"
A: The absence of a continuous benchmarking gate in CI is the root
   cause, not the dependency bump itself — dependency bumps will
   always happen. The fix is a pinned benchmark suite that runs on
   every merge (or at minimum every dependency update) with a
   statistically-aware regression threshold that blocks or alerts
   automatically, so detection doesn't depend on someone noticing
   in production.

Q: "How do you avoid false-positive regression alerts given natural
    GPU timing variance?"
A: Don't use a fixed percentage threshold — compare the new
   measurement's distribution against the baseline's own historical
   variance (e.g., is the new median outside N standard deviations
   of the baseline's rolling distribution, not just >X% different
   from a single prior number). Also control what you can: pin
   clocks/power state where possible, run enough repetitions to get
   a stable median, and isolate from noisy-neighbor co-tenancy if
   the infrastructure allows it.

Q: "What's the difference in scope between a Senior and a Principal
    engineer solving the same 'model is slow' problem?"
A: A Senior engineer profiles it, finds the fix, ships the fix —
   the JD's own framing. A Principal engineer asks why the
   regression wasn't caught before it shipped, and builds the
   measurement/automation/observability mechanism that catches the
   next one automatically — turning a single fix into a durable
   platform capability, per the JD's explicit Principal-level
   responsibility language.
```

## Examples

```
Minimal CI benchmark gate (conceptual):
  on: [pull_request]
  steps:
    - pin: model=llama-70b, batch=8, seq_len=2048, precision=bf16,
           hardware=A100-80GB, driver=<pinned version>
    - warm_up: 5 iterations (discarded)
    - measure: 20 iterations → report median, p90
    - compare: against rolling 30-day baseline distribution
    - gate: fail if median falls outside baseline's historical
      variance band (not a flat % threshold)
    - on_failure: auto-trigger bisection over the commit range
```

## Sources
- [[ml/concepts/gpu-performance-engineering]]
- [[ml/concepts/ml-compilers-and-runtimes]]
- [[ml/scenarios/llm-service-design]]
- [[ml/companies/microsoft-ai-frameworks]]
