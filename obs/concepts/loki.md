# Loki

**Topic:** [[obs/topics/logging]]
**Related:** [[obs/concepts/grafana]], [[obs/concepts/prometheus]]

## What it is

Loki is Grafana Labs' log aggregation system. It is "Prometheus, but for logs" — it indexes only metadata labels (not log content), stores compressed log chunks in object storage, and uses a Prometheus-inspired query language called LogQL.

**Key design decision:** Unlike Elasticsearch, Loki does **not** build a full-text inverted index of log content. This makes it dramatically cheaper (10–50× less storage and CPU). The tradeoff: full-text search is done via regex on compressed chunks, so it's slower than Elasticsearch for ad-hoc text queries.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                            Loki Cluster                               │
│                                                                       │
│  Write Path:                        Read Path:                        │
│  Distributor → Ingester → Chunk  ←──────── Querier → Query Frontend  │
│                  (mem + WAL)                     │                    │
│                       │                          │                    │
│                  Store Gateway ←─── Object Store (S3/GCS)            │
│                       │             (chunks + index)                  │
│              BoltDB / Table Manager                                   │
│              (index: label → chunk location)                          │
└──────────────────────────────────────────────────────────────────────┘
```

**Write path:** Log lines arrive at the Distributor, which hashes the stream's labels to route to Ingester replicas. Ingesters buffer in memory + WAL (Write-Ahead Log), then flush compressed chunks to object storage.

**Read path:** Querier fetches recent data from Ingesters (in-memory) and historical data from object storage via the Store Gateway, merges and returns results.

**Single binary mode:** All components run in one process for small-scale deployments — perfect for dev or small teams.

---

## The Label Model

Loki uses labels to identify log streams — exactly like Prometheus uses labels for time series.

```
{app="checkout", namespace="prod", pod="checkout-xyz-123"}
```

A "stream" is a unique label set. All log lines with the same label set form one stream.

**Critical rule: Keep the number of streams (label cardinality) low.**

```
BAD:  {app="api", request_id="abc123def456"}  ← new stream per request = millions of streams
BAD:  {app="api", user_id="12345"}            ← same problem
GOOD: {app="api", namespace="prod", env="production"}  ← bounded, small set
```

Loki performs poorly with high stream cardinality — each stream has its own chunk files and index entries. Cardinality explosion kills Loki the same way it kills Prometheus.

**Rule of thumb:** Labels should come from infrastructure metadata (namespace, pod, app, environment, region) — not from log content.

---

## LogQL — The Query Language

LogQL has two families of queries: **log queries** (return log lines) and **metric queries** (return time-series from log data).

### Log Queries (stream selector + pipeline)

```logql
# All logs from checkout service in prod
{app="checkout", namespace="prod"}

# Filter: only error lines
{app="checkout"} |= "error"

# Filter: lines NOT matching
{app="checkout"} != "health check"

# Regex filter
{app="checkout"} |~ "timeout|connection refused"

# JSON parsing + field filter
{app="checkout"} | json | level="error" | duration > 1000

# Pattern extraction (no JSON — parse with pattern)
{app="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <size>`
             | status >= 500

# Logfmt parsing
{app="go-service"} | logfmt | level="error"
```

### Metric Queries (aggregate log streams)

```logql
# Count error lines per minute across services
sum by (app) (
  count_over_time({namespace="prod"} |= "error" [1m])
)

# Rate of error lines per second
sum(rate({app="checkout"} |= "error" [5m]))

# p99 latency extracted from structured logs
quantile_over_time(0.99,
  {app="api"} | json | unwrap duration_ms [5m]
) by (endpoint)

# Error rate from nginx logs
sum(rate({app="nginx"} | json | status=~"5.." [1m]))
/
sum(rate({app="nginx"} [1m]))
```

### Label Filters (post-parse)
```logql
# After JSON parsing, filter on extracted fields
{app="api"} | json | http_method="POST" | http_status >= 400 | duration_ms > 500

# Numeric comparison on extracted field
{app="api"} | logfmt | level="error" | duration > 1s
```

---

## Loki vs. Elasticsearch

| | Loki | Elasticsearch |
|---|---|---|
| Index type | Label index only | Full-text inverted index |
| Storage cost | Low (compressed chunks in S3) | High (index 2–5× raw size) |
| Query speed (labels) | Fast | Fast |
| Query speed (full-text) | Slower (regex on chunks) | Very fast |
| Cardinality limit | Medium | Higher |
| Integration | Native Grafana | Kibana |
| Best for | Label-filtered log queries, metric aggregation from logs | Full-text search, complex structured queries |

**When to choose Loki:** You already have Grafana, you want to correlate with Prometheus metrics, your queries are mostly label-based, and cost is a concern.

**When to choose Elasticsearch:** You need fast full-text search, complex field-level queries, or your team is already invested in the ELK stack.

---

## Grafana + Loki Integration

In Grafana, Loki is a data source. You can:
- Explore log streams interactively (Explore view)
- Create log panels in dashboards (show raw logs alongside metric graphs)
- Derive metrics from logs (LogQL metric queries in panels)
- **Correlate traces and logs**: click a trace span in Tempo → auto-query Loki for logs with the same `trace_id`

```yaml
# Grafana data source config (datasource.yaml)
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo  # link to Tempo instance
          matcherRegex: '"trace_id":"(\w+)"'
          name: TraceID
          url: '$${__value.raw}'  # open in Tempo
```

---

## Interview Questions

**Q: Why does Loki not index log content?**
A: Full-text indexing (Elasticsearch approach) is expensive — the index is often 2–5× the raw data size. Loki trades query speed for dramatically lower storage cost. This is acceptable because most operational queries filter on labels first (service, namespace, level), then do regex on the small resulting set. Ad-hoc text search that needs to scan billions of log lines is rare in production debugging.

**Q: What happens if a Loki ingester crashes before flushing its chunks?**
A: Ingesters use a Write-Ahead Log (WAL) — every log line is written to disk before being acknowledged. On crash recovery, the ingester replays the WAL and recovers in-memory data. Additionally, Loki runs multiple ingester replicas (typically 3) with a replication factor, so a single ingester crash doesn't cause data loss.

**Q: How do you query Loki for logs from a specific trace?**
A: Include `trace_id` as a structured field in your logs (OTel does this automatically). Then: `{app="checkout"} | json | trace_id="4bf92f3577b34da6a3ce929d0e0e4736"`. Grafana Loki's "derived fields" feature lets you click a trace_id in a log line to jump directly to the trace in Tempo.

## Sources
- [[obs/topics/logging]]
- [[obs/concepts/grafana]]
