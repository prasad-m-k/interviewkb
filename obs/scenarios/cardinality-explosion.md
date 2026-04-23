# Scenario: Prometheus Cardinality Explosion

**Topic:** [[obs/topics/metrics]]
**Related:** [[obs/concepts/cardinality]], [[obs/concepts/prometheus]]

## The Scenario

Prometheus RAM usage jumped from 2GB to 14GB overnight. Scrape durations are approaching the scrape interval (15s). PromQL queries are timing out. Dashboards show "No data" for some metrics.

---

## Immediate Diagnosis

### 1. Check total series count
```promql
prometheus_tsdb_head_series
# If this jumped from ~100K to ~5M → cardinality explosion
```

### 2. Find the exploding metric
```promql
# Top 10 metrics by series count
topk(10,
  count by (__name__) ({__name__=~".+"})
)
```

### 3. Identify the bad label
```promql
# For the offending metric (e.g., http_requests_total):
# Find which label has unbounded values
count by (job, label_name) (
  label_replace(http_requests_total, "label_name", "$1", "__name__", "(.*)")
)
-- substitute with a group_left label inspection
```

Simpler approach — check unique values per label with a quick query:
```bash
# Via Prometheus HTTP API
curl 'http://prometheus:9090/api/v1/label/user_id/values' | jq '.data | length'
# If this returns 1,000,000 → user_id is the culprit
```

### 4. Find which target introduced it
```promql
# Series by job (scrape target)
count by (job) ({__name__=~".+"})

# When did it start? Check the rate of series creation
rate(prometheus_tsdb_head_series[1h])
```

Correlate the timestamp of the series explosion with recent deploys using Grafana annotations.

---

## Immediate Mitigation (Stop the Bleeding)

### Option A: Drop the label at scrape time
```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: 'api'
    static_configs:
      - targets: ['api:8000']
    metric_relabel_configs:
      # Drop the offending label (user_id) before storing
      - source_labels: [user_id]
        action: labeldrop
        regex: 'user_id'
      # Or: drop the entire metric if it can't be fixed
      - source_labels: [__name__]
        action: drop
        regex: 'http_requests_by_user_total'
```

Reload config without restarting Prometheus:
```bash
curl -X POST http://prometheus:9090/-/reload
```

### Option B: Drop at the application level (requires deploy)
```python
# Fix the metric instrumentation
# BEFORE (bad — user_id label)
requests_by_user.labels(user_id=user_id).inc()

# AFTER (good — bounded label)
requests_by_tier.labels(tier=get_user_tier(user_id)).inc()
# AND: log the user_id in structured logs for per-user queries
logger.info("request completed", extra={"user_id": user_id, "trace_id": trace_id})
```

### Option C: Reduce retention (temporary)
If Prometheus is out of disk too:
```yaml
# prometheus.yml
global:
  retention.size: 10GB  # hard cap instead of time-based
```

---

## Remediation (Fix the Root Cause)

1. **Deploy the fixed instrumentation** (remove/replace the bad label).
2. **Verify series count drops** after old TSDB blocks with the bad label age out (default block rotation: 2h).
3. **Add cardinality limits** to prevent recurrence:

```yaml
# prometheus.yml global limits
global:
  enforce_sample_limit: 100000    # max samples per target per scrape

scrape_configs:
  - job_name: 'api'
    sample_limit: 10000           # per-scrape limit (specific to this target)
    label_limit: 30               # max labels per time series
    label_value_length_limit: 200 # max length of any label value
```

4. **Add an alert for cardinality growth:**
```yaml
- alert: PrometheusHighCardinality
  expr: prometheus_tsdb_head_series > 1000000
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus series count > 1M — cardinality check needed"

- alert: PrometheusRapidSeriesGrowth
  expr: rate(prometheus_tsdb_head_series[1h]) > 10000
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus series growing at > 10K/hour"
```

---

## Prevention

### Instrumentation Code Review Checklist
Every PR that adds/modifies metrics should be reviewed for:
- [ ] No unbounded label values (user IDs, request IDs, raw paths, IP addresses)
- [ ] Path parameters are normalized (`/api/users/{id}` not `/api/users/12345`)
- [ ] Label cardinality documented in the PR description
- [ ] Metric follows naming conventions

### Automated Cardinality Budget Enforcement
```yaml
# Validate metrics in CI before deploy
# prometheus-cardinality-check.sh
MAX_SERIES=100000
ACTUAL=$(curl -s http://staging-prometheus:9090/api/v1/query?query=prometheus_tsdb_head_series | jq '.data.result[0].value[1]' -r)
if [ "$ACTUAL" -gt "$MAX_SERIES" ]; then
  echo "ERROR: Prometheus series count ($ACTUAL) exceeds budget ($MAX_SERIES)"
  exit 1
fi
```

---

## Interview Q&A

**Q: A developer added a `url` label to their HTTP metrics. It's been 24 hours and Prometheus is at 50GB RAM. What do you do right now?**
A: Immediate: add a `metric_relabel_config` to drop the `url` label at scrape time and reload Prometheus config (no restart needed). This stops the bleeding immediately. Next: deploy fixed code that removes the label from instrumentation or normalizes the URL. Long-term: add `label_limit` to scrape configs and a cardinality alert.

**Q: After dropping the bad label, when will Prometheus RAM decrease?**
A: New scrapes stop adding new series immediately. But the existing series in the TSDB Head block (in-memory) are retained until they're no longer active (no data in 5× scrape interval) and the block is compacted (every 2 hours by default). Expect RAM to drop gradually over 4–6 hours as old blocks compact and the Head block shrinks.

**Q: Why can't you just increase Prometheus RAM to handle high cardinality?**
A: You can temporarily, but it doesn't scale. Each time series needs ~3KB of RAM. At 5M series: 15GB just for the Head block. At 50M series: 150GB — prohibitively expensive. Horizontal scaling (Thanos/Cortex/Mimir) helps with query load but not with per-instance cardinality — you still need to reduce the cardinality at the source.

## Sources
- [[obs/concepts/cardinality]]
- [[obs/concepts/prometheus]]
