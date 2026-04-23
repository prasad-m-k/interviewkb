# Meta Observability Interview Prep

**Related:** [[obs/overview]], [[sre/companies/meta]]

## Meta's Internal Observability Stack

Meta runs some of the most sophisticated internal observability systems in the world. Understanding them is important because:
1. Meta PE interviews frequently ask "how would you design X at Facebook scale?"
2. Several Meta tools (Scuba, ODS) have architectural patterns that appear in interview system design questions.
3. Meta interviewers expect familiarity with the names and concepts even if you haven't used the tools.

### Scuba (Fast Log Analytics)
Scuba is Meta's in-memory, real-time log and event analytics database. Used primarily for **ad-hoc debugging and investigation**, not monitoring.

**Key characteristics:**
- **In-memory only:** All data in RAM across a cluster. No disk persistence.
- **Optional schema (schema-on-write):** Rows can have any columns. Missing columns in a row are just absent — no NULL handling.
- **Sub-second queries:** On billions of rows. Designed for interactive investigation, not batch analytics.
- **Short retention:** Typically days to weeks, not months. Scuba is ephemeral by design.
- **Used for:** "Show me all checkout errors in the last 2 hours, grouped by error_type, broken down by region."

**Conceptual equivalent:** Like Elasticsearch for logs, but faster for aggregations, with no inverted index, and all in memory.

**Interview use:** When asked "how do you design a fast log analytics system for debugging," describe Scuba: columnar in-memory storage, optional schema, aggregation-first (COUNT, SUM, AVG), time-partitioned for fast range queries.

### ODS (Operational Data Store)
Meta's time-series metrics system — the Prometheus equivalent.

**Key characteristics:**
- **Pull-based collection** (like Prometheus): ODS scrapes targets.
- **Gorilla-style compression:** Time-series data is delta-of-delta compressed — same algorithm that became the Gorilla paper (from Meta's research team, 2015).
- **Global storage:** Unlike Prometheus (single-cluster), ODS aggregates metrics globally across all datacenters.
- **Used for:** Service health dashboards, on-call monitoring, SLO tracking.

**The Gorilla Paper (must-know for Meta interviews):**
```
Gorilla compression for time-series:
- XOR consecutive floating-point values (instead of storing raw values)
- Most metrics change slowly: delta is tiny → XOR compresses well
- Delta-of-delta on timestamps: scrape intervals are regular → deltas are often 0
- Result: 12 bytes per data point → 1.37 bytes on average (8.7× compression)
```

### Laser (Distributed Key-Value Cache)
Meta's distributed cache — positioned between TAO (graph DB) and Memcache. Laser provides persistent, fast K-V lookups for read-heavy workloads with higher hit rates than Memcache.

Relevant for observability: Laser stores aggregated metric summaries for fast dashboard rendering — the equivalent of Prometheus recording rules.

### Pixel (Real User Monitoring — RUM)
Meta uses client-side instrumentation to measure real user experience — actual browser/app performance, not server-side proxies.

**Key RUM metrics:**
- **LCP (Largest Contentful Paint):** Time until the largest visible element loads.
- **FID (First Input Delay):** Time from user's first interaction to browser response.
- **CLS (Cumulative Layout Shift):** Visual stability of the page.
- **TTFB (Time to First Byte):** Server response time as experienced by the client.

**Why RUM matters:** Server-side p99 latency can be 50ms but client experience can be 3s due to JavaScript execution, CDN routing, and client hardware. RUM captures the real user experience.

---

## Meta PE Observability Questions

### System Design
- "Design Scuba — a real-time log analytics system for 100 petabytes of logs."
  → In-memory columnar store, schema-on-write, time-partitioned across a cluster, aggregation push-down, short retention (evict after 1 week), write path: structured log → Kafka → ingestion workers → Scuba nodes.

- "Design ODS — a metrics system for Meta's global infrastructure."
  → Pull-based collection (like Prometheus), Gorilla-compressed time-series storage, global fanout via hierarchical aggregation (region → global), OTel-compatible collection agent, recording rules for dashboard performance.

- "How do you monitor for memory leaks in a fleet of 100,000 containers?"
  → ODS metrics: `container_memory_working_set_bytes` trend over 24h per service; alert if growth rate > 10MB/hour sustained for > 6h (linear trend detection). Then: continuous profiling (Pyroscope/Parca) to identify the allocating code path. The metric detects, the profiler diagnoses.

### Debugging Scenarios (PE Round)
- "You see in Scuba that checkout error rate spiked 5x starting at 14:37. How do you investigate?"
  → (1) Cross-reference with deploys at 14:37 in the deployment system. (2) Filter Scuba to error_type breakdown — which error type spiked? (3) Find a `trace_id` from the error logs, open in the trace system to see which service failed. (4) Correlate with ODS metrics — did latency spike too? Did connection pool exhaust?

---

## Meta Observability Philosophy

### "Move Fast With Stable Infra"
Meta's observability investment is justified by this principle: you can only move fast if the infrastructure is reliable enough to catch and contain failures before they cascade. Observability is the feedback loop that makes fast iteration safe.

### Scuba as the Debugging First Step
At Meta, the debugging flow for production issues:
1. Check ODS (metrics) for which service's RED metrics changed.
2. Query Scuba (logs) to see error details and distribution.
3. Open a trace (internal tracing system) to find the exact request path.
4. Use profiling (Pyroscope equivalent) if it's a performance regression.

The key insight: Scuba replaces "ssh into the server and grep the logs" with a SQL-like query against a distributed in-memory store — 10-second query vs. 10-minute grep.

---

## Sources
- [[sre/companies/meta]]
- [[sre/sources/scaling-memcache-facebook]]
