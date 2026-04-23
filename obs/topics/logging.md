# Logging

**Topic:** [[obs/overview]]
**Related:** [[obs/concepts/loki]], [[obs/topics/tracing]], [[obs/patterns/alert-routing]]

Survey of the logging pillar: what logs are, how they flow, and how they're used in investigations and interviews.

---

## What Are Logs?

Logs are **timestamped records of discrete events**. They answer: "What happened, in what order, and in what context?"

Key properties:
- **Discrete:** Each line is one event, not an aggregation.
- **High-cardinality:** Logs can contain user IDs, request IDs, full stack traces — details metrics can't.
- **Expensive to store and query:** A busy service can emit GB/hour. Storage and indexing cost is significant.
- **The link between metrics and traces:** A log line can carry a `trace_id` that lets you jump from a log event to the full request trace.

---

## Structured vs. Unstructured Logs

```
# Unstructured (legacy)
2026-04-22 10:34:21 ERROR User 123 failed to checkout order 456 — payment_timeout after 3421ms

# Structured JSON (preferred)
{
  "timestamp": "2026-04-22T10:34:21.432Z",
  "level":     "error",
  "service":   "checkout-svc",
  "trace_id":  "4bf92f3577b34da6a3ce929d0e0e4736",
  "user_id":   "123",
  "order_id":  "456",
  "event":     "checkout_failed",
  "reason":    "payment_timeout",
  "latency_ms": 3421
}
```

**Why structured?** Structured logs are parseable without regex. You can query `level=error AND reason=payment_timeout` in Loki, Elasticsearch, or CloudWatch Insights instantly. Unstructured logs require fragile regex patterns that break when the message format changes.

**The `trace_id` field is the critical bridge** — it connects a log event to the distributed trace for that request, enabling the Metrics → Trace → Log debugging workflow.

---

## Log Levels

| Level | Meaning | Production volume |
|---|---|---|
| **FATAL / CRITICAL** | Service is about to die or is dead | Zero (ideally never happens) |
| **ERROR** | A request or operation failed; needs investigation | Low; every occurrence matters |
| **WARN** | Unexpected but recoverable; may indicate a growing problem | Low–medium |
| **INFO** | Normal significant events (startup, config loaded, request completed) | Medium |
| **DEBUG** | Detailed diagnostic info — enabled only during development/troubleshooting | Disabled in prod by default |
| **TRACE** | Extremely verbose — every function call, every byte | Never enabled in production |

**Interview rule:** "Never log DEBUG in production" — the volume overwhelms storage and the signal/noise ratio is useless. Use sampling or dynamic log level adjustment via config (no redeploy needed).

---

## The Log Pipeline

```
Application → stdout/stderr
                    │
              ┌─────▼──────┐
              │  Log Agent  │   (runs as DaemonSet on each node)
              │  Fluentd /  │   collects, parses, batches, filters
              │  Fluent Bit │
              └─────┬──────┘
                    │
          ┌─────────▼──────────┐
          │   Message Buffer   │   (optional, for durability)
          │   Kafka / Kinesis  │   backpressure + replay
          └─────────┬──────────┘
                    │
          ┌─────────▼──────────┐
          │    Log Store       │
          │  Loki / ES / CW    │   index + chunk/store
          └─────────┬──────────┘
                    │
          ┌─────────▼──────────┐
          │   Query + Viz      │
          │   Grafana / Kibana │   LogQL / KQL / Insights query
          └────────────────────┘
```

### Fluent Bit vs. Fluentd
| | Fluent Bit | Fluentd |
|---|---|---|
| Written in | C | Ruby |
| Memory footprint | ~650KB | ~40MB |
| Best for | Edge/node agents (Kubernetes DaemonSet) | Central aggregation |
| Plugins | Fewer but growing | Extensive ecosystem |
| Recommendation | Use Fluent Bit as node agent, Fluentd for aggregation tier |

---

## Log Storage Systems

### Loki (Grafana Labs)
- Inspired by Prometheus but for logs.
- **Index only labels** (pod name, namespace, service), not the full log content.
- Full log content stored in compressed chunks (object storage: S3/GCS).
- **Much cheaper than Elasticsearch** because the content index is tiny.
- Query with **LogQL** — a log query language similar to PromQL.

```logql
# All error logs from checkout service in the last 1 hour
{app="checkout-svc"} |= "error" | json | level="error"

# Error rate per minute from nginx
sum by (status) (
  rate({app="nginx"}[1m])
)

# p99 latency from structured logs
quantile_over_time(0.99,
  {app="api"} | json | unwrap latency_ms [5m]
) by (endpoint)
```

Deep dive: [[obs/concepts/loki]]

### Elasticsearch / OpenSearch
- Inverts the index: every word in every log is indexed.
- Extremely fast full-text search.
- **Expensive:** index is often 2–5× the raw log size.
- **Kibana** for visualization; **KQL** (Kibana Query Language) or Lucene for queries.

```
# KQL query in Kibana
level: "error" AND service: "checkout" AND latency_ms > 1000

# Lucene
+level:error +service:checkout +latency_ms:[1000 TO *]
```

### CloudWatch Logs Insights
AWS-managed. No infrastructure to run. Query with its own SQL-like language:
```sql
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as error_count by bin(5m)
| sort error_count desc
```

---

## Log Sampling

At high volume (>100K events/sec), logging everything is impractical. Strategies:

| Strategy | How | Tradeoff |
|---|---|---|
| **Head sampling** | Log 1 in N requests | Simple; misses rare events |
| **Tail sampling** | Log all errors and a fraction of successes | Better coverage; requires buffering |
| **Dynamic log level** | Temporarily increase to DEBUG for specific services | Surgical; requires runtime config |
| **Adaptive sampling** | Rate-limit logs per category | Complex; avoids bursts |

**Recommendation:** Always log ERROR and WARN at 100%. Sample INFO at 10–100% depending on volume. Never log DEBUG in prod unless requested.

---

## Interview Questions

**Q: What is the difference between Loki and Elasticsearch?**
A: Loki only indexes labels (metadata); content is stored in compressed chunks in object storage. Much cheaper but only good for label-based filtering + regex on content. Elasticsearch indexes every word — much faster for full-text search, but storage and compute costs are 5–10× higher.

**Q: How do you correlate a log with a trace?**
A: The application includes a `trace_id` field in every structured log line (injected by the OTel SDK automatically). From a log, you click the `trace_id` to open the full trace in Tempo/Jaeger. From an alert, you jump to the offending trace, then to its logs. This is the "exemplars" feature in Prometheus+Grafana.

**Q: How do you handle PII (Personally Identifiable Information) in logs?**
A: Never log raw PII. Options: (1) mask at write time (hash user IDs, truncate emails), (2) structured log field with a policy tag that triggers downstream redaction, (3) log tokens/references rather than the data itself. At Apple scale: don't log PII at all — log anonymized request IDs and resolve to user data only in secure, audited systems.

**Q: What is log rotation and why does it matter for SREs?**
A: Log rotation limits on-disk log file growth by compressing and archiving old logs (logrotate, journald with max-size). Without it, a verbose application fills the disk and crashes the service. The SRE gotcha: a file deleted by logrotate but still held open by the writing process keeps consuming inodes until the process restarts — the "deleted file open handle" trap. [[sre/concepts/disk-and-io]]

## Sources
- [[obs/concepts/loki]]
- [[obs/sources/prometheus-docs]]
