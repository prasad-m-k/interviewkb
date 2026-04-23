# Top 15 Observability Interview Q&A

**Topic:** [[obs/overview]]
**Format:** Question → Key Points → Depth pointer

---

## 1. What are the three pillars of observability? What does each answer?

**Key points:**
- **Metrics:** "How is the system performing?" — aggregated numeric measurements, cheap to store, fast to query. Prometheus/Datadog.
- **Logs:** "What happened and why?" — discrete events, high-cardinality, expensive. Loki/Elasticsearch.
- **Traces:** "Which service is slow?" — request journeys across services, establishes causality. Tempo/Jaeger.
- Fourth pillar (emerging): **Profiling** — continuous CPU/memory flame graphs. Pyroscope/Parca.

**Depth:** [[obs/overview]]

---

## 2. What are the Four Golden Signals? Give a PromQL query for each.

**Key points:**
- **Latency:** `histogram_quantile(0.99, sum by (le)(rate(http_request_duration_seconds_bucket[5m])))`
- **Traffic:** `sum(rate(http_requests_total[5m]))`
- **Errors:** `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`
- **Saturation:** `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])`

**Trap:** Average latency is wrong — always use histograms and percentiles (p50/p95/p99).

**Depth:** [[obs/patterns/four-golden-signals]]

---

## 3. What is the difference between RED and USE methods?

**Key points:**
- **RED (per service):** Rate, Errors, Duration — measures from user perspective.
- **USE (per resource):** Utilization, Saturation, Errors — measures infrastructure health.
- Use RED for microservice dashboards and SLOs; use USE for debugging resource bottlenecks.
- On-call alerts: 4GS (symptoms). Dashboards: RED. Debugging: USE.

**Depth:** [[obs/patterns/red-use-method]]

---

## 4. What is cardinality and why does high cardinality kill Prometheus?

**Key points:**
- Cardinality = number of unique time series = product of unique values per label.
- Each series needs ~3KB RAM. 5M series = 15GB just for in-memory Head block.
- High-cardinality labels: user_id, request_id, IP, raw URL path — avoid as Prometheus labels.
- Fix: `metric_relabel_configs` to drop labels at scrape time; normalize paths; use logs for per-user data.
- Prevention: `label_limit`, `enforce_sample_limit` in Prometheus config.

**Depth:** [[obs/concepts/cardinality]]

---

## 5. What is an SLO burn rate and how do you alert on it?

**Key points:**
- `burn_rate = error_rate / (1 - SLO_target)`. If SLO=99.9% and error rate=1%: burn rate = 10×.
- At 10× burn rate, monthly budget exhausts in 3 days.
- Multi-window alerting: fast burn (5m + 1h, >14.4×) → page; slow burn (30m + 6h, >6×) → warning.
- Two windows prevent false positives from brief spikes.

**Depth:** [[obs/concepts/slo-burn-rate]]

---

## 6. What is the difference between head sampling and tail sampling for traces?

**Key points:**
- **Head:** Decision at trace start, before outcome known. Simple, zero overhead for non-sampled. Misses rare errors.
- **Tail:** Decision after trace completes. Buffers all spans for 30s. Can 100% sample errors + slow traces. Requires OTel Collector (stateful buffer).
- Use `ParentBased` sampler always — downstream services must respect the upstream sampling decision.
- `TraceIdRatioBased` without `ParentBased` causes broken traces (inconsistent sampling per service).

**Depth:** [[obs/concepts/sampling]]

---

## 7. How does Prometheus differ from Loki in what they index?

**Key points:**
- Prometheus indexes label sets (metadata) and stores numeric values. Rich query (PromQL).
- Loki indexes **only labels**, not log content. Content stored in compressed chunks (S3). LogQL for queries.
- Loki is 10-50× cheaper than Elasticsearch but slower for full-text search.
- Loki cardinality rule: same as Prometheus — never put user_id or request_id as a Loki label.

**Depth:** [[obs/concepts/loki]]

---

## 8. How do you correlate a log entry with a distributed trace?

**Key points:**
- Include `trace_id` as a structured field in every log line (OTel SDK does this automatically).
- Grafana "derived fields": clicking `trace_id` in a log line opens the trace in Tempo.
- Prometheus exemplars: attach `trace_id` to histogram samples → click metric spike → open trace.
- The full workflow: alert (metric) → trace (Tempo) → logs (Loki, filtered by trace_id) → root cause.

**Depth:** [[obs/concepts/tempo-jaeger]], [[obs/concepts/grafana]]

---

## 9. What is OpenTelemetry and why is it important?

**Key points:**
- Vendor-neutral standard for instrumenting metrics, logs, and traces. One API → any backend.
- Components: SDK (per language), OTLP (wire protocol), Collector (pipeline agent).
- Replaces vendor lock-in: before OTel, switching from Jaeger to Datadog required rewriting instrumentation.
- Auto-instrumentation: OTel instruments frameworks (FastAPI, Django, SQLAlchemy) without code changes.
- OTel Collector is required for tail sampling (stateful buffer the SDK can't do).

**Depth:** [[obs/concepts/opentelemetry]]

---

## 10. What is alert fatigue and how do you fix it?

**Key points:**
- Alert fatigue: too many noisy alerts → engineers ignore pager → real incidents missed.
- Fix step 1: audit last month's alerts — classify as actionable/self-resolving/informational/duplicate.
- Fix step 2: add `for: 5m` to suppress transient spikes.
- Fix step 3: switch to SLO burn rate alerting instead of raw thresholds.
- Fix step 4: add Alertmanager inhibition rules to suppress child alerts when parent fires.
- Target: ≤ 2 actionable pages per shift.

**Depth:** [[obs/scenarios/alert-fatigue]]

---

## 11. How do you debug "high latency but zero errors"?

**Key points:**
1. Confirm: is all pods slow or one? (pod-specific vs. shared dependency)
2. Break down by endpoint: which URL is slow?
3. Open a trace → flame view shows which span is consuming time.
4. If DB span is slow: check `pg_stat_statements`, EXPLAIN for missing index.
5. In parallel: check USE metrics — CPU throttling, memory pressure, thread pool.
6. Check Loki logs for pool exhaustion, GC pauses, retries.

**Depth:** [[obs/scenarios/high-latency-no-errors]]

---

## 12. What is the OTel Collector and what does it do that the SDK can't?

**Key points:**
- OTel Collector: vendor-neutral pipeline between apps and backends.
- **Tail sampling:** requires buffering full traces — can't do in app SDK (no visibility into all spans).
- **Protocol translation:** OTLP → Prometheus remote write → Loki push — one SDK, any backend.
- **PII scrubbing:** remove/hash sensitive attributes before export.
- **Buffering + retry:** if backend is down, Collector buffers and retries; SDK might drop.
- Collector is optional but recommended for production.

**Depth:** [[obs/concepts/opentelemetry]]

---

## 13. What is eBPF and how is it used for observability?

**Key points:**
- eBPF: Linux kernel technology that runs sandboxed programs in response to events (syscalls, network packets) without modifying app code.
- Cilium/Hubble: L4/L7 network observability between pods without Istio sidecars.
- Pyroscope/Parca: continuous CPU flame graphs without code changes (uprobes on functions).
- Zero-instrumentation tracing: Odigos/Groundcover use eBPF uprobes to auto-generate OTel spans.
- Safety: eBPF verifier prevents kernel crashes — safer than kernel modules.

**Depth:** [[obs/concepts/ebpf-observability]]

---

## 14. What is Alertmanager and how does inhibition work?

**Key points:**
- Alertmanager: receives alerts from Prometheus, handles routing, dedup, grouping, silences, inhibition.
- **Inhibition:** suppress "child" alerts when "parent" alert fires. If DB is down, suppress upstream service error alerts (they're symptoms, not root cause).
- **Grouping:** batch related alerts (50 crashing pods) into one notification.
- `group_wait` = time before first notification. `group_interval` = time before sending about new alerts in group. `repeat_interval` = re-notify if still firing.

**Depth:** [[obs/patterns/alert-routing]]

---

## 15. What is the difference between Tempo and Elasticsearch for trace storage?

**Key points:**
- **Elasticsearch:** Full tag index — fast search by any span attribute (`user.id=123`). Storage: 5-10× raw size.
- **Tempo:** Object storage (S3) only. Parquet files. No per-attribute index by default. Very cheap.
- Tempo 2.x: optional tag index enables TraceQL attribute search without full scan.
- TraceQL: `{resource.service.name="payment" && duration > 1s}` — find slow payment spans.
- Choose Tempo: new deployments, Grafana shop, cost-sensitive. Choose Jaeger+ES: need rich attribute search.

**Depth:** [[obs/concepts/tempo-jaeger]]
