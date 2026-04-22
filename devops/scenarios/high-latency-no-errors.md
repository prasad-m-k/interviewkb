# Scenario: Latency Spiked but Error Rate Is Zero

**Type:** Observability / Debugging
**Difficulty:** Hard
**Frequency:** High — tests deep observability knowledge; separates strong candidates

## The Question

*"p99 latency on your API just jumped from 200ms to 4 seconds. Error rate is still 0%. Users are complaining. What do you do?"*

---

## What the Interviewer Is Testing

- Do you know that "no errors" doesn't mean "no problem"?
- Do you know where latency hides? (locks, queues, GC, saturation)
- Do you know the USE method? (Utilization, Saturation, Errors)
- Can you systematically narrow the blast radius?
- Do you understand the difference between p50, p95, p99?

---

## The Critical Insight

**Zero errors + high latency = requests are completing but slowly.**

This rules out: crashes, 5xx errors, connection refused, pod OOMKills.

This points to: **saturation** — something is at capacity and requests are queuing.

---

## Systematic Debugging Approach

### Step 1: Characterize the latency (2 min)

```
What exactly changed?
- p50 normal, p99 4s → only the slowest requests affected (outliers)
  → suspect: GC pauses, lock contention, occasional slow queries
- p50 also elevated → all requests slow
  → suspect: saturation in the hot path (DB, cache, downstream API)
- Latency affects one endpoint only → look at that endpoint's dependencies
- Latency affects all endpoints → look at shared infrastructure
```

```bash
# Grafana: look at the latency histogram, not just the average
# Is it bimodal? (two humps = two classes of requests with different behavior)
# Is it a smooth tail? (long tail = occasional slow outlier)
# Did it spike suddenly or drift up gradually?
#   Sudden: external change (deploy, traffic shift, dependency change)
#   Gradual: resource exhaustion (memory leak, disk filling, connection leak)
```

### Step 2: Correlate with the timeline (3 min)

```bash
# Was there a deploy? (most common cause)
kubectl rollout history deployment/my-app

# Did traffic increase?
# Check RPS graph — same latency at 2× RPS → you hit a scaling limit

# Did anything change in dependencies?
# Did a downstream service slow down?
# Did a DB migration run?
# Did a certificate rotate?
# Did a config change go out?

# Check deploy log / change management system
```

### Step 3: Apply USE method to each layer

For every resource in the request path, check **Utilization, Saturation, Errors**:

```
Resource           Utilization         Saturation              Errors
──────────────────────────────────────────────────────────────────────
App pods           CPU %               Request queue depth     5xx (none)
Thread pool        Active threads      Pending tasks           Rejected tasks
DB connections     Pool active/total   Waiting for connection  Query errors
DB server          CPU, I/O            Lock wait time          Deadlocks
Cache (Redis)      Memory used         Eviction rate           Connection errors
Message queue      Consumer lag        Queue depth growth      DLQ messages
Downstream APIs    —                   —                       Latency at source
Network            Bandwidth           Packet retransmits      TCP resets
```

```bash
# App pod CPU and memory
kubectl top pods -n prod

# DB connection pool (Postgres example)
kubectl exec -it postgres-pod -- psql -U postgres -c \
  "SELECT count(*), state, wait_event_type, wait_event 
   FROM pg_stat_activity 
   GROUP BY state, wait_event_type, wait_event 
   ORDER BY count DESC;"
# Look for: rows with state='idle in transaction' or wait_event='Lock'

# DB slow queries
kubectl exec -it postgres-pod -- psql -U postgres -c \
  "SELECT query, calls, mean_exec_time, total_exec_time 
   FROM pg_stat_statements 
   ORDER BY mean_exec_time DESC LIMIT 10;"

# Redis latency
redis-cli --latency-history -i 1
```

### Step 4: Check distributed traces (most powerful for microservices)

```
If you have distributed tracing (Jaeger, Zipkin, Tempo):

1. Find a slow request trace (filter by duration > 2s)
2. Expand the trace waterfall:
   - Which span is slow? (which service? which operation?)
   - Is there a gap before a span starts? → queuing time
   - Is a span itself slow? → that service/DB is the bottleneck
   - Are spans sequential that should be parallel? → code smell, not infra

Example trace:
[API gateway      ────────────────────────────────────── 4.1s]
  [Auth service   ─ 12ms]
  [Payment svc    ──────────────────────────────── 4.0s]    ← HERE
    [DB query     ─ 3ms]
    [wait         ───────────────────── 3.9s]               ← queuing!
    [DB query     ─ 80ms]

Conclusion: Payment service was waiting 3.9s before getting a DB connection
→ DB connection pool exhausted → add PgBouncer or increase pool
```

### Step 5: Common causes by symptom pattern

| Pattern | Most likely cause | Check |
|---|---|---|
| p99 high, p50 normal | GC pauses / lock contention / slow DB query (rare) | GC logs, DB slow query log |
| All percentiles elevated | Saturation (DB pool, thread pool, cache miss storm) | Connection pool metrics, cache hit rate |
| Single endpoint slow | That endpoint's specific dependencies | Trace that endpoint specifically |
| Started after deploy | Code regression (N+1 query, removed index, new lock) | Diff the deploy; check DB query count |
| Gradual over hours/days | Resource leak (memory → swap, connection leak, disk) | Memory trend, fd count, disk usage |
| Correlated with traffic spike | Scaling bottleneck (not enough replicas, DB at limit) | CPU/connection saturation at that time |
| Specific time of day | Batch job, cron, backup running concurrently | Scheduled job list; DB I/O at that time |

---

## Deep Dives

### GC Pauses (JVM / Go / Python)

```bash
# JVM: enable GC logging
-Xlog:gc*:file=/tmp/gc.log:time,uptime:filecount=5,filesize=20M

# Look for Stop-The-World pauses > 500ms
# Pattern: GC pause every N seconds → p99 latency spike every N seconds
# Fix: tune heap size, switch GC algorithm (G1 → ZGC for low-latency)

# Go: GODEBUG=gctrace=1 shows GC frequency and pause duration
```

### Thread Pool / Executor Saturation

```python
# If your async service has a thread pool (e.g., gunicorn workers):
# All workers busy → new requests queue → latency = queue wait + processing time

# Check: active_threads / max_threads
# If consistently > 80% → scale pods or increase thread count

# Diagnose:
# - Add queue_depth metric to your executor
# - Add span around thread acquisition (shows queue wait in traces)
```

### N+1 Query After a Deploy

```
New code: "get all orders, then for each order get the user"

Loop: SELECT * FROM orders;  →  100 rows
Then: SELECT * FROM users WHERE id=1;
      SELECT * FROM users WHERE id=2;
      ...100 queries...

Total: 101 DB queries per API call
Before: 1 query with JOIN

Fix: JOIN or batch fetch (SELECT * FROM users WHERE id IN (...))
Signal: DB query count jumped after deploy despite same traffic
```

---

## Communication During Incident

```
1. Immediately post in incident channel:
   "p99 latency 4s, error rate 0%, investigating. No user-visible errors yet."

2. After initial triage (5 min):
   "Latency is in the DB layer — connection pool exhausted.
    Applying PgBouncer config and scaling connection limits. ETA 10 min."

3. After mitigation:
   "Latency back to normal. Root cause: N+1 query introduced in deploy at 14:32.
    Hotfix in progress. Postmortem scheduled for tomorrow."
```

---

## Common Follow-Up Questions

**Q: "How would you have caught this before it hit prod?"**
A: Load testing in staging (k6, Locust) with production-like traffic patterns. DB query count as a metric (alert if queries/request jumps 2×). Canary deploys with p99 latency in the canary analysis template — not just error rate.

**Q: "What if you can't find the cause but users are complaining?"**
A: Mitigate first: rollback the most recent deploy if timing correlates. If timing doesn't correlate, consider: rate limiting (reduce load on the saturated resource), circuit breaker (stop forwarding to a slow downstream), or scaling out (throw more resources at it while investigating).

**Q: "What's the difference between p50, p95, and p99?"**
A: p50 = median, half of requests are faster. p99 = 99th percentile, only 1% of requests are slower. Averages hide the tail — a service averaging 100ms could have p99 of 5s if some requests are very slow. FAANG companies care about p99/p999 because at high volume, 1% of requests is millions of users.

## Sources
- [[devops/concepts/observability-pillars]]
- [[devops/topics/observability]]
- [[devops/patterns/incident-response]]
