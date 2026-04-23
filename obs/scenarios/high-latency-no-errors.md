# Scenario: High Latency With Zero Error Rate

**Topic:** [[obs/topics/tracing]], [[obs/topics/metrics]]
**Pattern:** [[obs/patterns/four-golden-signals]], [[obs/patterns/red-use-method]]
**Companies:** [[obs/companies/google]], [[obs/companies/meta]], [[obs/companies/apple]]

## The Scenario

Alert fires: `p99 latency > 2s on checkout-svc (SLO threshold: 500ms)`

But: `error rate = 0.0%` — all requests are completing successfully.

How do you diagnose this?

---

## Why Zero Errors + High Latency is Tricky

When errors are > 0, the error logs are your starting point. When errors are 0 but latency is high, the system is working — just slowly. This means:
- No stack traces in error logs
- No failed requests to filter on
- The slowness is in the "happy path"

Common causes:
1. **Slow downstream dependency** (DB, cache, external API) — request waits
2. **Resource saturation** (CPU throttling, memory pressure, GC pauses)
3. **Network degradation** (retransmits, packet loss, congestion) 
4. **Lock contention** (DB row locks, mutex in application code)
5. **Thread pool exhaustion** — requests queue waiting for a thread
6. **Serialization bottleneck** — a single goroutine/thread doing all work
7. **Data growth** — a DB query that was fast at 1M rows is slow at 100M

---

## Step-by-Step Diagnosis

### Step 1: Confirm the Signal (2 min)
```promql
# Is the latency really elevated across all pods or just one?
histogram_quantile(0.99,
  sum by (le, pod) (
    rate(http_request_duration_seconds_bucket{service="checkout"}[5m])
  )
)
```

If only one pod is slow: likely a pod-specific issue (bad node, corrupted JVM heap, memory pressure on that node).

If all pods are slow: a shared dependency (DB, cache, external service).

```promql
# Has it been slow continuously or in spikes?
histogram_quantile(0.99, sum by (le) (
  rate(http_request_duration_seconds_bucket[1m])  # 1m resolution shows spikes
))
```

Continuous = structural issue (schema, resource, config). Spiky = GC pauses, traffic bursts, mutex contention.

---

### Step 2: Isolate the Slow Endpoint (2 min)
```promql
# p99 by endpoint — which URL is slow?
histogram_quantile(0.99,
  sum by (le, endpoint) (
    rate(http_request_duration_seconds_bucket{service="checkout"}[5m])
  )
)
```

If only `/api/checkout/finalize` is slow, not `/api/checkout/items`: the problem is in the payment or order-creation path, not the cart logic.

---

### Step 3: Trace the Slow Requests (3 min)

Open a trace from the alert time window. Use Tempo/Jaeger:

```traceql
# Tempo: find slow traces from checkout
{resource.service.name="checkout-svc" && duration > 1s}
```

The flame view shows which span is consuming the time:

```
checkout (root)  0ms → 2100ms
├─ auth-check      0ms → 5ms    ← fast
├─ cart-fetch     10ms → 15ms   ← fast
└─ payment-call   20ms → 2090ms ← SLOW
   └─ db.execute  25ms → 2085ms ← DB is the bottleneck
```

The `db.execute` span attribute usually contains the SQL query (parameterized):
```
db.statement: "SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC"
db.system: postgresql
```

---

### Step 4: Check the Resource the Trace Points To (3 min)

If the trace shows a DB query is slow:
```promql
# DB connection pool saturation
db_pool_connections_active / db_pool_connections_max > 0.9

# DB query latency (if instrumented)
histogram_quantile(0.99, rate(db_query_duration_seconds_bucket{query="get_user_orders"}[5m]))
```

Check DB directly:
```sql
-- PostgreSQL: find slow queries
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Active queries right now
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '1 second';

-- Check for missing indexes
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at DESC;
```

If the issue is a missing index on `(user_id, created_at)`, a query that was fast at 10K rows is slow at 10M rows — this is a data growth trigger.

---

### Step 5: Resource Saturation (Parallel Check) (2 min)

While investigating the trace, also check USE metrics:

```promql
# CPU throttling (containers are CPU-constrained)
rate(container_cpu_cfs_throttled_periods_total{service="checkout"}[5m])
  /
rate(container_cpu_cfs_periods_total{service="checkout"}[5m])
> 0.25   # >25% throttling = CPU is the constraint

# Memory pressure
container_memory_working_set_bytes{service="checkout"}
  /
container_spec_memory_limit_bytes{service="checkout"}
> 0.9    # >90% of limit = GC pauses likely in JVM, risk of OOM

# Thread pool (app-specific — check application metrics)
thread_pool_active_threads / thread_pool_max_threads
```

---

### Step 6: Check Logs for the Slow Requests (2 min)

In Loki, search for warnings around the alert time:

```logql
{app="checkout-svc"} | json
  | level="warn" or level="error"
  | line_format "{{.timestamp}} {{.message}} {{.duration_ms}}"
```

Look for:
- `connection pool timeout` → thread pool or DB pool exhaustion
- `GC pause` → JVM memory pressure
- `context deadline exceeded` → downstream timeout
- `retry attempt` → downstream failing intermittently

---

### Step 7: Network (if all else fails)

```bash
# From inside the pod: check retransmit rate to DB
ss -i dst <db-ip> | grep -E "retrans|rto"

# Or via node metrics
rate(node_network_transmit_errs_total[5m]) > 0
```

High TCP retransmit rate → packet loss between app and DB, possibly due to noisy neighbor on shared NIC.

---

## Summary of Cause → Signal → Fix

| Cause | Signal | Fix |
|---|---|---|
| Slow DB query | Trace: db.execute span > 1s | Add index; rewrite query; cache result |
| DB connection pool full | `db_pool_active/max > 0.9` | Increase pool size; add DB read replica |
| CPU throttling | `cfs_throttled_periods > 25%` | Increase CPU limit; optimize hot path |
| GC pauses (JVM) | Memory > 90% of limit; `gc_pause_ms` metric | Increase heap; tune GC; reduce allocations |
| Thread pool exhausted | `thread_pool_active/max > 0.9` | Increase pool; use async I/O |
| Missing index (data growth) | Trace: DB slow; EXPLAIN shows seq scan | `CREATE INDEX CONCURRENTLY` |
| Network retransmits | `ss` retrans count; Cilium network policy | Fix NIC queue depth; move noisy neighbor |

---

## Interview Script

When asked "how would you debug high latency with zero errors?":

> "First I'd confirm whether the latency is elevated across all pods or just one — to decide if it's a shared dependency or a pod-specific issue. Then I'd break it down by endpoint to narrow the scope. I'd pull a trace from the affected time window using Tempo to see which span is consuming the time — the flame view usually points directly to the slow service. If the trace shows a DB query, I'd check `pg_stat_statements` for slow queries and use EXPLAIN to check for missing indexes. In parallel I'd check USE metrics — CPU throttling, memory pressure — since those cause application-level slowdowns that trace spans also capture. Finally I'd search Loki for any warnings around pool exhaustion or retry storms."

## Sources
- [[obs/patterns/four-golden-signals]]
- [[obs/concepts/tempo-jaeger]]
- [[obs/concepts/prometheus]]
