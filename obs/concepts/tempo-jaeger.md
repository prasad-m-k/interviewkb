# Tempo and Jaeger — Distributed Tracing Backends

**Topic:** [[obs/topics/tracing]]
**Related:** [[obs/concepts/opentelemetry]], [[obs/concepts/sampling]], [[obs/concepts/grafana]]

## Overview

Both Tempo and Jaeger are open-source backends that store and query distributed traces. They differ in architecture, storage model, and query capability.

| | Tempo | Jaeger |
|---|---|---|
| Maintainer | Grafana Labs | CNCF (originally Uber) |
| Storage | Object storage (S3/GCS/Azure) only | Elasticsearch, Cassandra, BadgerDB |
| Query language | TraceQL | Jaeger UI (limited) |
| Integration | Native Grafana | Jaeger UI, some Grafana support |
| Scalability | Virtually unlimited (object storage) | Limited by backend storage |
| Cost | Very low (S3 is cheap) | Higher (ES/Cassandra licensing + ops) |
| Index | Trace ID only (no tag index) | Tag-based search |

**Modern recommendation:** Tempo for new deployments. Jaeger for teams with existing Elasticsearch or Cassandra infrastructure.

---

## Tempo Architecture

```
Write Path:
  App (OTLP) ──► Distributor ──► Ingester ──► Flush ──► Object Store (S3)
                                                          (parquet files)

Read Path:
  Query Frontend ──► Querier ──► Ingester (recent) + Store (historical)
        │
  TempoQL/TraceQL engine
```

**Critical design choice:** Tempo stores traces in **object storage only** (S3, GCS, Azure Blob). There is no separate database. This makes Tempo:
- Extremely cheap (object storage is 10–100× cheaper than database storage)
- Trivially scalable (S3 is effectively infinite)
- Simpler to operate (no Elasticsearch cluster to maintain)

**Tradeoff:** Tempo cannot search by span attributes (e.g., "find all traces where `db.system=postgresql`"). It can only look up a specific trace by trace_id, or use TraceQL with the new "tag search" feature (requires additional index).

### Tempo with Tag Search (Tempo 2.x)
Tempo 2.x added a **tag index** (stored in object storage alongside trace data) that enables TraceQL attribute searches without a full scan:

```
# Works in Tempo 2.x with tag index enabled
{span.http.url =~ ".*/checkout.*" && duration > 500ms}
{resource.service.name="payment-svc" && span.status.code=error}
```

---

## TraceQL

Tempo's query language for traces. Inspired by PromQL + LogQL.

### Trace Selectors

```traceql
# Select traces from a specific service
{resource.service.name="checkout-svc"}

# Select slow traces
{duration > 1s}

# Select traces with an error span
{span.status.code=error}

# Select traces from a service with a specific attribute
{resource.service.name="payment-svc" && span.http.status_code=500}

# Combine: slow traces from payment service with errors
{resource.service.name="payment-svc" && duration > 500ms && span.status.code=error}
```

### Span Sets (child selector)

```traceql
# Traces where any span in the trace exceeds 1s (not just the root)
{} | select(span.duration > 1s)

# Traces where the db span is slow
{span.db.system="postgresql" && span.duration > 200ms}

# Count spans per service in a trace
{} | select(count() by resource.service.name)
```

### Structural Operators (Tempo 2.3+)

```traceql
# Traces where A called B (parent-child relationship)
{resource.service.name="checkout"} >> {resource.service.name="payment"}

# Traces where A and B both appear (not necessarily parent-child)
{resource.service.name="checkout"} && {resource.service.name="payment"}
```

---

## Jaeger Architecture

```
App (OTLP/Thrift) ──► Collector ──► Queue (Kafka optional) ──► Store
                                                                (ES/Cassandra/Badger)
                                                                     │
                                           Query Service ◄───────────┘
                                                │
                                           Jaeger UI
```

**Storage options:**
- **Elasticsearch (recommended for scale):** Full tag indexing; can search by any span attribute.
- **Cassandra:** High write throughput; good for time-based queries; limited tag search.
- **BadgerDB:** Embedded, in-memory-first; development only.

**Jaeger UI features:**
- Search by service, operation, tag, duration, error
- Trace comparison
- Service graph (call topology)
- Flamegraph view (in newer versions)

---

## Jaeger vs Tempo — When to Choose

**Choose Tempo when:**
- You're deploying from scratch on Kubernetes
- You use Grafana (native integration)
- You want minimal infrastructure (no ES cluster)
- You can tolerate limited attribute-based search (or use Tempo 2.x with tag index)
- Cost is a priority

**Choose Jaeger when:**
- You already operate Elasticsearch or Cassandra
- You need rich attribute-based search (find all traces with `user.id=123`)
- Your team is familiar with Jaeger UI
- You want trace-level search without TraceQL

---

## OTLP Export Configuration

### To Tempo (via OTel Collector)
```yaml
# otel-collector-config.yaml
exporters:
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

### To Jaeger (via OTel Collector)
```yaml
exporters:
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
```

### Direct from App (without Collector)
```python
# Direct OTLP to Tempo (not recommended for production — use Collector)
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(endpoint="http://tempo:4317", insecure=True)
```

---

## Exemplars: The Metrics ↔ Traces Link

Prometheus histograms can carry **exemplars**: sample `{trace_id, span_id}` attached to a specific bucket observation.

```python
# Python prometheus_client with exemplar
from prometheus_client import Histogram
from opentelemetry import trace

request_duration = Histogram('http_request_duration_seconds', 'Duration', ['method'])

def handle_request():
    span = trace.get_current_span()
    ctx = span.get_span_context()
    
    with request_duration.labels(method='GET').time():
        # Attach exemplar with trace context
        result = process()
    
    # Exemplar is automatically attached by OTel if using prometheus_client >= 0.13
    # with OTEL_TRACES_EXPORTER configured
```

**In Grafana:** When viewing a histogram panel, enable "Exemplars" to show dots on the graph — each dot is a sample request. Click a dot → opens the full trace for that request in Tempo.

This is the critical "metrics → trace → log" correlation workflow.

---

## Interview Questions

**Q: What is the difference between Tempo and Elasticsearch for trace storage?**
A: Elasticsearch indexes span attributes, enabling arbitrary tag-based search. Tempo stores traces as objects (parquet files) in S3 — no attribute index by default, only trace_id lookup. Tempo is 10–100× cheaper but trades attribute search for cost. Tempo 2.x adds a lightweight tag index that partially closes the gap.

**Q: What is a trace exemplar?**
A: An exemplar is a sample trace_id attached to a Prometheus histogram observation. When you see a p99 spike in Grafana, you can click an exemplar dot to jump directly to the trace that caused that spike. It bridges metrics (aggregated trends) and traces (specific request detail).

**Q: How do you find which service introduced a performance regression after a deploy?**
A: (1) Check p99 latency metrics per service — which one spiked? (2) Open a trace from the spike period in Tempo. (3) Use Tempo's service graph to visualize call latencies. (4) Use the flame view to identify which span's duration changed. (5) Correlate with the deploy timestamp in the Grafana annotation.

## Sources
- [[obs/topics/tracing]]
- [[obs/concepts/opentelemetry]]
