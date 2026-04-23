# System Design for SRE Interviews

**Related:** [[sre/concepts/slo-sli-sla]], [[sre/concepts/load-balancers]], [[sre/concepts/networking-fundamentals]]

This is the SRE lens on system design. SRE interviews differ from pure SWE system design: interviewers expect you to discuss **reliability, capacity, failure modes, and operational concerns** alongside the architecture itself.

---

## The SRE System Design Interview Framework

Use this structure for every design question. Adapt depth per round.

### Step 0: Clarify (2–3 min)
Never start drawing boxes until you've asked:
- **Scale:** DAU, QPS at peak, storage growth per day/month/year?
- **SLO targets:** What latency and availability is expected? (p99 < 200ms? 99.9%? 99.99%?)
- **Geographic scope:** Single region? Multi-region? Global?
- **Consistency requirements:** Strong consistency (payments) vs. eventual (social feed)?
- **Read/write ratio:** 80/20? 99/1? Determines caching and storage decisions.

### Step 1: Napkin Math (2–3 min)
Google calls this **NALSD (Non-Abstract Large Scale Design)**. State the numbers before designing.

Example: "Design a URL shortener for 100M URLs, 1B daily reads"
- Reads: 1B/day = ~12K QPS average, 36K QPS peak (3× headroom)
- Storage: 100M URLs × 500 bytes = 50 GB total (trivial for one DB)
- Bandwidth: 12K QPS × 500 bytes = 6 MB/s (trivial for a CDN)
- Conclusion: scale isn't the hard problem; latency and availability are.

### Step 2: High-Level Architecture (5 min)
Draw the critical path:
```
Client → CDN → Load Balancer → API Servers → Cache → DB
                                           ↓
                                      Message Queue → Workers → Downstream
```

Identify immediately: what is the **single point of failure**? What is the **bottleneck**?

### Step 3: Reliability Design (5 min — the SRE differentiator)
Where most candidates stop, SRE candidates go deeper:

- **Redundancy:** Every tier should be N+2. No single-AZ dependencies.
- **Circuit breakers:** Wrap all downstream calls. Fail fast when a dependency is degraded.
- **Bulkheads:** Isolate critical paths. The login service should never be starved by the recommendation engine.
- **Graceful degradation:** Define what the system does when each tier fails. Serve cached data? Return empty? Show an error page?
- **Health checks:** L4 (TCP) vs. L7 (HTTP /healthz). Know the difference. L7 health checks catch app-level failures L4 misses.
- **Timeouts everywhere:** Every network call needs a timeout. Default timeouts are usually too long.

### Step 4: Capacity Planning (3 min)
- **Provisioned vs. on-demand:** Static capacity (EC2 reserved instances) for baseline; burst via autoscaling.
- **Storage growth:** At current rate, when does the DB need sharding? When does the S3 bill explode?
- **Caching tier sizing:** Cache hit rate target (95%? 99%?) → working set size → cache cluster size.

### Step 5: Observability (2 min)
State the SLIs you'd define and how you'd monitor them:
- **Availability SLI:** HTTP 5xx rate (not just uptime)
- **Latency SLI:** p99 latency of API calls (not just average)
- **Saturation:** Queue depth, DB connection pool utilization, CPU headroom

---

## The 10 Most Common SRE Design Questions

### 1. Design a Rate Limiter
**Algorithms:** Token bucket (burst-friendly), fixed window (simple), sliding window log (accurate), sliding window counter (approximate but scalable).

**Token bucket with Redis + Lua (atomic):**
```lua
-- KEYS[1] = key, ARGV[1] = tokens_per_sec, ARGV[2] = burst_size, ARGV[3] = now
local tokens = redis.call('get', KEYS[1])
if not tokens then tokens = ARGV[2] end
tokens = math.min(tonumber(ARGV[2]), tonumber(tokens) + tonumber(ARGV[1]))
if tokens >= 1 then
    redis.call('set', KEYS[1], tokens - 1, 'EX', 3600)
    return 1  -- allowed
else
    return 0  -- rate limited
end
```

**Distributed gotchas:** Without Lua atomicity, two concurrent requests can both read the same token count and both succeed (TOCTOU race). Redis Lua is single-threaded and atomic.

**SRE follow-up:** "What if Redis goes down?" → Fall back to in-memory rate limiting per instance (less accurate, but service stays up). Log that rate limiting is degraded.

### 2. Design a URL Shortener
**Core:** Hash collision-free shortcode generation → KV store → redirect.

**Reliability design:**
- Primary DB: PostgreSQL with replicas for reads.
- Cache: Redis caching the most popular short codes (90% of traffic hits 0.1% of URLs — Zipf distribution).
- CDN: Cache redirects at the edge for hot URLs (TTL: 60s to allow updates).
- Tombstoning: Mark deleted URLs as deleted in cache; don't immediately purge (prevents cache stampede).

### 3. Design a Distributed Job Scheduler
**Components:** Job queue (Kafka/SQS), workers (stateless, horizontally scalable), job registry (DB with idempotency key), dead letter queue, retry logic.

**Key reliability properties:**
- **Exactly-once semantics:** Use idempotency keys + DB unique constraints. At-least-once delivery + idempotent workers = effectively exactly-once.
- **Poison pill detection:** After N retries, move to DLQ. Alert on DLQ depth.
- **Job timeout:** Each job has a maximum execution time; a watchdog kills and reschedules overdue jobs.
- **Backpressure:** Queue depth → autoscale workers. Shed load by delaying low-priority jobs when queue is saturated.

### 4. Design a Monitoring / Metrics System
**Components:** Agents → Collection tier → Time-series DB → Query tier → Alerting.

**Push vs. pull:**
- **Pull (Prometheus model):** Collector scrapes targets. Simpler operationally. Targets must be discoverable.
- **Push (StatsD/Graphite model):** Agents push to aggregator. Works behind firewalls. Harder to detect agent death.

**Storage:** Time-series DBs (Prometheus, Thanos, VictoriaMetrics) use columnar storage with delta + delta-of-delta encoding (Gorilla compression). 10× compression vs. raw floats.

**Alerting design:**
- Avoid single-threshold alerts (too many false positives).
- Use multi-window burn rate (Google model): fast burn for critical alerts, slow burn for tickets.
- Alert on SLO burn rate, not raw metrics.

### 5. Design a CDN
**Core:** Origin servers → Edge PoPs (Points of Presence) → Clients.

**Cache miss path:** Edge → Regional cache → Origin. Two-tier caching reduces origin load.

**Cache invalidation:** Hard problem. Options:
- **TTL-based:** Simple, stale data possible.
- **Purge API:** Immediate invalidation by URL pattern. Must handle cache stampede (dog-piling) after purge.
- **Versioned URLs:** `/static/v42/app.js` — never expires, always fresh. Best for static assets.

**Reliability:** Each PoP is N+1 within AZ. If a PoP fails, DNS TTL < 30s routes to next nearest PoP.

### 6. Design a Pub/Sub System
**Components:** Producers → Topics → Partitions → Consumer Groups.

**Kafka model internals:**
- Topic is split into **partitions** (unit of parallelism and ordering).
- Each partition is an append-only log. Consumers track their **offset**.
- Consumer group: each partition assigned to exactly one consumer in the group (horizontal scaling).
- Replication: each partition has one leader + N-1 followers. Leader handles reads/writes.

**Delivery guarantees:**
- **At-most-once:** Consumer commits offset before processing. Loss possible on crash.
- **At-least-once:** Consumer commits offset after processing. Duplicate possible on crash. Make consumers idempotent.
- **Exactly-once:** Kafka transactions (producer + consumer atomicity). Higher complexity.

**Reliability:** What happens when a consumer group member dies? → Partition is re-assigned (rebalance). Gap: messages are not lost (they're in the log), but re-processed from last committed offset.

### 7. Design a Global Service with Multi-Region
**Trade-offs:**
| Strategy | Consistency | Availability | Complexity |
|---|---|---|---|
| Active-Passive (single region writes) | Strong | Lower (failover takes time) | Low |
| Active-Active (local writes, async replication) | Eventual | High | High |
| CRDTs (conflict-free replicated data types) | Eventual | High | Very high |

**SRE angle:** 
- RPO (Recovery Point Objective): How much data loss is acceptable? Determines replication strategy.
- RTO (Recovery Time Objective): How fast must you recover? Determines failover automation.
- Traffic steering: Use anycast or GeoDNS to route users to nearest healthy region. Monitor health with synthetic probes, not just self-reported health.

### 8. Design a Chat System
**Delivery path:** Client → WebSocket server → Message queue → Fan-out workers → Push/Storage.

**Fan-out problem:** A user with 1M followers sends a message → fan-out writes 1M records. Solutions:
- **Write-time fan-out (push):** Works for most users. Cap at 10K followers for write-time.
- **Read-time fan-out (pull):** For "celebrity" accounts, users pull from celebrity's timeline at read time. Instagram's hybrid model.
- **Hybrid:** Write-time for regular users, read-time for accounts above a follower threshold.

**Presence system:** WebSocket heartbeat every 30s. Server-side expiry at 90s. Store presence in Redis with TTL. 10M online users × ~200 bytes = 2 GB — fits in Redis.

### 9. Design a Search Engine (Simplified)
**Indexing path:** Documents → Tokenizer → Inverted index → Ranking → Serving.

**Inverted index:** `term → [(docID, tf, positions)]`. Stored sorted by docID for fast intersection.

**Ranking factors:** TF-IDF (term frequency × inverse document frequency), PageRank, freshness, user signals (CTR, dwell time).

**Reliability:** 
- Index is write-heavy during crawl, read-heavy during serving. Separate serving and indexing tiers.
- Index updates are eventually consistent — new documents appear in search after a crawl + indexing cycle.
- For SRE questions: how do you prevent a crawler from causing DoS on origin servers? → Politeness delay + `robots.txt` + rate limiting per domain.

### 10. Design an Autoscaling System
**Trigger signals:** CPU utilization, queue depth, request latency (p99), custom business metrics.

**Scale-out vs. scale-in:** Scale out aggressively (1 minute), scale in conservatively (10 minutes). Asymmetric cooldown prevents thrashing.

**Predictive autoscaling:** Use historical traffic patterns (daily/weekly seasonality) to scale before the spike, not during it.

**Kubernetes HPA + KEDA:** HPA scales on CPU/memory. KEDA adds custom metrics (Kafka queue depth, SQS queue length, Redis queue). [[k8s/concepts/hpa]]

---

## Distributed Systems Fundamentals (Interview Quick Reference)

### CAP Theorem
A distributed system can guarantee at most two of: **Consistency, Availability, Partition tolerance**. In practice, partition tolerance is required (networks fail), so the real trade-off is **CP vs. AP**.

| System | Guarantee | Example |
|---|---|---|
| CP | Consistent during partition; may refuse writes | HBase, Zookeeper, etcd |
| AP | Available during partition; stale reads possible | Cassandra, DynamoDB, CouchDB |

**Interview trap:** "SQL is always CP" — wrong. PostgreSQL with async replicas is AP (stale reads from replicas).

### Consistency Models (ordered weakest → strongest)
1. **Eventual consistency** — all replicas converge eventually (Cassandra)
2. **Monotonic reads** — once you read a value, you never read older versions
3. **Read-your-writes** — you always see your own writes (session consistency)
4. **Causal consistency** — if A causes B, everyone sees A before B
5. **Sequential consistency** — all operations appear in a global order
6. **Linearizability (strong consistency)** — reads always reflect the most recent write globally

### Failure Patterns and Mitigations
| Failure | Symptom | Mitigation |
|---|---|---|
| Thundering herd | Cache miss causes DB spike | Cache stampede lock; probabilistic early expiry |
| Cascading failure | One slow service backs up all callers | Circuit breaker; bulkhead; timeout |
| Split brain | Two nodes both think they're primary | Consensus (Raft/Paxos); quorum reads/writes |
| Hot partition | One DB shard gets all traffic | Shard on different key; virtual nodes; consistent hashing |
| Head-of-line blocking | Slow request blocks queue behind it | Request hedging; parallel fanout; timeout per request |

---

## Apple-Specific SRE Design Patterns

### Privacy-by-Design Architecture
- **Data minimization:** Log request metadata (timestamps, error codes), never log content (file names, message bodies).
- **On-device processing:** Move computation to the device (ML inference, search indexing) to avoid sending raw user data to servers.
- **Encryption in transit and at rest:** TLS 1.3 minimum; AES-256 at rest; per-user encryption keys for highest-sensitivity data.
- **Differential privacy:** Add calibrated noise to aggregate analytics to prevent re-identification.

### iCloud Scale Design Patterns
- **Consistency model:** iCloud uses a variant of "hub-and-spoke" with a central reconciliation service. Conflict resolution is timestamp-based with user-visible "keep both" fallback.
- **CDN strategy:** Apple operates its own CDN (Akamai + Apple-owned PoPs). Software updates use the same CDN infrastructure — a macOS release is one of the largest CDN events in the world.

## Google-Specific SRE Design Patterns

### NALSD (Non-Abstract Large Scale Design)
Every Google design must answer three capacity questions:
1. **Compute:** How many cores? At what utilization?
2. **Storage:** How many bytes? At what replication factor?
3. **Bandwidth:** Bytes/sec in and out? Can the network sustain it?

Useful numbers to memorize:
| Operation | Approximate latency |
|---|---|
| L1 cache access | 1 ns |
| L2 cache access | 4 ns |
| RAM access | 100 ns |
| SSD random read | 100 μs |
| HDD random read | 10 ms |
| Network round trip (same AZ) | 500 μs |
| Network round trip (cross-region) | 80–150 ms |

## Meta-Specific SRE Design Patterns

### The Two-Tier Cache (McRouter + Memcache)
Meta's cache layer uses **McRouter** (a Memcache proxy) for routing, replication, and failover. Key insight:
- Each region has a Memcache pool.
- McRouter routes reads to local pool, writes to all pools (write-through).
- On cache miss, a **lease** is issued (prevents thundering herd). Only the lease holder fetches from DB.

### TAO (The Associations and Objects store)
Meta's primary data store for social graph data. Key design:
- Read-optimized: ~99% reads, ~1% writes.
- Two tiers: **leader** (consistent writes) and **follower** (read scale, eventual consistency).
- Invalidation: write to leader → invalidate follower caches via a separate invalidation bus.

---

## Sources

- [[sre/concepts/slo-sli-sla]]
- [[sre/concepts/load-balancers]]
- [[sre/concepts/networking-fundamentals]]
- [[sre/companies/google]]
- [[sre/companies/apple]]
- [[sre/companies/meta]]
