# Metrics

**Topic:** [[obs/overview]]
**Related:** [[obs/concepts/prometheus]], [[obs/patterns/four-golden-signals]], [[obs/patterns/red-use-method]]

Survey of the metrics pillar: what metrics are, how they're stored, and how they're used in interviews.

---

## What Are Metrics?

Metrics are **numeric measurements aggregated over time**. They answer: "How is the system performing right now, and how has it trended?"

Key properties:
- **Time-stamped:** Every data point has a timestamp.
- **Aggregated:** Not per-request — they're counts, rates, or distributions over a time window.
- **Low cardinality:** Good metric design uses a small, bounded set of label combinations.
- **Cheap to query:** Pre-aggregated, so answering "what is the p99 latency?" takes milliseconds.

---

## Metric Types

### Counter
Monotonically increasing. Never decreases (resets to 0 on restart).

```
http_requests_total{method="GET", status="200"} 15234
```

**Derive rate with:** `rate(counter[5m])` — gives per-second rate over the last 5 minutes.

**Interview gotcha:** Never alert on a counter's raw value — it only goes up. Always use `rate()` or `increase()`.

### Gauge
Instantaneous value that can go up or down.

```
memory_usage_bytes{pod="api-server-xyz"} 524288000
queue_depth{queue="payments"} 423
```

**Alert directly on gauges.** A gauge at 95% disk usage is immediately actionable.

### Histogram
Distributes observations into pre-defined buckets. Allows computing percentiles.

```
http_request_duration_seconds_bucket{le="0.1"} 234
http_request_duration_seconds_bucket{le="0.5"} 891
http_request_duration_seconds_bucket{le="1.0"} 1023
http_request_duration_seconds_bucket{le="+Inf"} 1050
http_request_duration_seconds_count 1050
http_request_duration_seconds_sum   523.4
```

**Compute p99 with:** `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))`

**Why histograms over summaries?** Histograms are aggregatable across instances — you can compute the p99 across all replicas. Summaries compute quantiles per-instance and **cannot be aggregated**.

### Summary (Deprecated Pattern)
Pre-computes quantiles on the client side. Not aggregatable. Avoid in new instrumentation.

---

## Cardinality: The Most Important Constraint

**Cardinality** = the total number of unique time series in your metrics store = `∏(unique values per label)`.

```
Labels: {method, status, path, user_id}
If: method(4) × status(10) × path(100) × user_id(1,000,000) = 4,000,000,000 series
→ Prometheus dies. This is a cardinality explosion.
```

**Safe labels:** `method`, `status_code`, `region`, `service`, `version` — bounded, small sets.
**Dangerous labels:** `user_id`, `request_id`, `IP address`, `session_token` — unbounded.

Full deep dive: [[obs/concepts/cardinality]]

---

## Time-Series Databases

| Database | Architecture | Query Language | Best For |
|---|---|---|---|
| Prometheus | Pull-based; local disk; single node | PromQL | Single-cluster, scrape-based |
| Thanos | Prometheus + object storage (S3/GCS) | PromQL | Multi-cluster, long-term retention |
| Cortex / Mimir | Horizontally scalable Prometheus | PromQL | Large-scale, multi-tenant |
| VictoriaMetrics | Single binary, very fast, compressed | MetricsQL (PromQL superset) | High-cardinality, high-ingest |
| InfluxDB | Push-based; line protocol | Flux / InfluxQL | IoT, high-frequency writes |
| Datadog | SaaS; DogStatsD | Metrics Explorer | Managed, enterprise |

**Interview question:** "What's the difference between Prometheus and Thanos?"
→ Prometheus has local storage with a default 15-day retention. Thanos adds a sidecar that ships blocks to object storage (S3/GCS), enabling unlimited retention and global queries across multiple Prometheus instances.

---

## Key Metrics Frameworks

### The Four Golden Signals (Google SRE)
For any user-facing service: **Latency, Traffic, Errors, Saturation**
→ [[obs/patterns/four-golden-signals]]

### RED Method (Per Microservice)
**R**ate, **E**rrors, **D**uration — the service-level trinity
→ [[obs/patterns/red-use-method]]

### USE Method (Per Resource/Infrastructure)
**U**tilization, **S**aturation, **E**rrors — for CPUs, disks, network interfaces
→ [[obs/patterns/red-use-method]]

---

## PromQL Essentials

```promql
# Error rate (fraction of requests that are 5xx)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# p99 latency across all pods
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# CPU throttling rate per container
rate(container_cpu_cfs_throttled_periods_total[5m])
  /
rate(container_cpu_cfs_periods_total[5m])

# Memory working set (what the OOM killer uses)
container_memory_working_set_bytes{namespace="prod"}

# Available error budget (fraction remaining)
1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) / 0.001  -- SLO error budget is 0.1% = 0.001
```

Full reference: [[obs/concepts/prometheus]]

---

## Interview Common Questions

**Q: Why use rate() instead of increase() for counters?**
A: `rate()` gives per-second rate (normalized by time interval). `increase()` gives total increase over the window. Use `rate()` for dashboards (consistent scale regardless of scrape interval), `increase()` for "how many errors in the last hour."

**Q: A counter resets to 0 (service restarted). Does rate() break?**
A: No. Prometheus detects counter resets (when a value decreases) and adjusts the calculation automatically.

**Q: What's the difference between a histogram and a gauge?**
A: A gauge is a single instantaneous value. A histogram is a distribution — it captures *how values are distributed* (e.g., 80% of requests finish in <100ms, 99% in <500ms).

**Q: When would you choose Thanos over Cortex?**
A: Thanos is easier to operate (Prometheus-compatible sidecars, no additional database). Cortex/Mimir is better for multi-tenant scenarios with thousands of tenants and high ingest rates — it shards both the write and read paths.

## Sources
- [[obs/concepts/prometheus]]
- [[obs/sources/prometheus-docs]]
