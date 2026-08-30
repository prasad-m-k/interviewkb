# Observability Knowledge Base Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" obs/log.md | tail -10`

---

## [2026-08-19] ingest | AI for Observability (AIOps)

- User asked for interview content on "AI in Observability." Audit confirmed no AIOps-direction content existed anywhere in the vault — only the opposite-direction solution-arch/concepts/llm-observability-and-evals.md (observability FOR AI systems: tracing/evals for LLM apps). Framed the whole ingest around making that distinction explicit up front, since it's the most common way this question gets mis-answered in an interview.
- Created topic: ai-for-observability — the two-directions framing (AI for observability vs. observability for AI), why static thresholds/manual correlation/raw-log-reading don't scale, the 6-item AIOps taxonomy (anomaly detection, automated correlation/RCA, log clustering, intelligent alerting, NL query/copilots, predictive/capacity forecasting), and an explicit "where AI genuinely helps vs. overhyped" section framed around blast-radius/reversibility risk-tiering (reusing the human-in-the-loop reasoning already established for agentic AI elsewhere in this KB).
- Created concept: anomaly-detection — static-threshold failure modes, STL decomposition as the often-sufficient first step, ML methods (Isolation Forest, autoencoders, change-point detection) and why unsupervised methods matter given real incidents are always a scarce minority class, precision/recall trade-off applied specifically to paging-noise cost.
- Created concept: automated-rca-correlation — framed as a search-space-reduction problem, not a smarter-diagnosis problem; topology-aware correlation, deploy/change-point correlation as the highest-value low-risk case, log template clustering (Drain), and the correlation-vs-causation trap as the most common way automated RCA tooling loses trust. Explicitly positioned as machine assistance for the manual methodology in sre/concepts/rca-basics.md, not a replacement for it.
- Created concept: llm-observability-copilots — NL-to-query translation (with the "show the generated query" transparency principle), log/incident summarization, RAG-based incident copilots composing the enterprise RAG pattern with automated-rca-correlation output as grounding context, and a hallucination-risk section arguing this domain needs STRICTER grounding discipline than typical RAG because a wrong root-cause claim during a live SEV-1 costs response-critical minutes, not just user annoyance.
- Created flashcards: ai-observability-top10.
- Updated: index.md (1 topic, 3 concepts, 1 flashcard deck), solution-arch/concepts/llm-observability-and-evals.md (added a callout cross-linking back to this new topic so the two-directions distinction is discoverable from either page).
- Notes: deliberately did NOT touch INTERVIEW_NAVIGATOR.md — this is topic content, not a company/role path, consistent with how other topic-only ingests (e.g. the HTTP status codes/headers batch) were handled.

## [2026-04-22] update | Observability knowledge base initialized
- Created directory structure: obs/, topics/, concepts/, patterns/, scenarios/, companies/, flashcards/, sources/
- Created: index.md, log.md, overview.md
- Created topics: metrics, logging, tracing, alerting
- Created concepts: prometheus, opentelemetry, grafana, loki, tempo-jaeger, slo-burn-rate, cardinality, sampling, ebpf-observability
- Created patterns: four-golden-signals, red-use-method, alert-routing, dashboard-design
- Created scenarios: high-latency-no-errors, alert-fatigue, cardinality-explosion, missing-traces
- Created companies: google, meta, apple
- Created flashcards: obs-top15
- Created sources: google-sre-book-obs, opentelemetry-spec, prometheus-docs
- Domain: Observability (Metrics + Logs + Traces + Alerting) interview prep
- Notes: Existing devops/concepts/observability-pillars and devops/topics/observability provide a thin survey; this KB provides full depth for senior SRE/DevOps interviews.
