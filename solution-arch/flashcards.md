# Solution Architecture Flashcards (Anki Format)

> Import into Anki using the "Basic" note type. Each `## Q:` / `**A:**` block = one card.
> Or use the Obsidian Anki Sync plugin.

---

## Fundamentals

## Q: What is the difference between Availability and Reliability in system design?
**A:**
- **Availability:** Is the system responding? (% uptime)
- **Reliability:** Is it giving correct results?

A system can be available (returns 200 OK) but unreliable (returns wrong data). Both are needed; they require different mitigations — availability needs redundancy, reliability needs checksums and idempotency.

---

## Q: What do "four nines" and "five nines" of availability mean in actual downtime?
**A:**
- **99.99% (four nines):** ~52 minutes downtime per year / ~4.4 min per month
- **99.999% (five nines):** ~5.3 minutes per year / ~26 seconds per month

Four nines is achievable with multi-AZ architecture. Five nines requires active-active multi-region and extensive operational investment.

---

## Q: What is the NFR interview checklist a Solution Architect runs through before designing?
**A:**
1. **Scale:** DAU, RPS, read:write ratio, data volume
2. **Availability:** SLA uptime requirement
3. **Consistency:** strong or eventual?
4. **Latency:** P50/P99 target (user-facing vs background)
5. **Durability:** can acknowledged writes be lost?
6. **Security:** PII, regulated data, access model

---

## Q: What is an Architecture Decision Record (ADR)?
**A:** A short document capturing a significant architectural decision: Context (why the decision needed to be made), Decision (what was decided), Consequences (trade-offs, pros/cons), and Alternatives considered. Stored in version control alongside code. Enables future developers to understand why the system is designed as it is.

---

## Q: What is Conway's Law and how does it affect microservices?
**A:** "Any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure." — Mel Conway

Implication for microservices: if three teams work on one codebase, you get a three-module design. For effective microservices, structure teams around domain services (stream-aligned teams) so each team owns one service end-to-end.

---

## CAP Theorem

## Q: State the CAP theorem and what the practical trade-off is.
**A:** In a distributed system, you can guarantee at most 2 of: **Consistency** (every read gets the latest write), **Availability** (every request gets a non-error response), **Partition Tolerance** (works despite network failures).

Since partitions always happen in real distributed systems, the real choice is **C vs A** during a partition: either return an error (CP) or return possibly-stale data (AP).

---

## Q: Give a real example where you would choose CP over AP, and vice versa.
**A:**
- **CP:** Bank balance check. Showing stale balance could allow overdraft. Better to return an error than wrong data.
- **AP:** Social media feed. Showing a post from 2 seconds ago is acceptable. Availability > perfect freshness.
- **AP:** Product catalogue. Brief price staleness is fine; catalogue must be reachable.

---

## Q: What is PACELC and how does it improve on CAP?
**A:** PACELC extends CAP: **P**artition → **A**vailability vs **C**onsistency; **E**lse (no partition) → **L**atency vs **C**onsistency.

Even without a partition, a distributed system must trade latency (fast local reads) against consistency (slow coordination). CAP ignores this normal-operation trade-off. PACELC captures the full picture.

---

## Caching

## Q: Explain Cache-Aside vs Write-Through caching strategies.
**A:**
- **Cache-Aside (Lazy):** App checks cache; miss → query DB → populate cache → return. Cache only holds requested data. Cache failure is non-fatal.
- **Write-Through:** App writes to cache and DB synchronously on every write. Cache always consistent with DB. Higher write latency; cache may hold unread data.

Use cache-aside for read-heavy workloads. Write-through when reads always follow writes.

---

## Q: What is cache stampede (thundering herd) and how do you prevent it?
**A:** When a popular cached item expires, many simultaneous requests miss the cache and all hit the database simultaneously, overwhelming it.

Prevention:
1. **Mutex/lock:** only one request fetches from DB; others wait
2. **Probabilistic early expiry:** randomly expire items slightly before TTL to stagger refreshes
3. **Async refresh:** background process refreshes before expiry; serve stale until refreshed
4. **Jitter on TTL:** add random offset to TTLs to stagger expirations across the fleet

---

## Q: What is a hot key in Redis and how do you mitigate it?
**A:** A hot key is a single cache key that receives an overwhelming proportion of requests (e.g., a celebrity's profile, a viral product listing).

Mitigations:
1. **Local in-process cache** (L1 before Redis): serve from memory for hottest keys
2. **Key sharding:** store as `key:0`, `key:1`, ..., `key:N`; random read from any shard
3. **Read replicas:** Redis replication; route reads across replicas

---

## Load Balancing

## Q: What is the difference between L4 and L7 load balancing?
**A:**
- **L4 (Transport):** Routes by IP:port. Cannot inspect HTTP content. Faster. No SSL termination. Examples: AWS NLB, HAProxy TCP.
- **L7 (Application):** Routes by URL path, headers, cookies. Can do SSL termination, sticky sessions, path-based routing. Slower but smarter. Examples: AWS ALB, nginx, Envoy.

Use L4 for raw throughput; L7 for smart routing across microservices.

---

## Q: What is the "thundering herd" problem when a load balancer adds a new node?
**A:** When a new node joins (e.g., after auto-scaling), the load balancer might route a disproportionate initial burst of traffic to it before it's warm (caches empty, JIT not compiled). This makes the new node slow, and may cascade.

Mitigation: **Gradual ramp-up** — start new instance at 0% weight, increase gradually over 1–2 minutes as it warms up.

---

## Messaging

## Q: What are the three message delivery guarantees? Which is hardest to achieve?
**A:**
- **At-most-once:** Send once; may be lost on failure. Easiest.
- **At-least-once:** Retry until acked; may be duplicated. Common default.
- **Exactly-once:** Delivered and processed exactly once. Hardest — requires idempotent consumers + transactional producers.

In practice: use **at-least-once + idempotent consumers** = effectively exactly-once.

---

## Q: When would you use a queue vs a database for storing jobs?
**A:**
- **Queue (Kafka/SQS):** Natural at-least-once delivery, built-in retry, backpressure, fan-out, consumer groups. Best for: background jobs, event-driven workflows.
- **Database (job table):** Flexible queries, ACID guarantees, easy to monitor. Best for: scheduled jobs, jobs needing complex state transitions, audit requirements.

Rule of thumb: if you need to query/filter pending jobs by arbitrary criteria → DB. If you need high-throughput, simple enqueue/dequeue → queue.

---

## Q: What is Kafka consumer lag and why does it matter?
**A:** Consumer lag = difference between the latest offset in a partition and the consumer group's committed offset. It measures how far behind a consumer is.

Why it matters: high lag = consumers are not keeping up with producers. If lag grows unboundedly, the system is overloaded. SLO: if lag > X messages or Y minutes, alert — backlog may affect downstream freshness or fill disk.

---

## Patterns

## Q: Explain the Circuit Breaker pattern and its three states.
**A:** Prevents cascading failures by stopping calls to a failing service.

**CLOSED:** Normal operation. All calls pass through. Count failures.
**OPEN:** Threshold exceeded. All calls fail immediately (return error/fallback). No calls to dependency. Timer runs.
**HALF-OPEN:** After timeout, allow a probe request. Success → CLOSED. Failure → OPEN again.

---

## Q: What is the Saga pattern and what are its two styles?
**A:** Manages distributed transactions across microservices via a sequence of local transactions, each with a compensating transaction.

**Choreography:** Services emit events and react to each other's events. Decoupled; no central coordinator. Hard to trace.

**Orchestration:** Central Saga Orchestrator tells each service what to do. Clear flow; easy to debug. Orchestrator could become a bottleneck.

---

## Q: What is the Outbox Pattern and what problem does it solve?
**A:** Solves the **dual-write problem**: ensuring a DB write and a message publish are atomic without 2PC.

Implementation: In the same DB transaction, write to the business table AND an outbox table. A relay process (polling or CDC) reads the outbox and publishes to the message broker, then marks entries published.

Guarantee: at-least-once delivery. Consumers must be idempotent.

---

## Q: What is the Strangler Fig pattern?
**A:** Incrementally replaces a legacy system by building a new system alongside it. A facade routes requests to either old or new based on which functionality has been migrated. The old system is "strangled" as more traffic moves to the new one, until the old system can be decommissioned.

Key rules: Never rewrite while migrating. Facade must be transparent to clients. Exit criteria per domain.

---

## Q: What is the BFF (Backend for Frontend) pattern?
**A:** A dedicated aggregation layer per client type (mobile, web, partner). Each BFF calls multiple backend services and returns exactly the data that specific client needs — no over-fetching, no under-fetching.

When to create a BFF: clients have fundamentally different data needs, different auth requirements, or are owned by different teams. Not worth creating if clients need nearly identical data.

---

## Q: What is the difference between Blue-Green and Canary deployments?
**A:**
- **Blue-Green:** Two identical environments. Deploy to idle (Green), test, then switch 100% traffic instantly. Rollback = instant switch back. Cost: 2× infra.
- **Canary:** Gradually shift traffic (5% → 25% → 100%) monitoring metrics at each stage. Auto-rollback if error rate spikes. Cost: minimal infra overhead. Requires observability.

Use Blue-Green for major schema/version changes needing instant rollback. Use Canary for continuous delivery with risk monitoring.

---

## Data Architecture

## Q: What is the difference between OLTP and OLAP and why must they be separate?
**A:**
- **OLTP:** Many short transactions (ms), writes + reads, normalised, current data. e.g., PostgreSQL.
- **OLAP:** Few long analytical queries (seconds-minutes), read-heavy, denormalised, historical. e.g., BigQuery, Snowflake.

They must be separate because analytical queries (full table scans, complex aggregations) hold locks, consume CPU, and degrade transaction performance on OLTP databases. Use a data warehouse or CQRS read models for analytics.

---

## Q: What is CQRS and when would you use it?
**A:** Command Query Responsibility Segregation separates write (command) and read (query) paths into distinct models, often backed by different stores.

Use when:
- Read patterns are very different from write patterns
- Need multiple read models (dashboard, mobile, search)
- Read and write sides need to scale independently
- An audit/event log is needed

Don't use for simple CRUD apps — unnecessary complexity.

---

## Q: What is Event Sourcing and how does it differ from a traditional audit table?
**A:** Event Sourcing stores the full sequence of events that led to current state, making events the **primary source of truth**. Current state is derived by replaying events.

Traditional audit table: a changelog appended alongside a mutable state table. The mutable state is still the source of truth; the audit table is secondary.

Key difference: in event sourcing you can reconstruct any past state by replaying up to that point. With a traditional audit table, you cannot (you only know what changed, not the full context).

---

## Q: What are the consistency models from strongest to weakest?
**A:**
1. **Linearisability:** Every read sees the most recent write globally. Single-machine feel. High latency.
2. **Sequential consistency:** All nodes see ops in same order; may lag real time.
3. **Causal consistency:** Causally-related ops appear in order; concurrent ops may differ.
4. **Eventual consistency:** All replicas will converge given no new writes. Stale reads possible.

Money → linearisable. Social feed → eventual consistency is fine.

---

## Security

## Q: What is Zero Trust Architecture?
**A:** "Never trust, always verify." Every request is authenticated and authorised regardless of network origin — even internal traffic. No implicit trust for being "inside the network."

Pillars:
1. Verify explicitly (identity + device + context for every request)
2. Least privilege (minimal access, time-limited)
3. Assume breach (segment networks; monitor east-west traffic)

Enabled by: mTLS between services (service mesh), RBAC/ABAC, short-lived tokens, comprehensive audit logging.

---

## Q: What is the STRIDE threat model?
**A:**
- **S**poofing — impersonating another user/system → AuthN, mTLS
- **T**ampering — modifying data in transit/rest → integrity checks, HMAC
- **R**epudiation — denying an action was taken → audit logs, signing
- **I**nformation disclosure — data leakage → encryption, access control
- **D**enial of Service — overwhelming the system → rate limiting, WAF
- **E**levation of privilege — gaining unauthorised higher access → RBAC, least privilege

---

## Q: What is the difference between mTLS and regular TLS?
**A:**
- **TLS (one-way):** Client verifies server's certificate. Server doesn't verify client identity. Standard HTTPS.
- **mTLS (mutual):** Both sides present and verify certificates. Client AND server authenticated. Used for service-to-service authentication in microservices/service mesh.

mTLS eliminates the need for API keys between services — identity proven cryptographically.

---

## Scalability

## Q: Explain consistent hashing. Why is it better than modular hashing for distributed caches?
**A:** Nodes and keys are hashed to positions on a virtual ring (0–2³²). A key maps to the nearest node clockwise.

Why better than `hash(key) % N`:
- Adding/removing 1 node → only `~1/N` keys reassign (not all keys)
- Prevents cache stampede when scaling
- Used by: Redis Cluster, Cassandra, DynamoDB, Memcached

---

## Q: When would you choose sharding over read replicas for database scaling?
**A:**
- **Read replicas:** Write bottleneck not yet hit; read-heavy workload. Easier; data stays on one primary.
- **Sharding:** Write throughput exceeds single-node capacity OR dataset doesn't fit on single disk OR true horizontal write scaling needed.

Order of escalation: index tuning → caching → read replicas → vertical scale → sharding.
Sharding is last resort — adds significant complexity (cross-shard queries, resharding, hot spots).

---

## Q: What is the hot spot problem in sharding and how do you fix it?
**A:** A shard key that concentrates access on one shard (e.g., celebrity user_id, trending product) makes that shard a bottleneck.

Fixes:
1. **Compound key:** `(user_id, random_suffix)` distributes hot user across N virtual shards
2. **Dedicated shard:** give known hot entities their own shard
3. **Application-layer cache:** cache hot key aggressively in Redis, reducing DB hits
4. **Read replicas per shard:** scale reads on the hot shard specifically

---

## Q: What is Little's Law and how do you apply it to size a thread pool?
**A:** **L = λ × W** — average number of items in system = arrival rate × average time spent.

Thread pool sizing:
  `threads_needed = requests_per_second × average_latency_seconds`

Example: 100 req/sec, P99 latency 200ms → `100 × 0.2 = 20 concurrent requests` → set pool to 20–25 threads.

Undersized pool: requests queue → latency grows → cascading failure. Oversized: wasted resources.

---

## Q: What is the expand-migrate-contract pattern for zero-downtime DB schema changes?
**A:** A three-phase approach to changing DB schema while keeping both old and new app versions compatible:

1. **Expand:** Add new column/table (nullable, no constraint). Deploy app writing to both old and new schema.
2. **Migrate:** Backfill existing rows. Validate data quality.
3. **Contract:** Remove old column after all traffic is on new app version.

Never rename or drop a column in the same deployment that switches traffic — always spread across 3 separate deploys.
