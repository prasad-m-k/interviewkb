# Microservices Architecture

**Related:** [[solution-arch/topics/architectural-styles]], [[solution-arch/topics/integration-patterns]], [[solution-arch/topics/scalability-and-reliability]]
**Patterns:** [[solution-arch/patterns/microservices-decomposition]], [[solution-arch/patterns/api-composition]], [[solution-arch/patterns/database-per-service]], [[solution-arch/patterns/saga]], [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/sidecar-ambassador]], [[solution-arch/patterns/bff]]
**Scenarios:** [[solution-arch/scenarios/monolith-to-microservices]], [[solution-arch/scenarios/interview-questions]]

---

## What Microservices Are

A microservices architecture structures an application as a collection of **small, independently deployable services**, each:
- Owning a single business capability
- Running in its own process
- Owning its own data store (no shared DB)
- Communicating over a network (HTTP/gRPC or messaging)
- Deployable and scalable independently

> "Microservices" is an architectural style, not a technology. The defining characteristics are **autonomy** and **bounded context ownership** — not service size.

---

## The Eight Fallacies of Distributed Computing

These are assumptions that feel safe in a monolith but are false in microservices. Architects must design around all eight:

```
1. The network is reliable          → add retries, timeouts, circuit breakers
2. Latency is zero                  → design for latency budget per call
3. Bandwidth is infinite            → minimise data transferred; pagination
4. The network is secure            → mTLS, zero trust, encryption everywhere
5. Topology doesn't change          → use service discovery, not hardcoded IPs
6. There is one administrator       → distributed ops; runbooks per service
7. Transport cost is zero           → serialisation (Protobuf > JSON for perf)
8. The network is homogeneous       → heterogeneous infra; polyglot is normal
```

---

## Core Principles

### 1. Single Responsibility / Bounded Context
Each service owns one business domain (e.g., Orders, Payments, Users). Boundaries are derived from **Domain-Driven Design (DDD)** bounded contexts — not technical layers.

```
Good decomposition (by capability):
  User Service     → identity, profile, authentication
  Order Service    → order lifecycle, order history
  Payment Service  → charge, refund, payment methods
  Inventory Service→ stock levels, reservations

Bad decomposition (by technical tier):
  Data Service     → all CRUD operations (creates a shared DB again)
  Logic Service    → all business rules
  Presentation Svc → all UI concerns
```

### 2. Database per Service (Data Isolation)
Each service owns its data. No direct cross-service DB queries.

```
✅ Order Service   → orders_db (PostgreSQL)
✅ User Service    → users_db  (PostgreSQL)
✅ Search Service  → search_idx (Elasticsearch)

❌ Order Service joins users_db  → forbidden: couples services at DB level
```

See: [[solution-arch/patterns/database-per-service]]

### 3. Communicate via APIs or Events
- **Sync:** REST/gRPC for user-facing reads, query responses
- **Async:** Events (Kafka) for state changes, workflows, fan-out

### 4. Failure Isolation
One service failure must not cascade to others. Requires: circuit breakers, bulkheads, timeouts, graceful degradation.

### 5. Independent Deployability
A service can be built, tested, and deployed without coordinating with other services.

---

## Communication Patterns

```
Synchronous (request-response):
  Client ──HTTP/gRPC──▶ Service A ──HTTP/gRPC──▶ Service B
  Use for: user-facing real-time reads, simple queries
  Risk: cascading failures, latency amplification

Asynchronous (event-driven):
  Service A ──event──▶ Broker ──▶ Service B
                             └──▶ Service C
  Use for: state changes, workflows, background work
  Benefit: temporal decoupling, fan-out, resilience

Request/Response via Queue:
  Client ──request──▶ Queue ──▶ Service ──response──▶ Response Queue ──▶ Client
  Use for: long-running jobs where client polls for result
```

**API Gateway** handles all external traffic (see [[solution-arch/concepts/api-gateway]]).
**Service Mesh** handles all internal service-to-service traffic (see [[solution-arch/concepts/service-mesh]]).

---

## Data Management Strategies

```
Challenge: Cross-service queries (e.g., "get order with customer name")

Option 1: API Composition
  Order Service fetches order + calls User Service for name
  Pros: simple; Cons: synchronous coupling, N+1 latency risk

Option 2: CQRS Read Model
  Materialised view: pre-join order + user data into read DB
  Pros: fast reads, no coupling at query time
  Cons: eventual consistency, projection logic

Option 3: Event-driven data replication
  User Service publishes UserUpdated events
  Order Service maintains a local copy of user names
  Pros: fully decoupled reads; Cons: data duplication, staleness
```

---

## Observability in Microservices

Debugging a distributed system requires all three pillars:

```
Logs     → structured JSON; include: trace_id, service_name, user_id
           → aggregated in centralised log store (ELK, Loki)

Metrics  → per service: RPS, error rate, latency P50/P99/P999
           → per dependency: circuit breaker state, queue lag
           → scraped by Prometheus; dashboards in Grafana

Traces   → end-to-end request path across services
           → inject/propagate trace context (trace_id, span_id) via headers
           → W3C Trace Context standard: traceparent header
           → visualised in Jaeger, Zipkin, Tempo
```

**Correlation ID / Trace ID:** Every request gets a unique ID at the entry point (API Gateway). Propagated in all downstream calls. Essential for debugging "which service caused this failure."

---

## Service Resilience Stack

```
Incoming Request
       │
       ▼
Rate Limiter      ← shed excess load before it enters the system
       │
       ▼
Timeout           ← never wait forever; set per dependency
       │
       ▼
Retry (with backoff + jitter)   ← retry transient failures
       │
       ▼
Circuit Breaker   ← fail fast when dependency is unhealthy
       │
       ▼
Bulkhead          ← isolate thread pool per dependency
       │
       ▼
Fallback          ← cached result, default, or graceful error
```

---

## Team Organisation (Conway's Law)

```
Microservices require team autonomy. Wrong team structure = distributed monolith.

Stream-aligned team (owns service end-to-end):
  Team A → User Service (code, deploy, oncall, SLA, DB)
  Team B → Order Service
  Team C → Payment Service

Platform team (provides enabling infrastructure):
  CI/CD pipelines, service mesh, observability stack, shared libraries

Enabling team (coaches stream teams on new patterns):
  DDD workshops, security reviews, performance profiling

Rule: One service, one owning team.
      Shared ownership = no clear ownership = maintenance debt.
```

---

## When Microservices Are Overkill

```
Avoid microservices when:
  ❌ Team < 10 engineers (operational overhead not justified)
  ❌ Domain is not yet understood (wrong boundaries are worse than a monolith)
  ❌ Strong consistency required across all data (distributed txns are very hard)
  ❌ No CI/CD pipeline in place (manual deploys of 20 services = chaos)
  ❌ No observability infrastructure (blind distributed system = nightmare)

Start with a modular monolith → extract services as pain becomes concrete.
"Microservices are the goal of an organisation that has solved its team scaling problem."
```

---

## Microservices Pitfall Catalogue

| Pitfall | Description | Fix |
|---------|-------------|-----|
| Distributed monolith | Services tightly coupled via sync chains | Use async events; define clear contracts |
| Nanoservices | Too many tiny services; high coordination overhead | Merge overly granular services |
| Shared DB | Multiple services read/write same DB | Database per service |
| Synchronous chains | A→B→C→D all sync → latency amplification + cascading failure | Async for non-critical paths; circuit breakers |
| Missing observability | Can't debug failures across services | Distributed tracing from day 1 |
| No service ownership | "Everyone owns" = no one owns | Strict: one team per service |
| Premature decomposition | Services before domain is understood | Domain events storming first; extract later |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
- [[solution-arch/sources/clean-architecture]]
