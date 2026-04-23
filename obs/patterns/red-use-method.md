# RED Method and USE Method

**Topic:** [[obs/topics/metrics]]
**Related:** [[obs/patterns/four-golden-signals]], [[obs/concepts/prometheus]]

## Overview

RED and USE are complementary frameworks for structuring observability:

| Framework | Scope | Questions |
|---|---|---|
| **RED** | Per *service* | What is the service doing for users? |
| **USE** | Per *resource* | Is this infrastructure component healthy? |
| **4 Golden Signals** | Per *user-facing service* | Is the user experience good? |

Use RED to instrument your microservices. Use USE to instrument your infrastructure. Use 4GS for your SLOs and on-call alerts.

---

## RED Method (Per Microservice)

Coined by Tom Wilkie (Grafana Labs). Three metrics per service:

| Signal | Definition | PromQL example |
|---|---|---|
| **R**ate | Requests per second | `sum(rate(http_requests_total[5m]))` |
| **E**rrors | Error rate (fraction) | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **D**uration | Latency distribution | `histogram_quantile(0.99, sum by (le)(rate(http_request_duration_seconds_bucket[5m])))` |

### Why RED for Microservices?

In a microservice architecture with 50 services, you need a uniform instrumentation pattern that every team can implement without deciding from scratch what to measure. RED gives a consistent baseline:
- Every service gets the same 3 metric families.
- Every service gets the same dashboard template.
- On-call knows exactly where to look, regardless of which service is alerting.

### RED Dashboard (Minimal Viable)

```promql
# Row 1: Rate
sum by (service) (rate(http_requests_total[5m]))

# Row 2: Errors
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/ sum by (service) (rate(http_requests_total[5m]))

# Row 3: Duration (p50, p95, p99)
histogram_quantile(0.50, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

### RED for Different Service Types

```promql
# HTTP service
rate: sum(rate(http_requests_total[5m]))
errors: sum(rate(http_requests_total{status=~"5.."}[5m])) / ...
duration: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# gRPC service
rate: sum(rate(grpc_server_started_total[5m]))
errors: sum(rate(grpc_server_handled_total{grpc_code!="OK"}[5m])) / ...
duration: histogram_quantile(0.99, rate(grpc_server_handling_seconds_bucket[5m]))

# Kafka consumer
rate: sum(rate(kafka_messages_consumed_total[5m]))
errors: sum(rate(kafka_consumer_errors_total[5m])) / ...
duration: histogram_quantile(0.99, rate(kafka_message_processing_duration_seconds_bucket[5m]))
```

---

## USE Method (Per Resource)

Coined by Brendan Gregg. Three metrics per infrastructure resource:

| Signal | Definition | Examples |
|---|---|---|
| **U**tilization | % time the resource is busy | CPU: `rate(cpu_seconds_total{mode!="idle"}[5m])` |
| **S**aturation | Work queued waiting for the resource | CPU: run queue length; Disk: I/O queue depth |
| **E**rrors | Error events on the resource | Disk: `rate(node_disk_io_time_weighted_seconds_total[5m])` (I/O errors) |

### USE Applied to Common Resources

```promql
# CPU Utilization
avg by (node) (
  1 - rate(node_cpu_seconds_total{mode="idle"}[5m])
)

# CPU Saturation (throttled periods — containers)
rate(container_cpu_cfs_throttled_periods_total[5m])
/ rate(container_cpu_cfs_periods_total[5m])

# Memory Utilization (working set / limit)
container_memory_working_set_bytes
/ container_spec_memory_limit_bytes

# Memory Saturation (OOM kill events)
increase(container_oom_events_total[5m]) > 0

# Disk Utilization
1 - (node_filesystem_free_bytes / node_filesystem_size_bytes)

# Disk Saturation (I/O queue depth)
node_disk_io_time_weighted_seconds_total

# Network Utilization (% of bandwidth)
rate(node_network_transmit_bytes_total[5m]) * 8
/ node_network_speed_bytes

# Network Errors
rate(node_network_transmit_errs_total[5m])
rate(node_network_receive_errs_total[5m])

# DB Connection Pool Utilization
db_pool_connections_active / db_pool_connections_max

# DB Connection Pool Saturation (waiting for connection)
db_pool_connections_waiting
```

---

## Combining RED and USE: The Debugging Flow

When latency spikes (RED: Duration high), use USE to find the resource bottleneck:

```
Alert: p99 latency > 1s on checkout-svc (RED: Duration)
                        │
          ┌─────────────▼──────────────┐
          │  Check USE for each resource │
          └──────────┬─────────────────┘
                     │
    ┌────────────────┬┴───────────────────┐
    ▼                ▼                    ▼
  CPU?            Memory?             Disk/Network?
  ─────           ────────            ────────────
  Throttle > 25%? OOM events?        I/O queue high?
  Run queue high? Working set        Network errors?
                  near limit?
         │
         ▼
  Database?
  ─────────────
  Connection pool full? (USE: Saturation)
  Slow queries? (RED: Duration on DB layer)
  Lock contention?
```

---

## When to Use Which Framework

| Situation | Use |
|---|---|
| Defining SLOs and error budgets | 4 Golden Signals |
| Creating a standard service dashboard | RED |
| Debugging "what's consuming resources?" | USE |
| On-call alert design | 4 Golden Signals (alert on symptoms) |
| Capacity planning | USE (utilization trends) |
| Microservice-to-microservice calls | RED on each hop |
| Node/VM/bare-metal health | USE |

---

## Interview Questions

**Q: What's the difference between RED and USE?**
A: RED measures from the *user's perspective* — what the service is doing for users (rate, error, latency). USE measures from the *infrastructure's perspective* — how full or broken a resource is (utilization, saturation, errors). Use RED for service dashboards and SLOs; use USE for infrastructure debugging.

**Q: Can you apply USE to a database?**
A: Yes. Utilization = query throughput / max throughput. Saturation = connection pool waiters or lock contention queue depth. Errors = failed queries or deadlocks. USE on a database often surfaces the root cause of what RED surfaces as "latency spike" in an upstream service.

**Q: Which framework do you use for on-call alerts?**
A: 4 Golden Signals, specifically the latency and error signals — they represent direct user impact. RED and USE metrics are for dashboards and diagnosis, not primary alerts. Never alert on CPU utilization (USE) unless it directly predicts user-visible degradation.

## Sources
- [[obs/patterns/four-golden-signals]]
- [[obs/concepts/prometheus]]
