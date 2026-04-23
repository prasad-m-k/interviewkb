# The Four Golden Signals

**Topic:** [[obs/topics/metrics]], [[obs/topics/alerting]]
**Related:** [[obs/concepts/prometheus]], [[obs/patterns/red-use-method]]

## What They Are

The Four Golden Signals (Google SRE Book, Chapter 6) are the minimum set of metrics every user-facing service must monitor:

| Signal | Question answered | What to measure |
|---|---|---|
| **Latency** | How long does a request take? | p50, p95, p99 of request duration |
| **Traffic** | How much demand is on the system? | Requests per second, messages/sec |
| **Errors** | How often do requests fail? | 5xx rate, timeout rate, exception rate |
| **Saturation** | How full is the system? | CPU %, memory %, queue depth |

**Why these four?** They cover the full user-experience spectrum: users experience latency, errors, and unavailability (caused by saturation). Traffic provides context for whether high latency is expected load or anomalous.

---

## 1. Latency

**What to measure:** Request duration histogram. Never use averages.

```promql
# p99 latency across all pods (aggregate the histogram)
histogram_quantile(0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# Multi-percentile panel (p50, p95, p99)
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# p99 latency by endpoint
histogram_quantile(0.99,
  sum by (le, endpoint) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**Interview: Why p99 instead of average?**
Average hides the tail. If 1% of users get a 10-second response, the average barely moves (if 99% are at 50ms). The p99 captures the worst experience a significant fraction of users are having. Amazon reported that every 100ms of latency costs 1% in sales — the tail matters.

**Error vs. success latency:** Track them separately. A failing request that returns in 1ms is fast for the wrong reason. Your SLI should specify: "p99 latency of *successful* requests < 200ms."

---

## 2. Traffic

**What to measure:** Request rate (RPS/QPS), message throughput, or write/read volume.

```promql
# Requests per second
sum(rate(http_requests_total[5m])) by (service)

# Traffic breakdown by method
sum by (method) (rate(http_requests_total[5m]))

# Kafka consumer lag (alternative traffic signal for async systems)
kafka_consumer_group_lag{topic="payments", group="checkout-worker"}

# gRPC traffic
sum(rate(grpc_server_started_total[5m])) by (grpc_method)
```

**Traffic alerts:** Alert on unusual drops (service went down silently) or unusual spikes (DDoS, viral event, cron job misbehavior). Use percentage-of-baseline alerting, not absolute thresholds:

```promql
# Alert if traffic drops to < 20% of the same time yesterday
sum(rate(http_requests_total[5m]))
  /
sum(rate(http_requests_total[5m] offset 24h))
  < 0.2
```

---

## 3. Errors

**What to measure:** Rate of failed requests — HTTP 5xx, timeouts, exceptions.

```promql
# Error rate (fraction of requests that are 5xx)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# Error rate by service and endpoint
sum by (service, endpoint) (rate(http_requests_total{status=~"5.."}[5m]))
  /
sum by (service, endpoint) (rate(http_requests_total[5m]))

# Include 4xx client errors separately (don't conflate with server errors)
sum(rate(http_requests_total{status=~"4.."}[5m]))  # client errors — not your fault
sum(rate(http_requests_total{status=~"5.."}[5m]))  # server errors — your fault

# Timeout errors (often excluded from status code metrics)
sum(rate(http_requests_total{error="timeout"}[5m]))
```

**4xx vs 5xx:** 4xx errors are client errors (bad request, unauthorized, not found). They reflect client behavior, not service health. SLOs should be based on 5xx rates, not all error rates — unless your clients are misconfigured because of your API, which is still your problem.

**Error budget alerting:** Use SLO burn rate, not raw error rate. [[obs/concepts/slo-burn-rate]]

---

## 4. Saturation

**What to measure:** How full is the resource? Saturation predicts degradation before it causes errors.

```promql
# CPU saturation (throttling rate — more meaningful than utilization)
rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m])
  /
rate(container_cpu_cfs_periods_total{container!=""}[5m])

# Memory saturation (working set vs limit)
container_memory_working_set_bytes
  /
container_spec_memory_limit_bytes

# Database connection pool saturation
db_pool_connections_active / db_pool_connections_max

# Queue depth saturation
kafka_consumer_group_lag / kafka_topic_partition_end_offset * 100

# Disk saturation
node_filesystem_size_bytes - node_filesystem_free_bytes
  /
node_filesystem_size_bytes
```

**CPU utilization vs. throttling:** CPU utilization (% of time cores are busy) can be high but fine. CPU throttling (% of scheduling periods where the container exceeded its CPU limit) directly causes latency spikes. Alert on throttling, not just utilization.

**Saturation is predictive:** A service at 80% capacity is not in trouble today but may be tomorrow when traffic grows 10%. Alert to create headroom:

```promql
# Alert if memory working set > 85% of limit
container_memory_working_set_bytes / container_spec_memory_limit_bytes > 0.85
```

---

## Combining All Four: The Golden Signal Dashboard Row

```
┌──────────────────────────────────────────────────────────────────────┐
│  Service: checkout-svc | Namespace: prod | Time: last 1h            │
├──────────────────────────────────────────────────────────────────────┤
│  Latency (p99)         Traffic (RPS)                                 │
│  ┌─────────────────┐  ┌─────────────────┐                           │
│  │ 142ms ▁▃▅▇█▄▂▁  │  │ 1.2k  ▂▄▆██▆▄▂ │                           │
│  │ ↑ 12ms from 1h  │  │ ↑ 8% from 1h   │                           │
│  └─────────────────┘  └─────────────────┘                           │
│                                                                      │
│  Errors (rate)         Saturation (CPU throttle)                    │
│  ┌─────────────────┐  ┌─────────────────┐                           │
│  │ 0.02% ▁▁▁▁▁▁▁▁  │  │ 3.2%  ▁▁▂▁▁▁▁▁ │                           │
│  │ ✓ SLO: 0.1%     │  │ ✓ < 25% target │                           │
│  └─────────────────┘  └─────────────────┘                           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Applying to Different Service Types

| Service type | Latency SLI | Traffic | Errors | Saturation |
|---|---|---|---|---|
| HTTP API | p99 request duration | RPS | 5xx rate | CPU throttle, connection pool |
| gRPC | p99 RPC duration | RPC/sec | non-OK status rate | CPU, thread pool |
| Message consumer | p99 processing time | Messages/sec | Failed/DLQ rate | Consumer lag, batch size |
| Database | p99 query duration | Queries/sec | Error query rate | Connection pool, lock wait |
| CDN / cache | Cache hit latency | Cache requests/sec | Cache error rate | Cache hit rate, eviction rate |
| Batch job | Job completion time | Jobs/hour | Failed job rate | Queue depth, worker saturation |

---

## Interview Questions

**Q: Why use p99 and not average?**
A: Averages mask tail latency. p99 = 99% of users experience this latency or better. For user-facing services, the tail experience is what drives churn and SLA breaches. A 1% tail at 10 seconds affects thousands of users per hour.

**Q: What's the difference between saturation and utilization?**
A: Utilization = % of time the resource is busy. Saturation = how much work is *waiting* (queue depth, throttled periods). A CPU at 80% utilization with no throttling is fine. A CPU at 70% utilization with 50% throttling rate means the service is CPU-constrained and latency is being added by the scheduler.

**Q: A service has 0% errors and normal latency, but traffic dropped 90%. Is this an incident?**
A: Almost certainly yes — the service isn't accepting requests. Possible causes: upstream router is misconfigured and sending traffic elsewhere, the service is silently crash-looping and restarting before it can serve requests, or a caching layer absorbed all traffic. Traffic drops are a signal even when latency and errors look healthy.

## Sources
- [[obs/topics/metrics]]
- [[obs/concepts/prometheus]]
- [[obs/sources/google-sre-book-obs]]
