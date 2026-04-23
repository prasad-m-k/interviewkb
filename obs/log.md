# Observability Knowledge Base Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" obs/log.md | tail -10`

---

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
