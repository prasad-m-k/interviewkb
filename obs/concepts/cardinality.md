# Cardinality in Observability

**Topic:** [[obs/topics/metrics]], [[obs/topics/logging]]
**Related:** [[obs/concepts/prometheus]], [[obs/concepts/loki]]

## What is Cardinality?

**Cardinality** is the number of unique values in a set. In observability:

- **Metrics cardinality** = total number of unique time series = product of unique values per label
- **Log stream cardinality** = total number of unique label combinations in Loki

```
Metric: http_requests_total{method, status, endpoint}
  method: GET, POST, PUT, DELETE       → 4 values
  status: 200, 201, 400, 404, 500, 503 → 6 values
  endpoint: /api/users, /api/orders    → 2 values
  Total: 4 × 6 × 2 = 48 series        ← acceptable

Same metric + user_id label:
  user_id: 1,000,000 unique users      → 1,000,000 values
  Total: 4 × 6 × 2 × 1,000,000 = 48,000,000 series  ← CATASTROPHIC
```

---

## Why High Cardinality Kills Prometheus

Every time series requires:
- RAM for the in-memory cache (Head TSDB)
- Disk I/O for compaction
- CPU for query evaluation

**At 1M+ series:**
- Prometheus RAM usage: 3–10 GB
- Scrape time exceeds scrape interval → targets appear down
- PromQL queries take minutes instead of milliseconds
- Prometheus OOM-kills itself

**The cardinality explosion scenario:**
A developer adds `user_id` or `request_id` as a Prometheus label. At low traffic, tests pass. At production traffic (100K unique users/day), Prometheus adds 100K new series daily. After 1 week: 700K series. Prometheus is slow. After 1 month: 3M series. Prometheus falls over.

---

## High vs. Low Cardinality Data

| Data type | Cardinality | Right home |
|---|---|---|
| HTTP status code | ~10 values | Prometheus metric label |
| HTTP method | ~5 values | Prometheus metric label |
| Region / AZ | ~20 values | Prometheus metric label |
| Pod name | ~1000 values | Kubernetes labels (k8s metadata) |
| User ID | 1M+ values | Log field, trace attribute |
| Request ID / Trace ID | Infinite | Trace backend, log field |
| SQL query (raw) | Very high | Trace span attribute (parameterized) |

**Rule:** If a label value can be an unbounded set, it does not belong in Prometheus metrics. Use logs (Loki/Elasticsearch) or traces (Tempo/Jaeger) for high-cardinality data.

---

## Detecting Cardinality Problems

### Prometheus self-metrics
```promql
# Total time series in Prometheus
prometheus_tsdb_head_series

# Series per job
topk(10,
  count by (job) ({__name__=~".+"})
)

# Metric with highest series count
topk(10,
  count by (__name__) ({__name__=~".+"})
)

# Label with highest unique value count (shows which label is the culprit)
topk(10,
  count by (label_name) (
    group by (label_name, label_value) ({__name__="http_requests_total"})
  )
)
```

### Warning signs
- Prometheus RAM > 4GB on a typical-load cluster
- `prometheus_target_scrape_pool_exceeded_label_limits_total` counter increasing
- Scrape duration approaching scrape interval
- `prometheus_tsdb_head_chunks_created_total` rate growing faster than new services

### Loki cardinality
```
# Loki self-metrics
loki_ingester_streams_count  -- total active streams
loki_distributor_streams_per_tenant  -- streams per tenant
```

High stream count in Loki means the index is large and memory-heavy. Same symptom as Prometheus: OOM crashes.

---

## Fixing a Cardinality Explosion

### Step 1: Identify the culprit
```promql
# Find the metric with the most series
topk(5, count by (__name__) ({__name__=~".+"}))

# Find labels with the most unique values on the worst offender
count by (offending_label_name) (http_requests_total)
```

### Step 2: Remove or replace the high-cardinality label

**Option A: Drop the label entirely**
```yaml
# Prometheus scrape config: drop user_id label at scrape time
scrape_configs:
  - job_name: 'api'
    metric_relabel_configs:
      - source_labels: [user_id]
        action: labeldrop
        regex: 'user_id'
```

**Option B: Replace with a bounded category**
```python
# Instead of label user_id="12345", use tier="premium"/"free"
requests_total.labels(tier=get_user_tier(user_id)).inc()
```

**Option C: Replace with a log field**
```python
# Log the user_id instead of metricking it
logger.info("request completed", extra={
    "user_id": user_id,
    "trace_id": get_trace_id(),
    "latency_ms": latency
})
# Now queryable in Loki: {app="api"} | json | user_id="12345"
```

### Step 3: Add cardinality limits
```yaml
# Prometheus global limits (prevents future accidents)
global:
  enforce_sample_limit: 100000  # max samples per scrape

  # Per-series label count limits
  target_limit: 5000            # max targets per scrape config
  label_limit: 30               # max labels per time series
  label_name_length_limit: 200  # max label name length
  label_value_length_limit: 200 # max label value length
```

---

## Cardinality-Safe Instrumentation Guidelines

```python
# BAD: unbounded labels
http_requests.labels(
    user_id=user_id,            # millions of users
    request_id=request_id,      # unique per request
    path=request.path,          # /api/users/12345/orders — includes IDs
    ip=client_ip,               # millions of IPs
).inc()

# GOOD: bounded labels
http_requests.labels(
    method=request.method,      # GET/POST/PUT/DELETE
    status=str(response.status)[:3],  # 200/400/500
    endpoint=normalize_path(request.path),  # /api/users/{id}/orders
    region=os.environ["REGION"],       # us-east-1
).inc()

def normalize_path(path: str) -> str:
    """Replace numeric IDs and UUIDs with {id} placeholder."""
    import re
    path = re.sub(r'/\d+', '/{id}', path)
    path = re.sub(r'/[0-9a-f-]{36}', '/{uuid}', path)
    return path
```

**Path normalization** is critical for endpoints with IDs in the path. Without it, `/api/users/1`, `/api/users/2`, ... `/api/users/1000000` each become separate time series.

---

## Cardinality and Tracing

Distributed traces are **natively high-cardinality** — each trace is a unique record. This is why trace data lives in object storage (Tempo) or Elasticsearch (Jaeger), not in Prometheus. The cardinality problem doesn't apply to traces because trace backends are designed for it.

However, trace **attributes** can still cause storage problems:
- Don't index raw SQL in the trace attribute index (parameterize: `SELECT ... WHERE id = ?`)
- Don't include large binary data as span attributes (attach as trace events or links, not attributes)

---

## Interview Questions

**Q: A developer wants to add `user_id` as a Prometheus label to track per-user error rates. How do you respond?**
A: Prometheus is the wrong tool for per-user data. With 1M users, adding `user_id` creates 1M new time series per metric — Prometheus will OOM. Instead: (1) use a log field (`user_id` in structured JSON logs, queryable in Loki), (2) use a trace attribute (trace all failed requests, search by `user.id` in Tempo), or (3) aggregate into bounded cohorts if a metric is needed (`tier=premium/free`, `account_age=new/existing`).

**Q: How do you monitor for cardinality growth before it becomes a crisis?**
A: Alert on `prometheus_tsdb_head_series > 500000` (or whatever your safe limit is). Alert on the growth rate: `rate(prometheus_tsdb_head_series[24h]) > 10000` (series added per second over 24 hours is high). Set `enforce_sample_limit` in scrape configs to hard-cap any single target.

**Q: What is label value explosion from dynamic path parameters?**
A: An HTTP framework that uses the raw request path as a label creates a new time series for every unique path. `/api/users/1`, `/api/users/2` are different labels. Fix: normalize paths (replace numeric segments with `{id}`) before using as a label value.

## Sources
- [[obs/concepts/prometheus]]
- [[obs/concepts/loki]]
