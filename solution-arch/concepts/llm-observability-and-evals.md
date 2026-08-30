# LLM Observability & Evaluation

**Topic:** [[solution-arch/topics/llm-application-architecture]], [[solution-arch/topics/agentic-ai-architecture]]
**Related:** [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/topics/cost-architecture-finops]]

> [!NOTE] Not to be confused with [[obs/topics/ai-for-observability]]
> This page is observability **FOR** AI (tracing/evaluating an AI system's own behavior). The opposite direction — using AI/ML **FOR** observability itself (anomaly detection, automated RCA, log summarization) — is [[obs/topics/ai-for-observability]]. Interviewers ask about both under similar-sounding phrasing; know which one is being asked.

## What it is

LLM observability extends the classic three pillars (logs, metrics, traces — see [[solution-arch/topics/microservices]]) with signals unique to non-deterministic model behavior: what prompt was sent, what the model actually returned, how good that output was, and how much it cost. **Evaluation** ("evals") is the discipline of scoring model/agent output against a fixed test set so that quality is measured, not assumed — the direct substitute for unit/integration tests, which don't work against non-deterministic output.

## How it works

```
┌────────────────────────────────────────────────────────────┐
│               LLM OBSERVABILITY DATA MODEL                     │
├────────────────────────────────────────────────────────────┤
│ Per request, capture:                                          │
│   - Full prompt sent (system + retrieved context + user turn)   │
│   - Model + version used                                        │
│   - Tool calls made, in order, with arguments and results         │
│   - Final output                                                  │
│   - Token counts (input/output) and $ cost                         │
│   - Latency: time-to-first-token AND total completion time          │
│   - User feedback signal if available (thumbs up/down, correction)  │
│                                                                   │
│ This is a TRACE, not a log line — the same shape as distributed    │
│ tracing in microservices (see [[solution-arch/topics/microservices]]),│
│ with each LLM/tool call as a span, and the whole agent loop as      │
│ the trace.                                                          │
└────────────────────────────────────────────────────────────┘
```

### Evaluation methods

```
1. Exact match / rule-based
   Compare output against a golden answer exactly, or check a rule
   (e.g. "output must be valid JSON matching this schema", "must
   contain a citation"). Cheap, deterministic, but only works for
   tasks with a well-defined correct output.

2. Human evaluation
   Domain experts score a sample of outputs against a rubric.
   Highest quality signal, but slow and expensive — used to
   establish ground truth and to validate that automated methods
   (below) correlate with human judgment.

3. LLM-as-judge
   A separate LLM call scores the output against a rubric ("Is this
   response accurate, relevant, and policy-compliant? Score 1-5 with
   reasoning"). Scales far better than human eval; risk: judge model
   has its own biases/blind spots, so periodically validate judge
   scores against a human-labeled sample to confirm correlation
   holds.

4. Trajectory evaluation (agentic-specific)
   For multi-step agents, score not just the final answer but
   WHETHER THE RIGHT SEQUENCE of tool calls/decisions was taken —
   a correct final answer reached via a wrong or unsafe path (e.g.
   skipped a required validation step) is a failure the outcome-only
   check would miss entirely.

5. Production A/B and canary comparison
   Compare a new prompt/model version against the current one on
   live traffic (small %), using proxy metrics: user correction
   rate, escalation-to-human rate, thumbs-down rate — see
   [[solution-arch/patterns/blue-green-canary]] applied to prompts.
```

## Complexity

Not applicable algorithmically. The architectural cost is: eval-set maintenance (a living asset that must grow as new failure modes are discovered in production — same discipline as a regression test suite), and LLM-as-judge calls add real token cost that scales with how often the pipeline runs evals.

## When to use

```
Minimum required for ANY production LLM feature:
  ✅ A fixed eval set (even 20-30 examples) covering the happy path
     + known edge cases, run before every prompt/model change ships
  ✅ Production tracing sufficient to reconstruct exactly what
     happened on any given request (required for debugging AND
     for governance/audit — see [[solution-arch/topics/ai-governance-responsible-ai]])

Scale up to full eval infrastructure (large golden sets, automated
LLM-as-judge pipelines, continuous production sampling) when:
  ✅ The system is customer-facing at meaningful volume
  ✅ Regulatory/audit requirements mandate documented testing
     (high-risk-tier use cases under frameworks like the EU AI Act)
  ✅ Multiple teams ship prompt/model changes and need a shared
     quality gate to avoid regressions from each other's changes
```

## Common interview angles

```
Q: "How do you know your LLM feature got WORSE after a change, when
    there's no compiler error and outputs still look plausible?"
A: This is exactly what a fixed eval set catches that manual
   spot-checking can't — run the same golden set through old and
   new versions, diff the scores. "Looks plausible" is precisely
   the failure mode evals exist to catch; a confidently wrong
   answer looks identical to a correct one without a ground-truth
   comparison.

Q: "LLM-as-judge scores look great, but users are unhappy — what's
    wrong?"
A: The judge's rubric may not reflect what users actually value
   (e.g. judge rewards verbosity/confidence that users find
   annoying), or the judge model shares blind spots with the
   model being evaluated (both trained similarly, miss the same
   subtle errors). Validate judge output against real human/user
   feedback periodically — don't treat automated eval scores as
   ground truth indefinitely without that check.

Q: "How would you design monitoring to catch a prompt-injection
    attack in production, after the fact?"
A: Anomaly detection on the TRACE data: unexpected tool-call
   sequences, tool calls to unusual endpoints, output containing
   patterns matching known exfiltration attempts, or a spike in a
   specific tool's usage — flagged trajectories route to human
   review and feed back into both the eval set and the guardrail
   classifier's training data (see
   [[solution-arch/concepts/ai-guardrails-and-safety]]).

Q: "What's the AI-specific equivalent of a P99 latency SLO?"
A: There isn't a single one — you need a small dashboard of AI-
   specific SLOs alongside classic latency/availability: eval pass
   rate (quality), thumbs-down/correction rate (user-perceived
   quality), cost-per-request (see
   [[solution-arch/topics/cost-architecture-finops]]), and
   escalation-to-human rate (a proxy for how often the system is
   operating outside its competence).
```

## Examples

```
Eval-gated deploy pipeline for a support-ticket triage agent:
  1. Golden set: 200 labeled tickets with correct category + priority
  2. On every prompt change: run all 200, require 95%+ exact-match
     on category, human-reviewed spot-check on the 5% misses
  3. Canary: new prompt handles 5% of live traffic for 48 hours;
     compare escalation rate against control before full rollout
  4. Production trace sampling: 100% of escalated tickets get full
     trace review to catch systematic gaps the golden set missed
```

## Sources
- [[solution-arch/topics/microservices]] — observability pillars this extends
- [[solution-arch/topics/ai-governance-responsible-ai]]
