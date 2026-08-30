---
tags:
  - observability
  - ai
  - aiops
  - interview-prep
---

# AI for Observability (AIOps)

**Topic:** [[obs/overview]]
**Related:** [[obs/concepts/anomaly-detection]], [[obs/concepts/automated-rca-correlation]], [[obs/concepts/llm-observability-copilots]], [[sre/concepts/rca-basics]]

Survey of how AI/ML is applied TO observability systems — not to be confused with observability FOR AI systems (LLM tracing, evals — see [[solution-arch/concepts/llm-observability-and-evals]]), which is the opposite direction and a completely separate interview topic. This page exists specifically to keep those two straight, because interviewers routinely ask "what do you know about AI in observability" expecting the AIOps direction, and a candidate who answers with LLM tracing/evals has answered the wrong question.

---

## The Two Directions — Say This First

```
AI FOR observability (AIOps) — THIS PAGE
  Using ML/AI techniques to make observability itself better:
  detect anomalies automatically, correlate signals across metrics/
  logs/traces to suggest a root cause, cluster/summarize noisy
  logs, reduce alert noise, answer questions in natural language.
  → The observability SYSTEM gets smarter.

Observability FOR AI (LLM observability & evals)
  See [[solution-arch/concepts/llm-observability-and-evals]].
  Tracing prompts/tool calls/outputs, evaluating LLM/agent quality,
  because traditional observability (metrics/logs/traces built for
  deterministic code) doesn't capture what a non-deterministic
  model actually did or whether it did it well.
  → The thing BEING observed happens to be an AI system.

An observability platform team can own BOTH simultaneously — they
are not mutually exclusive — but they are answers to two different
questions, and naming the distinction explicitly is itself a strong
interview signal.
```

---

## Why "Traditional" Observability Runs Out of Road at Scale

Every technique below exists because a specific traditional approach breaks down past a certain scale or complexity:

```
Static thresholds don't scale
  "Alert if CPU > 80%" works for one service. At thousands of
  services with different normal baselines (some are bursty, some
  are flat, some have daily/weekly seasonality), maintaining
  per-service static thresholds by hand becomes its own full-time
  job — and every threshold is stale the moment traffic patterns
  shift. See [[obs/concepts/anomaly-detection]].

Manual correlation doesn't scale
  A human staring at 40 dashboards trying to spot which metric
  moved first, across which service, correlated with which deploy,
  works for a small system. At hundreds of services it's the
  single biggest driver of slow time-to-root-cause. See
  [[obs/concepts/automated-rca-correlation]].

Reading raw logs doesn't scale
  Millions of log lines per minute, mostly near-duplicates with
  different IDs/timestamps — a human can't visually scan this.
  Needs clustering/summarization before a human is useful again.
  See [[obs/concepts/automated-rca-correlation]] and
  [[obs/concepts/llm-observability-copilots]].

Writing PromQL/LogQL by hand doesn't scale across an org
  Every engineer investigating an incident needs fluency in the
  query language of whichever backend is in play — natural-language
  query interfaces lower that bar. See
  [[obs/concepts/llm-observability-copilots]].
```

## The AIOps Taxonomy

```
1. Anomaly detection
   Flag "this metric/log pattern is behaving unusually" without a
   human pre-defining a static threshold. See
   [[obs/concepts/anomaly-detection]].

2. Automated correlation / RCA assistance
   Given an anomaly, automatically narrow down WHICH other signals
   (metrics, logs, recent deploys, topology-adjacent services)
   moved at the same time, to suggest a likely root cause faster
   than a human manually cross-referencing dashboards. See
   [[obs/concepts/automated-rca-correlation]].

3. Log clustering & summarization
   Group near-duplicate log lines into templates ("mine" the
   underlying pattern from noisy raw text), and summarize a burst
   of logs into a human-readable sentence. A sub-technique that
   feeds both anomaly detection (a NEW cluster appearing is itself
   a signal) and RCA (a summarized log narrative is faster to read
   than 10,000 raw lines).

4. Intelligent alerting / noise reduction
   Predict which alerts are likely to be actionable vs. noise
   based on historical outcomes, or automatically group/deduplicate
   alerts that stem from the same underlying event — a machine-
   learning layer on top of the rule-based grouping already covered
   in [[obs/patterns/alert-routing]].

5. Natural-language query & incident copilots
   Let an engineer ask "why is checkout latency up" in plain
   English and get back a grounded answer (ideally with the
   underlying queries/evidence shown, not just an assertion). See
   [[obs/concepts/llm-observability-copilots]] — including why
   "grounded" is doing a lot of work in that sentence.

6. Predictive / capacity forecasting
   Forecast resource exhaustion (disk, connection pool, queue
   depth) before it becomes an incident, using time-series
   forecasting rather than a fixed "you have N days left at current
   rate" linear extrapolation. A narrower, more mature use case —
   often the first AIOps capability a platform ships, because
   time-series forecasting is well-understood ML, unlike open-ended
   RCA correlation.
```

## Where AI Genuinely Helps vs. Where It's Overhyped

```
Genuinely valuable, production-proven:
  ✅ Anomaly detection on seasonal time series (better than static
     thresholds, well-understood ML, low risk if wrong — a false
     anomaly alert is just noise, not a wrong action taken)
  ✅ Log clustering to compress volume before a human reads anything
  ✅ Alert deduplication/grouping (clear win, low risk)
  ✅ NL-to-query translation AS A DRAFT a human reviews before
     running, not as an autonomous action

Overhyped / needs heavy guardrails:
  ⚠️ Autonomous RCA presented as definitive fact rather than a
     ranked hypothesis — see the hallucination risk in
     [[obs/concepts/llm-observability-copilots]]; an LLM confidently
     naming the WRONG root cause during an active incident actively
     wastes responder time and erodes trust in the tool
  ⚠️ Fully autonomous remediation triggered by an ML anomaly signal
     alone, with no human approval gate — the same human-in-the-loop
     reasoning as [[solution-arch/patterns/human-in-the-loop]]
     applies here: high-blast-radius actions (auto-scaling is
     usually fine; auto-rollback of a deploy is usually fine;
     auto-restarting a stateful service is NOT usually fine)
     need a risk-tiered approval gate, not blanket autonomy
```

**Interview framing:** when asked "would you let an AI system automatically remediate an incident," the strong answer distinguishes by blast radius and reversibility — the same risk-tiering logic used everywhere else in this wiki for agentic AI (see [[solution-arch/patterns/human-in-the-loop]]), not a blanket yes or no.

## Sources
- [[obs/concepts/anomaly-detection]]
- [[obs/concepts/automated-rca-correlation]]
- [[obs/concepts/llm-observability-copilots]]
- [[solution-arch/concepts/llm-observability-and-evals]] — the opposite-direction topic (observability FOR AI, not AI FOR observability)
- [[sre/concepts/rca-basics]] — the manual RCA methodology this whole page is trying to help scale
