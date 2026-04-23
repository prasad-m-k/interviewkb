# Google Observability Interview Prep

**Related:** [[obs/overview]], [[sre/companies/google]], [[obs/concepts/prometheus]]

## Google's Internal Observability Stack

Google does not use Prometheus or Grafana internally (those are open-source tools they've influenced but don't use). Understanding Google's proprietary systems is valuable for:
1. Interview discussions about "how would you design X at Google scale?"
2. Understanding the conceptual models that influenced Prometheus, OpenTelemetry, and the SRE book.

### Monarch (Metrics)
Google's production metrics system — the successor to Borgmon.

**Key design principles:**
- **In-memory, distributed:** Monarch stores all data in RAM across thousands of servers. No persistent storage layer — data is evicted after the retention window.
- **Pull-based collection** (like Prometheus): Monarch scrapes targets. This provides health checking of monitored services as a side effect.
- **Universal metrics:** Every process on Google infrastructure exposes metrics via an internal library. Coverage is near 100%.
- **Multi-dimensional data model:** Same as Prometheus labels. Each time series = metric name + label set.
- **Global queries:** Unlike Prometheus (single-cluster), Monarch can query across all of Google's clusters in a single query. This is the key scale difference.
- **Scale:** Monarch handles ~10 billion time series and ~10 trillion data points per day.

**Comparison to Prometheus/Thanos:**
```
Prometheus: single-cluster, local disk storage, PromQL
Thanos:     multi-cluster, object storage for long-term, global PromQL
Monarch:    global, in-memory, proprietary query language, no per-cluster separation
```

### Dapper (Distributed Tracing)
Google's original distributed tracing system (2010). Directly inspired Jaeger, Zipkin, and OpenCensus (now OpenTelemetry).

**Key Dapper concepts that became standards:**
- **Trace and Span model:** Every trace is a tree of spans. Each span has trace_id, span_id, parent_span_id. This is now OTel's data model.
- **Annotation (now "Events" in OTel):** Timestamped log entries within a span.
- **Sampling:** Dapper introduced the concept that 100% tracing is impossible at scale. Head sampling (sampling at request entry) with trace-consistent propagation.
- **Context propagation via headers:** Dapper propagates trace context in HTTP headers. This became the W3C Trace Context standard.

**Dapper's scale:**
- At the time of the 2010 paper: ~1 billion spans per second across all Google services.
- Sampled at 1/1024 = still hundreds of millions of spans stored per day.

### Borgmon (Historical — now replaced by Monarch)
The original Google monitoring system (before 2014). Directly inspired Prometheus.

**Borgmon concepts that became Prometheus:**
- "Varz" pages — each service exposes its metrics as HTTP-accessible variables (now Prometheus `/metrics` endpoint).
- Time-series data model with labels.
- In-process computation (now PromQL).
- Alert rules evaluated against metrics.

### Stackdriver / Cloud Operations Suite
Google Cloud's managed observability platform for GCP customers (not internal Google):

| Component | Purpose | Open-source equivalent |
|---|---|---|
| Cloud Monitoring | Metrics | Prometheus + Grafana |
| Cloud Logging | Logs | Loki / Elasticsearch |
| Cloud Trace | Distributed traces | Tempo / Jaeger |
| Cloud Profiler | CPU/memory profiling | Pyroscope / Parca |
| Error Reporting | Exception tracking | Sentry |

---

## Google SRE Observability Philosophy

### "Monitor Symptoms, Not Causes"
Alerts should fire on user-visible symptoms (high latency, errors) not internal implementation details (high CPU, disk space). An alert on CPU > 80% might be wrong (the service is fine). An alert on p99 > SLO is always right (users are affected).

### The Four Golden Signals
Google's minimum required instrumentation for every user-facing service. [[obs/patterns/four-golden-signals]]

### Error Budget and Burn Rate Alerting
The key Google innovation in alerting. [[obs/concepts/slo-burn-rate]]

### "Toil" in Observability
Manual alert investigation without a runbook is toil. Google's expectation: every alert has a runbook; every alert is resolved within its runbook. If it can't be, that's an engineering gap, not a hero moment.

---

## Google Observability Interview Questions

**System Design:**
- "Design a monitoring system for 1 million services."
  → Monarch-inspired: distributed pull-based collection, in-memory with write-ahead log, global query federation, per-cluster Prometheus for local + Thanos for global.
- "Design Google's distributed tracing system."
  → Dapper-inspired: context propagation in RPC headers, head sampling at 1/1024, probabilistic with error-always policy, Bigtable for span storage indexed by trace_id.
- "How would you alert on a service that processes 10 billion requests per day?"
  → SLO-based alerting, not threshold alerting; multi-window burn rate; exemplars for metrics→traces link.

**Deep Technical:**
- "Why does Prometheus use pull instead of push?"
  → Pull detects unreachable targets (monitored service appears down automatically); multiple Prometheus can scrape same target; curl-testable. Borgmon/Monarch also use pull.
- "How does Google handle trace sampling at 10M QPS?"
  → Head sampling at 1% with trace-consistent decisions (all services in call chain sample together via `ParentBased`); tail sampling for error traces via Collector; Adaptive sampling adjusting rate based on available storage budget.

## Sources
- [[obs/sources/google-sre-book-obs]]
- [[sre/companies/google]]
