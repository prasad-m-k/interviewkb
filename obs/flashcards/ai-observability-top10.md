# AI for Observability (AIOps) — Top 10 Flashcards

**Topic:** [[obs/topics/ai-for-observability]]
**Format:** Question → Key Points → Depth pointer

---

## 1. What's the difference between "AI for observability" and "observability for AI"?

**Key points:**
- **AI for observability (AIOps):** using ML/AI to make the observability system itself smarter — anomaly detection, automated RCA correlation, log summarization, NL query.
- **Observability for AI:** tracing/evaluating an AI system's own behavior (prompts, tool calls, outputs) because it's non-deterministic — a completely different topic.
- Interviewers ask "what do you know about AI in observability" expecting AIOps; answering with LLM tracing/evals is answering the wrong question.

**Depth:** [[obs/topics/ai-for-observability]]

---

## 2. Why do static alert thresholds fail at scale, and what replaces them?

**Key points:**
- One fixed threshold can't be simultaneously right for low-traffic (too loose) and high-traffic (too tight) periods, and per-service manual thresholds don't scale past a handful of services.
- Replace with a time-aware baseline: STL decomposition (Observed = Trend + Seasonal + Residual, alert on the residual) or ML-based anomaly detection for multi-dimensional signals.

**Depth:** [[obs/concepts/anomaly-detection]]

---

## 3. Why is Isolation Forest a common choice for anomaly detection, specifically because of the labeled-data problem?

**Key points:**
- Unsupervised — doesn't require labeled anomaly examples, only a corpus of mostly-normal data.
- Real incidents are always a scarce minority class by definition (severe class imbalance), so a supervised approach is starved of positive examples.
- Anomalies get isolated in fewer random splits than normal points — that's the detection signal.

**Depth:** [[obs/concepts/anomaly-detection]], [[ml/concepts/class-imbalance]]

---

## 4. Your team's new ML anomaly detector is noisier than the old static thresholds. What's the likely fix?

**Key points:**
- The detector is tuned toward recall (catch everything unusual) without enough weight on precision (only page on what matters) — the same precision/recall trade-off as any classifier.
- Fix: retune the operating point toward precision for PAGING alerts specifically; route low-confidence anomalies to a dashboard annotation instead of a page.

**Depth:** [[obs/concepts/anomaly-detection]], [[ml/concepts/precision-recall-auc]]

---

## 5. How does topology-aware correlation reduce false correlations at scale?

**Key points:**
- Naive "what else moved at the same timestamp" produces coincidental matches among thousands of metrics.
- Constraining correlation to services connected in the dependency graph (built from trace data) turns "everything that moved" into "everything that could plausibly have caused this."
- Deploy/change-event correlation is the single highest-value, lowest-risk automated correlation in practice.

**Depth:** [[obs/concepts/automated-rca-correlation]]

---

## 6. Why does log clustering matter for automated RCA, not just for storage cost?

**Key points:**
- Millions of raw near-duplicate log lines (differing only in IDs/timestamps) are computationally and cognitively useless until clustered into templates (e.g. via the Drain algorithm).
- Clustering turns "logs" into a manageable signal a correlation engine or a human can actually reason about — a NEW cluster appearing, or an existing one spiking, is itself an anomaly signal.

**Depth:** [[obs/concepts/automated-rca-correlation]]

---

## 7. An automated RCA tool named a wrong root cause during an incident. What went wrong with the tool, not the incident?

**Key points:**
- It most likely presented a CORRELATION as a CAUSAL claim without a confidence signal, or didn't validate causal direction — a downstream symptom correlated with, but not caused by, the real root cause.
- Fix: surface ranked hypotheses with confidence/evidence, never a single asserted answer; keep a human validating causal direction.

**Depth:** [[obs/concepts/automated-rca-correlation]]

---

## 8. Why does an observability copilot's hallucination risk need stricter guardrails than a typical customer-support chatbot?

**Key points:**
- The cost asymmetry and time pressure are both higher — a wrong root-cause claim acted on during a live SEV-1 can actively slow mitigation, not just annoy a user.
- Required guardrails: always cite the underlying query/data, use confidence framing ("the data suggests," not "the cause is"), never let it autonomously take remediation action without human approval.

**Depth:** [[obs/concepts/llm-observability-copilots]], [[solution-arch/patterns/human-in-the-loop]]

---

## 9. How would you evaluate an observability copilot's RCA-suggestion quality before rolling it out?

**Key points:**
- Build an eval set from PAST resolved incidents with known root causes; feed the copilot only the telemetry available at the time; check whether its suggested hypothesis matches the confirmed root cause.
- Track as a precision metric over time — the same eval-gated-deploy discipline as any other production LLM feature, not a one-time demo check.

**Depth:** [[obs/concepts/llm-observability-copilots]], [[solution-arch/concepts/llm-observability-and-evals]]

---

## 10. Would you let an AI system autonomously remediate an incident it detected?

**Key points:**
- Distinguish by blast radius and reversibility, not a blanket yes/no — the same risk-tiered approval-gate logic as agentic AI generally.
- Auto-scaling: usually fine, low blast radius, reversible. Auto-rollback of a recent deploy: usually fine, reversible, well-scoped. Auto-restarting a stateful service, or any action based on a single unconfirmed anomaly signal: needs a human approval gate.

**Depth:** [[obs/topics/ai-for-observability]], [[solution-arch/patterns/human-in-the-loop]]

---

## Sources
- [[obs/topics/ai-for-observability]]
- [[obs/concepts/anomaly-detection]]
- [[obs/concepts/automated-rca-correlation]]
- [[obs/concepts/llm-observability-copilots]]
