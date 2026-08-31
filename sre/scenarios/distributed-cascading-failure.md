# Troubleshooting: Distributed Cascading Failure (Fleet-Wide Latency Spike)

**Topic:** [[sre/topics/system-design]]
**Pattern:** [[sre/patterns/troubleshooting-framework]] (RED model, applied across a service graph instead of one service)
**Related:** [[sre/concepts/slo-sli-sla]], [[sre/concepts/load-balancers]], [[sre/problems/distributed-rate-limiter]]
**Companies:** [[sre/companies/google]]
**Level:** L6 — single-host tools (`top`, `strace`) don't apply here; the skill being tested is reasoning about a *system*, not a *machine*.

## Scenario
Five services deep in a call chain (edge → gateway → service A → service B → database), error rates and p99 latency spike simultaneously across *all* of them at once, not just the one closest to the database. No deploy happened in the last 24 hours. Diagnose the propagation, not just the symptom.

## Step 1: Establish direction of causation, not just correlation
The first move is not "which service is worst" — every service in a chain looks bad once one is. Pull per-service RED metrics ([[sre/patterns/troubleshooting-framework]]) and find which service's **error rate or latency rose first**, using timestamps precise to the second, plus distributed traces (Dapper/OpenTelemetry) to see where request latency actually accumulates hop-by-hop. The service that degraded *first* is the root; everything upstream of it degrading afterward is a symptom of the same underlying failure, propagating backward through the call chain.

## Step 2: Recognize the retry-storm / thundering-herd signature
The classic cascading-failure shape: service B gets slow (real cause: a downstream DB query regression, a GC pause, whatever). Service A's client has a retry policy with no backoff and no cap. Every A instance now retries every failed/slow call to B, multiplying B's incoming load several-fold at the exact moment B is least able to handle it. B's queue depth explodes, more calls time out, A retries those too. The gateway sees A's queue back up and starts its own retries. Within seconds, a slowdown in one leaf service becomes a fleet-wide outage.

Diagnostic tell: **request rate at B is higher than request rate at the gateway** — proof that retries are amplifying load somewhere in the middle of the chain, not that real user traffic increased.

## Step 3: Check whether backpressure controls existed at all
Ask, in order:
- Did A have a **circuit breaker** on calls to B? If it had tripped, A would fail fast instead of piling up threads waiting on a slow B — absence of one is often the actual root cause, not the original slowdown in B.
- Did A's retries have **exponential backoff with jitter**? Fixed-interval retries synchronize across instances and produce retry waves; jittered backoff spreads them out.
- Was there a **load shedding** policy at the edge (drop/reject low-priority requests once queue depth crosses a threshold) so the system degrades gracefully instead of falling over uniformly?

## Step 4: Mitigate before you finish root-causing
This is the part interviewers specifically probe for at L6 — do you know how to stop the bleeding while still investigating:
- Manually trip a circuit breaker or scale down the retry policy to stop the amplification loop, independent of fixing B's original slowness.
- Shed load at the edge (reject a percentage of requests) to bring B's queue back under control — a controlled, visible degradation beats an uncontrolled outage.
- Roll back the *last change anywhere in the chain*, even a config change, not just the last code deploy — retry policies and timeout values are configuration, not code, and are a common place for this kind of regression to originate.

## Step 5: Postmortem framing
Root cause is rarely "service B was slow" — that's a trigger. The root cause is usually "the system had no backpressure control between A and B, so a normal, survivable slowdown in B became a fleet-wide cascading failure." Action items should target the missing circuit breaker/backoff/load-shedding, not just "fix B's query." See [[sre/concepts/rca-basics]] for root cause vs. contributing factor vs. trigger.

## Why this is an L6-level question
It requires reading a *system*, not a host: correlating metrics across five services, recognizing a retry-amplification signature from rate deltas rather than a single dashboard, and proposing an architectural fix (backpressure) rather than a component-level one (restart B). This is also why Google's Round B (systems design) and Round D (troubleshooting) overlap in practice — the failure mode and the design pattern that prevents it are two sides of the same question.

## Sources
- [[sre/companies/google]]
- [[sre/problems/distributed-rate-limiter]]
