# Automated RCA & Signal Correlation

**Topic:** [[obs/topics/ai-for-observability]]
**Related:** [[sre/concepts/rca-basics]], [[obs/concepts/anomaly-detection]], [[obs/topics/tracing]], [[sre/concepts/escalation-management]]

## What it is

Automatically narrowing down WHICH other signals (metrics, logs, recent deploys, topology-adjacent services) moved at the same time as a detected problem, to suggest a likely root cause faster than a human manually cross-referencing dashboards. This is machine assistance for the manual methodology in [[sre/concepts/rca-basics]] — it doesn't replace 5 Whys or Fishbone reasoning, it narrows the search space those techniques then get applied to.

## How it works

### Why manual correlation is the actual bottleneck at scale

```
A human doing RCA manually is really doing a SEARCH problem:
  "Something is wrong with checkout. What else changed around the
  same time, anywhere in a system with hundreds of services?"

At small scale, a human can eyeball a handful of dashboards. At
hundreds of services, the search space is too large to eyeball —
time-to-root-cause becomes dominated by how fast a human can find
the RIGHT few signals out of thousands of candidates, not by how
hard the actual diagnosis is once those signals are in front of
them. Automated correlation's whole value proposition is shrinking
that search space, not doing the diagnosis itself.
```

### Topology-aware correlation

```
Naive correlation: "which metrics moved at the same timestamp as
the anomaly" — produces a lot of coincidental, causally unrelated
matches at scale (thousands of metrics; SOME will move together by
chance in any given minute).

Topology-aware correlation: constrain the search to services that
are ACTUALLY connected to the affected service in the dependency
graph (built from trace data — see [[obs/topics/tracing]] — or a
service catalog) — upstream callers, downstream dependencies,
shared infrastructure (same DB, same node pool, same region).
Correlating with a topology PRIOR is what turns "everything that
moved" into "everything that could have plausibly CAUSED this,"
which is a massively smaller and more useful candidate list.
```

### Change-point / deploy correlation

```
The single highest-value automated correlation in practice: cross-
referencing an anomaly's onset timestamp against a deploy/config-
change event log for that service (or its direct dependencies).

Why this works so well: [[sre/concepts/rca-basics]]'s own "trigger
event" framing already establishes that most incidents have a
concrete triggering change — automating "what changed right before
this" is automating exactly the FIRST question a manual RCA asks,
just done instantly across every service instead of one human
checking one deploy log at a time.
```

### Log clustering — compressing volume before correlation is useful

```
Raw production logs at scale are mostly near-duplicates differing
only in request IDs, timestamps, and specific values — a human (or
even a naive correlation algorithm) can't usefully process millions
of raw lines.

Log template mining (e.g. the Drain algorithm, widely used in
production log pipelines):
  Raw:      "User 4821 failed to authenticate at 10:34:02"
            "User 9102 failed to authenticate at 10:34:07"
  Template: "User <ID> failed to authenticate at <TIMESTAMP>"

Clustering logs into templates turns "10,000 raw lines" into "12
DISTINCT templates, one of which suddenly spiked from 5/min to
800/min" — a signal a correlation engine (or a human) can actually
act on. This is the same instinct as the cardinality-reduction
principle in [[obs/concepts/cardinality]], applied to unstructured
log text instead of structured metric labels.
```

### Causal graphs vs. correlation — the trap interviewers probe

```
Correlation: "these two signals moved together."
Causation: "one of these signals moving CAUSED the other to move."

An automated correlation engine surfaces CANDIDATES, ranked by
correlation strength and topology proximity — it does NOT establish
causation on its own. Presenting a correlated signal as "the root
cause" without a human validating the causal direction is the
single most common way automated RCA tooling erodes trust: a
downstream service's error rate spiking is often CORRELATED with
the actual root cause (a shared upstream dependency) without being
the CAUSE of it, and a tool that states it as fact rather than as
a ranked hypothesis will eventually be wrong in a visible,
embarrassing way.
```

## Complexity

Not algorithmic in the classic sense. The real cost is building and maintaining the topology graph accurately (a stale service dependency map produces confidently wrong correlations) and the false-positive cost of over-trusting correlation as causation — both compound at scale rather than being one-time engineering costs.

## When to use

```
✅ Large service topologies where manual dashboard-hopping is the
   actual bottleneck in time-to-root-cause, not diagnostic reasoning
   itself
✅ As a RANKING/NARROWING tool that produces a short candidate list
   for a human to then apply [[sre/concepts/rca-basics]]'s
   methodology against — not as a replacement for that methodology
✅ Deploy/change correlation specifically — this narrow slice has
   the best effort-to-value ratio and the lowest risk of a wrong
   "confident causal claim," since a deploy timestamp is an
   objective fact, not an inferred correlation

❌ As an autonomous "here is THE root cause" declaration during an
   active incident without a clear confidence signal and without a
   human validating the causal direction — see the "overhyped"
   section in [[obs/topics/ai-for-observability]]
```

## Common interview angles

```
Q: "How would you help engineers find the root cause faster across
    a system with hundreds of microservices?"
A: Frame it as a search-space reduction problem, not a smarter-
   diagnosis problem: build a topology graph from trace data, only
   correlate anomalies against topology-ADJACENT services (not the
   whole fleet), and always surface deploy/config-change timestamps
   for the affected service and its direct dependencies first,
   since that's the highest-value, lowest-risk automated
   correlation available.

Q: "An RCA tool suggested a root cause during an incident, and it
    was wrong. What went wrong with the TOOL, not the incident?"
A: Most likely the tool presented a CORRELATION as a CAUSAL claim
   without a confidence signal or without validating causal
   direction — a downstream symptom correlated with, but not
   caused by, what the tool named. The fix is presenting ranked
   hypotheses with confidence/evidence, not a single asserted
   answer, and always keeping a human in the loop to validate
   causal direction before acting on it.

Q: "Why does log clustering matter for automated RCA specifically,
    not just for storage cost?"
A: Correlation and human review both need a MANAGEABLE number of
   distinct signals to work with. Millions of raw near-duplicate
   log lines are computationally and cognitively useless until
   clustered into templates — clustering is what turns "logs" into
   a signal a correlation engine (or a human doing RCA) can
   actually reason about, the same way metric aggregation turns raw
   events into a queryable time series.
```

## Examples

```
Automated correlation output for a checkout-latency anomaly
(conceptual, ranked by confidence — presented as hypotheses, not fact):

  1. [HIGH] payment-svc deploy at 10:28 UTC, 6 min before anomaly
     onset — direct upstream dependency of checkout-svc
  2. [MED]  db-connection-pool metric for payment-svc's DB dropped
     at 10:29 UTC — topology-adjacent, temporally correlated
  3. [LOW]  unrelated-svc error rate also rose at 10:31 UTC —
     NOT topology-adjacent to checkout-svc; likely coincidental,
     shown for completeness only
```

## Sources
- [[sre/concepts/rca-basics]]
- [[obs/concepts/anomaly-detection]]
- [[obs/topics/tracing]]
- [[obs/concepts/cardinality]]
- [[obs/topics/ai-for-observability]]
