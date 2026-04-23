# Google SRE Book — Observability Chapters

**Type:** Book (selected chapters)
**Source:** [[sre/sources/google-sre-book]]
**Ingested:** 2026-04-22
**Topics covered:** [[obs/topics/metrics]], [[obs/topics/alerting]], [[obs/concepts/slo-burn-rate]]

## Relevant Chapters

**Chapter 6: Monitoring Distributed Systems**
Defines the four golden signals (Latency, Traffic, Errors, Saturation). Establishes "alert on symptoms not causes." Introduces the philosophy that every alert must be actionable and every page must require human judgment.

**Chapter 10: Practical Alerting from Time-Series Data**
Describes Borgmon (Google's internal monitoring → predecessor to Prometheus). Aggregation rules (now recording rules in Prometheus). The concept that monitoring code should be treated as production code.

## Key Takeaways for Observability Interviews

- **Why p99, not average:** "A slowdown which applies to only 1% of users can still affect thousands per hour. The 99th percentile is the experience of the worst-off 1%."
- **Every alert = question:** "The question to ask of every alert is: is this actionable?"
- **Symptom vs. cause distinction:** Monitor symptoms (high latency, high error rate); use causes (high CPU, queue depth) for dashboard context, not paging alerts.
- **The four golden signals are sufficient:** "If you can only measure four metrics of your user-facing system, focus on these four."

## What it Updated
- Informed [[obs/patterns/four-golden-signals]]
- Informed [[obs/topics/alerting]]
- Informed [[obs/concepts/slo-burn-rate]]
