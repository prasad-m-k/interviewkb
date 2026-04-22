# Observability — The Three Pillars

**Topic:** [[devops/topics/observability]]
**Related:** [[devops/patterns/incident-response]]

## Pillar 1: Metrics

Numeric measurements aggregated over time. Answer "how much" and "how fast."

### Prometheus data model

```
metric_name{label1="value1", label2="value2"} <value> <timestamp>

http_requests_total{method="GET", status="200", path="/api/users"} 15234 1714512000
```

**Metric types:**
| Type | Use | Example |
|---|---|---|
| **Counter** | Monotonically increasing; rate = slope | `http_requests_total`, `errors_total` |
| **Gauge** | Instantaneous value; can go up or down | `memory_usage_bytes`, `queue_depth` |
| **Histogram** | Distribution of values in buckets | `http_request_duration_seconds` (p50/p95/p99) |
| **Summary** | Pre-computed quantiles (avoid — not aggregatable) | Legacy |

### Key PromQL patterns

```promql
# Request rate (per-second over 5min window)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# p99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU utilization per pod
rate(container_cpu_usage_seconds_total{namespace="prod"}[5m])
```

**Cardinality trap:** each unique label combination = one time series. Never use `user_id`, `request_id`, or `IP` as labels — that's millions of series. Use logs or traces for per-request data.

---

## Pillar 2: Logs

Timestamped text records of events. Answer "what happened" and "why."

### Structured vs unstructured

```
Unstructured (bad):
2026-04-21 10:34:21 ERROR user 123 failed to checkout order 456

Structured JSON (good):
{"timestamp":"2026-04-21T10:34:21Z","level":"error","user_id":"123",
 "order_id":"456","event":"checkout_failed","reason":"payment_timeout","latency_ms":3421}
```

Structured logs are **queryable** — you can filter by `user_id=123 AND event=checkout_failed` in Loki/Elasticsearch without regex.

### Log levels

| Level | Use |
|---|---|
| ERROR | Something failed; needs investigation |
| WARN | Something unexpected but recoverable |
| INFO | Normal significant events (startup, config loaded, request served) |
| DEBUG | Detailed diagnostic; disabled in prod |

### Log aggregation pipeline

```
Pod stdout/stderr
      │
      ▼
DaemonSet (Fluentd / Fluent Bit)   ← runs on every node, scrapes container logs
      │
      ▼
Log aggregator (Kafka / direct)
      │
      ▼
Log store (Elasticsearch / Loki / CloudWatch)
      │
      ▼
Query + visualize (Kibana / Grafana)
```

---

## Pillar 3: Distributed Traces

Track a single request as it flows through multiple services. Answer "where is the latency" and "which service failed."

### Anatomy of a trace

```
Request: POST /checkout
│
├─ [frontend-svc     0ms → 250ms]  ← root span
│  ├─ [auth-svc      10ms → 30ms]  ← child span (20ms)
│  ├─ [inventory-svc 35ms → 80ms]  ← child span (45ms)
│  └─ [payment-svc   85ms → 240ms] ← child span (155ms) ← bottleneck!
│     └─ [db-query   90ms → 230ms] ← grandchild span
```

A **trace** = one request's journey. A **span** = one unit of work within that trace.

### Context propagation

Each service must forward trace headers to downstream calls:
```
W3C Trace Context headers:
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  tracestate: vendor-specific
```

### OpenTelemetry (OTel)

The standard: one SDK instruments your code, emits to any backend (Jaeger, Tempo, Datadog, Zipkin).

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

tracer = trace.get_tracer("my-service")

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("user.id", user_id)
    result = process(order)
    span.set_attribute("result.status", result.status)
```

---

## Combining All Three: A Debug Story

```
1. Alert fires: p99 latency > 2s on checkout service (METRIC)
2. Check trace for a slow request → payment-svc span taking 1.8s (TRACE)
3. Look at payment-svc logs for that time window:
   {"level":"warn","event":"db_pool_exhausted","wait_ms":1750} (LOG)
4. Root cause: database connection pool exhausted → pool size too small
5. Fix: increase pool size + add connection pool saturation metric + alert
```

Metrics → alert → traces → narrow scope → logs → root cause.

---

## Sources
- [[devops/topics/observability]]
- [[devops/overview]]
