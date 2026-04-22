# Solution Architecture — Scenario Interview Questions

**Related:** [[solution-arch/scenarios/design-url-shortener]], [[solution-arch/scenarios/design-rate-limiter]], [[solution-arch/scenarios/design-notification-system]], [[solution-arch/scenarios/monolith-to-microservices]], [[solution-arch/scenarios/high-availability-platform]]

---

## How to Answer System Design Questions

```
1. Clarify requirements (5 min)
   ├── Functional: what must the system do?
   └── Non-functional: scale, latency, availability, consistency

2. Estimate scale (2 min)
   ├── DAU / MAU → requests per second
   ├── Storage per day / year
   └── Read:write ratio

3. High-level design (10 min)
   ├── Identify major components
   ├── Draw Level 2 C4 diagram
   └── Define APIs (endpoints or event schemas)

4. Deep dive (10 min)
   └── Interviewer picks one area; go deeper

5. Trade-offs (5 min)
   └── What you'd do differently at different scales
```

---

## Fundamentals

### Q1: What are the trade-offs between microservices and a monolith?

**A:**

| | Monolith | Microservices |
|--|---------|--------------|
| Complexity | Low (one codebase) | High (distributed system) |
| Deploy | One artefact | Independent per service |
| Scale | As a unit | Per service |
| Data | Single DB | Per-service DB |
| Failure | One service down = all down | Partial failure possible |
| Team | Single team | Multiple autonomous teams |
| Latency | In-process calls | Network calls |
| Debugging | Unified traces | Distributed tracing needed |

**Answer framework:** "Start with a monolith until you have clear bounded contexts, scaling needs, or team ownership boundaries that justify the operational cost of microservices."

---

### Q2: Your API P99 latency is 2 seconds. How do you diagnose and fix it?

**A: Methodical approach:**

```
1. Instrument first (don't guess)
   → Distributed tracing (Jaeger/Zipkin): where is time spent?
   → Flame graphs: which functions are slow?
   → DB slow query log: expensive queries?

2. Common culprits:
   → N+1 queries: query in a loop → single JOIN/batch query
   → Missing index: full table scan → add index
   → Synchronous external API call in hot path → async or cache result
   → Memory pressure causing GC pauses → profiler
   → Lock contention in DB → shorter transactions

3. Fixes by layer:
   → Application: cache expensive computations (Redis)
   → DB: index tuning, query optimisation, read replicas
   → Network: CDN for static, co-locate services in same AZ
   → Architecture: async non-critical work (email, analytics)
```

---

### Q3: How do you design for 99.99% availability?

**A:**

```
Eliminate all SPOFs:
  ✅ Multiple LB instances (anycast or DNS)
  ✅ Multiple app instances (horizontal scale)
  ✅ DB primary + standby (auto failover)
  ✅ Multi-AZ deployment (one AZ fails → traffic shifts)
  ✅ Multi-region (extreme: active-active or active-passive)

Graceful degradation:
  ✅ Circuit breakers on all external dependencies
  ✅ Return cached stale data if live is unavailable
  ✅ Feature flags to disable non-critical features under load

Operational:
  ✅ Health checks + auto-restart (Kubernetes liveness probes)
  ✅ Chaos engineering: inject failures to validate resilience
  ✅ Runbooks for every alert
  ✅ Canary deployments (don't ship broken code to 100% at once)
  ✅ SLOs with error budgets: quantify acceptable downtime
```

---

### Q4: How does consistent hashing work and when would you use it?

**A:**

```
Normal hash: shard = hash(key) % N
  Problem: adding/removing a node → almost all keys reassign → cache stampede

Consistent hash ring:
  Nodes and keys both hashed to position on 0–2^32 ring
  Key maps to nearest node clockwise
  
  Ring: 0 ────────── A ──────────── B ──────────── C ──── 2^32
        key1 → A    key2 → B    key3 → C

Adding node D between A and B:
  Only keys between A and D move from B to D
  All other keys unaffected (1/N keys move, not all)
```

**Uses:** distributed caches (Redis Cluster, Memcached), load balancing with session affinity, distributed databases (Cassandra, DynamoDB), CDN routing.

---

### Q5: Explain CAP theorem. Can you give a real example where you chose CP over AP?

**A:**

```
CAP: During a network partition, choose:
  C — refuse to serve (return error or wait for consistency)
  A — serve possibly stale data

Real example: Online banking balance check
  Chose CP (Zookeeper + PostgreSQL with synchronous replication)
  Reason: showing stale balance → customer could overdraft
  Trade-off: service returns error during partition rather than stale data
  Mitigated by: small partition window (< 30s before failover), retry logic in client

Counter-example: Product review display
  Chose AP (DynamoDB default)
  Reason: showing review from 2 seconds ago is acceptable
  Trade-off: brief inconsistency for higher availability
```

---

### Q6: How do you handle distributed transactions without 2PC?

**A: Saga pattern:**

```
1. Break distributed transaction into local transactions
2. Each local transaction publishes an event or message
3. If any step fails, execute compensating transactions in reverse

Two styles:
  Choreography: services react to each other's events (decoupled, hard to trace)
  Orchestration: central coordinator tells each service what to do (visible, easy to debug)

Example: Order placement
  Step 1: Create order (Order Service)
  Step 2: Reserve stock (Inventory Service)
  Step 3: Charge payment (Payment Service)
  Step 4: Schedule shipping (Shipping Service)

  If Step 3 fails:
    Compensate Step 2: release stock reservation
    Compensate Step 1: cancel order
    Notify user: payment failed
```

---

### Q7: How do you scale a relational database?

**A: Escalating strategies:**

```
Level 1: Optimise the single node
  → Index tuning, query optimisation
  → Connection pooling (PgBouncer)

Level 2: Read replicas
  → Route reads to replicas, writes to primary
  → Async replication; replicas may lag

Level 3: Caching
  → Redis/Memcached in front of DB
  → Cache hit rate > 95% dramatically reduces DB load

Level 4: Vertical scale
  → Bigger server (limits: cost, hardware ceiling)

Level 5: Sharding
  → Horizontal partition by shard key
  → Last resort: complex cross-shard queries, resharding

Level 6: NewSQL / distributed SQL
  → CockroachDB, PlanetScale, Spanner
  → Horizontal scale + ACID, managed resharding
```

---

### Q8: Describe the outbox pattern and why it's needed.

**A:**

```
Problem: dual write — update DB and publish event must be atomic
  But DB and message broker are separate systems → two-phase commit is too costly

Without outbox:
  Write to DB (success)
  App crashes
  Publish to Kafka (never happens) → downstream never notified

With outbox:
  Single transaction:
    INSERT into orders table
    INSERT into outbox table (same DB transaction)
  Relay process:
    Poll outbox table → publish to Kafka → mark published

  Atomicity guaranteed: both inserts succeed or both fail
  At-least-once delivery: relay retries on failure (consumers must be idempotent)
```

---

### Q9: How would you design a system to handle 1 million websocket connections?

**A:**

```
WebSocket Challenge: stateful long-lived connections (can't use stateless HTTP LB easily)

Architecture:
  Client ──▶ L4 Load Balancer (sticky by IP or connection ID)
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
    WS Server A          WS Server B
    [500k conns]         [500k conns]
          │                    │
          └──────┬─────────────┘
                 ▼
          Message Broker (Redis Pub/Sub or Kafka)
          (route messages between servers to reach any client)

Message fan-out:
  Server A has connection for user 1234
  Server B wants to send message to user 1234
  Server B publishes to Redis channel for user 1234
  Server A subscribed to user 1234's channel → delivers to connection

Scale:
  Each WS server: ~50k connections (Node.js) or ~250k (Go)
  Horizontal scale: add more WS servers
  Redis Pub/Sub: handles routing across servers
```

---

### Q10: How do you design for idempotency in a payment API?

**A:**

```
Problem: payment API called twice (retry, double-click) → double charge

Solution:
  Client generates idempotency key (UUID) per payment intent
  
  POST /payments
  Idempotency-Key: "pay-abc123"
  { amount: 99.00, card_token: "..." }

  Server:
    1. BEGIN TRANSACTION
    2. SELECT FOR UPDATE FROM idempotency_keys WHERE key = 'pay-abc123'
    3. IF found: RETURN stored_response (no charge)
    4. IF not found:
         Process payment with provider
         INSERT idempotency record with response
         COMMIT
    5. RETURN response

  Key details:
    - Lock prevents two concurrent identical requests both processing
    - TTL on idempotency key: 24 hours (after that, same key could be new intent)
    - Idempotency key is the client's responsibility to generate
```

---

### Q11: You need to migrate a monolith to microservices. Walk me through your approach.

**A:** → Full walkthrough: [[solution-arch/scenarios/monolith-to-microservices]]

---

### Q12: How would you design a global content delivery network (CDN)?

**A:**

```
Components:
  1. Origin servers (where actual content lives)
  2. Edge PoPs (Points of Presence) — datacentres close to users
  3. DNS with GeoDNS — route user to nearest PoP

Request flow:
  User (Tokyo) ──▶ DNS resolves to Tokyo PoP
  Tokyo PoP ──HIT──▶ serve from cache (< 5ms)
  Tokyo PoP ──MISS──▶ fetch from origin (US) ──cache──▶ serve (~150ms first time)

Cache hierarchy:
  L1: Edge PoP (smallest, fastest, user-facing)
  L2: Regional cache (larger, serves multiple edges)
  L3: Origin shield (protect origin from cache-busting storms)

Cache invalidation:
  Versioned URLs: /static/app.v2.4.1.js (never expires)
  Cache-Control headers per content type
  API: POST /cdn/invalidate { paths: ["/products/123"] }

Challenges:
  Thundering herd on invalidation: many edges fetch origin simultaneously
    → Use lock/semaphore at regional cache: one edge fetches, others wait
  Dynamic content: personalised data can't be cached naively
    → Edge computing (Cloudflare Workers) to personalise at edge
```

---

### Q13: Design a rate limiter that works across a cluster of API servers.

**A:** → Full walkthrough: [[solution-arch/scenarios/design-rate-limiter]]

---

### Q14: How would you handle a sudden 10× traffic spike?

**A:**

```
Short-term (seconds-to-minutes):
  Auto-scaling triggers (if configured):
    → New instances launch (takes 30–120 seconds)
  Circuit breakers open on degraded services
  Rate limiting: shed excess traffic gracefully (429 responses)
  Caches absorb reads (no DB increase)
  CDN handles static assets (no origin hit)

Medium-term (hours):
  Increase read replicas for DB
  Provision additional cache nodes
  Route some traffic to cheaper/different region

Architectural guardrails (pre-emptive):
  Load test to know your ceiling
  Auto-scaling on CPU/RPS metrics (scale before saturation)
  Graceful degradation: non-critical features return cached/empty
  Queue async work (email, analytics): shed to queue, process later
  Static asset CDN: removes significant origin load

Communication:
  Alert on-call SRE when scaling events occur
  Status page update if user-facing impact
```

---

### Q15: Compare SQL vs NoSQL — when to use each?

**A:**

```
Use SQL (PostgreSQL, MySQL) when:
  ✅ Data is relational (entities with relationships)
  ✅ ACID transactions required
  ✅ Schema is stable
  ✅ Complex queries with joins/aggregations
  ✅ Team familiar with SQL

Use NoSQL when:
  ✅ Document: MongoDB — flexible/nested schema, content management
  ✅ Key-value: Redis, DynamoDB — simple lookups, sessions, cache
  ✅ Wide-column: Cassandra — time-series, write-heavy, high throughput
  ✅ Graph: Neo4j — relationship traversal (social graph, fraud detection)
  ✅ Search: Elasticsearch — full-text, faceted search

Mixed patterns (common in production):
  PostgreSQL for transactions
  + Redis for cache/sessions
  + Elasticsearch for search
  + Kafka for events
```

---

### Q16: How does a message queue improve system resilience?

**A:**

```
Without queue:
  Order Service ──▶ Email Service (sync HTTP)
  
  Email Service down → Order Service fails (cascading)
  Email Service slow → Order Service slow (coupling)

With queue:
  Order Service ──▶ Queue ──▶ Email Service (async)
  
  Email Service down → messages accumulate; Email processes when back up
  Email Service slow → queue absorbs the spike
  Order Service: always returns 202 Accepted immediately
  
  Benefits:
    Temporal decoupling: producer/consumer don't run simultaneously
    Load levelling: queue smooths traffic spikes
    Retry: dead letter queue captures failures for inspection
    Backpressure: producer doesn't overwhelm consumer
```

---

### Q17: What is the strangler fig pattern and when would you use it?

**A:** → Full walkthrough: [[solution-arch/patterns/strangler-fig]]

---

### Q18: How do you design a system that must guarantee exactly-once message processing?

**A:**

```
True exactly-once is very hard. Practical approach:
  "At-least-once delivery + idempotent consumers = effectively exactly-once"

Producer side (at-least-once):
  Use transactional outbox: write DB + publish event atomically
  Relay retries until acked by broker

Consumer side (idempotent):
  Track processed message IDs:
    BEGIN TRANSACTION
    IF message_id IN processed_ids: skip and ack
    ELSE: process, insert into processed_ids, commit, ack

  Deduplication window: keep IDs for message retention window (7 days)

Kafka exactly-once (within Kafka):
  Producer: enable.idempotence=true + transactions
  Consumer: read_committed isolation, manual offset commit in same transaction as DB write
  Cost: 20-30% throughput reduction
```

---

### Q19: Explain Blue-Green vs Canary deployment. When would you use each?

**A:** → Full details: [[solution-arch/patterns/blue-green-canary]]

**Short answer:**
- **Blue-Green:** Instant, atomic cutover. Best for: major version changes, migrations with DB schema changes. Requires: 2× infra.
- **Canary:** Gradual traffic shift with monitoring. Best for: continuous delivery, risk-averse changes. Requires: observability stack to detect regressions.

---

### Q20: How do you prevent a slow microservice from taking down the entire system?

**A: Three-layer defence:**

```
1. Timeouts (first line)
   All network calls must have explicit timeouts
   Never wait forever
   
2. Circuit Breaker (second line)
   After N failures: fail fast without calling the service
   Protects: call stack, thread pools
   Auto-recovers when service heals

3. Bulkhead (third line)
   Separate thread pools per dependency
   Slow service fills its own pool; other services unaffected

4. Fallback (fourth line)
   Circuit open → return cached data, default response, or degrade feature
   Never return an unhandled error to user

Combined:
  Request ──▶ [Bulkhead: pool limit] ──▶ [Circuit Breaker] ──▶ Service
                                              │ open
                                              ▼
                                         [Fallback: cached result]
```

---

## Microservices Architecture

### Q21: What is a "distributed monolith" and why is it the worst of both worlds?

**A:**
```
A distributed monolith = services deployed separately but tightly coupled via:
  - Shared database (all services read/write the same DB schema)
  - Synchronous chain dependencies (A→B→C→D all sync, co-deploy required)
  - Shared library with business logic (all services must update together)

Worst of both worlds:
  Monolith downside:  Can't deploy one service without coordinating others
  Microservices cost: Network latency, operational overhead, distributed debugging

Root cause: drew service boundaries at technical layers (Data/Logic/API)
            instead of business capabilities (Orders/Users/Payments)

Fix: 
  1. Enforce database-per-service (no cross-service DB access)
  2. Replace sync chains with async events for non-critical paths
  3. Re-draw boundaries by business capability (Event Storming)
```

---

### Q22: How do you decide on the right service boundaries?

**A:**
```
Three complementary techniques:

1. Decompose by Business Capability
   "What does the business do?" → each answer is a candidate service
   Orders, Payments, Inventory, Notifications, User Accounts

2. Decompose by Bounded Context (DDD)
   "Domain" has sub-domains; each bounded context = one service
   Each context has its own ubiquitous language
   "Order" in Sales context ≠ "Order" in Warehouse context

3. Decompose by Volatility
   Things that change together → keep together
   Things that change independently → separate services

Sizing heuristic (Sam Newman):
  "A service you could rewrite in 2 weeks"
  One team can own, understand, and maintain it
  Most operations don't cross service boundaries

Red flags for wrong boundaries:
  Every feature touches 3+ services → probably too granular
  Two services always deploy together → probably should be one
  Service is a "data service" (just CRUD, no business logic) → wrong cut
```

---

### Q23: How does a service discover the address of another service?

**A:**
```
Two approaches:

Client-Side Discovery:
  Service queries Registry (Consul, Eureka) directly
  Client picks an instance and calls it
  Pros: client controls LB algorithm
  Cons: needs per-language SDK; more client complexity

Server-Side Discovery (Kubernetes default):
  Client calls a stable virtual address (Kubernetes Service ClusterIP)
  kube-proxy (or service mesh) routes to a healthy pod
  DNS: payment-service.default.svc.cluster.local → ClusterIP
  Pros: client is simple; no SDK needed
  Cons: extra hop (proxy)

Health checking:
  Registry polls /health endpoint every N seconds
  Unhealthy instance removed from registry → traffic stops routing there
  Kubernetes: liveness probe (restart if unhealthy) + readiness probe (remove from LB)
```

---

### Q24: How do you query data that belongs to two different services without a shared database?

**A:**
```
Three patterns (choose based on requirements):

1. API Composition (synchronous):
   Order Service calls User Service API at query time
   Merge results in memory
   Pros: simple, fresh data
   Cons: latency adds up; coupled to User Service availability
   Use when: simple join, low volume, low latency tolerance

2. CQRS Read Model (async materialised view):
   User Service publishes UserUpdated events
   Order Service consumes events → maintains local copy of user names
   Client queries Order Service's read model (pre-joined data)
   Pros: decoupled, fast reads, resilient
   Cons: eventual consistency, data duplication
   Use when: complex queries, high volume, resilience needed

3. Data Warehouse (analytics only):
   Both services publish to Kafka → pipeline → warehouse
   Analytics queries hit warehouse (read-only, minutes latency)
   Never use as a back door for production writes
```

---

### Q25: What is the Service Mesh and what problems does it solve in microservices?

**A:**
```
A service mesh handles east-west (service-to-service) traffic via sidecar proxies
injected alongside each service — without changing application code.

Problems it solves:
  mTLS:          Encrypts + authenticates all inter-service traffic automatically
  Observability: Per-request metrics and traces for every service pair
  Traffic mgmt:  Canary deployments, retries, timeouts via config (not code)
  Circuit breaking: Outlier detection removes unhealthy instances from LB pool
  Access policy: Service A is allowed to call Service B; Service C is not

How it works:
  Sidecar proxy (Envoy) intercepts all traffic to/from the pod
  Control plane (Istiod) distributes routing rules and rotates mTLS certs
  App code never sees: TLS, retry logic, timeout config

vs API Gateway:
  API Gateway: north-south (external → cluster); AuthN, rate limiting
  Service Mesh: east-west (service → service); mTLS, observability, retries
  Both are needed; they are complementary
```

---

### Q26: Walk through how the Saga pattern handles a failed distributed transaction.

**A:**
```
Scenario: Order placement (Order → Reserve Stock → Charge Payment → Ship)
  Step 3 (Payment) fails.

Orchestration Saga:
  Orchestrator calls:
    1. Order Service:     CREATE order #123           ✓
    2. Inventory Service: RESERVE 5 units of SKU-A    ✓
    3. Payment Service:   CHARGE $99                  ✗ FAILED

  Orchestrator triggers compensations in reverse:
    2. Inventory Service: RELEASE reservation (SKU-A, 5 units)
    1. Order Service:     CANCEL order #123

  User notified: "Payment failed, order cancelled"

Key requirements:
  Each forward step: idempotent (safe to retry)
  Each compensating step: idempotent (safe to call multiple times)
  Orchestrator state: persisted (can resume after crash)
  Eventual consistency: order may briefly appear "pending" before cancellation visible
```

---

### Q27: What is the difference between choreography and orchestration in Saga?

**A:**
```
Choreography (event-driven, decentralised):
  Each service reacts to events and emits its own events
  No central coordinator

  OrderService  ──OrderCreated──▶ InventoryService
  InventoryService ──StockReserved──▶ PaymentService
  PaymentService ──PaymentFailed──▶ InventoryService (compensate)
                                 ──▶ OrderService (compensate)

  Pros: truly decoupled; services don't know each other
  Cons: hard to trace the overall flow; cyclic event dependencies possible
        no single place to see saga state

Orchestration (centralised coordinator):
  Saga Orchestrator explicitly tells each service what to do

  Orchestrator ──"Reserve stock"──▶ InventoryService
  Orchestrator ──"Charge payment"──▶ PaymentService
  PaymentService fails → Orchestrator ──"Release stock"──▶ InventoryService

  Pros: clear flow visible in one place; easy to debug; explicit state machine
  Cons: orchestrator can become a bottleneck; coupling to orchestrator

Recommendation:
  Start with orchestration (simpler to reason about)
  Use choreography when team is experienced and loose coupling is critical
```

---

### Q28: How do you handle service-to-service authentication in microservices?

**A:**
```
Options (weakest to strongest):

1. Network-level trust (IP allowlist):
   Only allow calls from within the cluster IP range
   Pros: simple
   Cons: any compromised pod can call any other; no identity

2. API Keys / Shared secrets:
   Each service has an API key it sends in headers
   Pros: simple to implement
   Cons: key rotation is manual; hard to revoke per-service

3. JWT with service identity:
   Calling service includes signed JWT asserting its identity
   Called service verifies JWT with auth server's public key
   Pros: stateless verification; standard
   Cons: key management; JWTs expire (short-lived)

4. mTLS (recommended for production):
   Both sides present X.509 certificates (SPIFFE/SPIRE standard)
   Identity = certificate subject (e.g., spiffe://cluster/ns/payments/sa/payment-svc)
   Automated via service mesh (Istio/Linkerd) — no code changes
   Pros: cryptographic proof of identity; automatic rotation; zero-code
   Cons: needs service mesh infrastructure

Pair with AuthorizationPolicy:
  Payment Service cert is allowed to call Order Service
  Anonymous caller is denied
```

---

### Q29: What observability does a microservices system need that a monolith doesn't?

**A:**
```
A monolith: one process, one log file, one stack trace = easy to debug

Microservices additional needs:

1. Distributed Tracing (most important addition)
   End-to-end request path across services
   Inject trace_id at entry (API Gateway); propagate in all downstream calls
   Tools: Jaeger, Zipkin, Tempo (via W3C traceparent header)
   Without this: "the request was slow" but no idea which service caused it

2. Structured Logging with Correlation ID
   All logs include: trace_id, service_name, instance_id, user_id
   Centralised aggregation: ELK, Loki, Splunk
   Query: "show all logs for trace_id=abc123 across all services"

3. Per-Service Golden Signals
   Each service: RPS, error rate, latency P50/P99, saturation
   Each dependency: circuit breaker state, queue lag, DB connection pool
   RED method: Rate, Errors, Duration (per service)
   USE method: Utilisation, Saturation, Errors (per resource)

4. Service-to-Service Traffic Metrics
   Which service calls which? What's the error rate between A and B?
   Service mesh (Istio/Linkerd) provides this automatically

5. Health Check APIs
   /health/live   → am I alive? (Kubernetes liveness probe)
   /health/ready  → am I ready to serve traffic? (readiness probe)
   Include: DB connectivity, dependency health, queue lag
```

---

### Q30: How do you prevent cascading failures in a microservices system?

**A:**
```
Three patterns, applied in layers:

1. Timeout on every network call (first line of defence)
   Never wait forever; set tight timeouts per dependency type
   Internal services: 200ms | External APIs: 2s | DB: 5s
   Without timeouts: slow service starves all threads

2. Circuit Breaker (second line)
   After N failures → OPEN: fail fast without calling the service
   After timeout → HALF-OPEN: probe with one request
   If probe succeeds → CLOSED: resume normal traffic
   Prevents: thread pool exhaustion, queue backup, latency amplification

3. Bulkhead (third line)
   Separate thread pool per downstream dependency
   Slow Payment Service fills its own pool of 10 threads
   Order Service's 30-thread pool for Inventory is unaffected
   Prevents: one slow service consuming all shared threads

4. Fallback (recovery)
   Circuit OPEN → return cached result, default, or graceful degradation
   "Recommendations unavailable" beats "500 Internal Server Error"

5. Retry with backoff + jitter (for transient failures only)
   Only retry idempotent operations
   Exponential backoff + random jitter prevents thundering herd
   Max 3 retries; give up and activate fallback

Full stack:
  Request → Rate Limiter → Timeout → Retry → Circuit Breaker → Bulkhead → Fallback
```

---

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
- [[solution-arch/sources/clean-architecture]]
