# Database per Service

**Topic:** [[solution-arch/topics/microservices]], [[solution-arch/topics/data-architecture]]
**Related concepts:** [[solution-arch/concepts/cqrs]], [[solution-arch/concepts/event-sourcing]], [[solution-arch/patterns/saga]], [[solution-arch/patterns/api-composition]]

## What it solves
In microservices, services must be independently deployable and loosely coupled. A shared database creates tight coupling: schema changes in one service can break another, services compete for the same connection pool, and you can't choose the best database type per service.

## The Pattern

Each service owns its own database, which no other service is allowed to access directly.

```
✅ Database per Service:
┌─────────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  Order Service  │    │  User Service   │    │  Search Service  │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  ┌────────────┐  │
│  │ Orders DB │  │    │  │  Users DB │  │    │  │ Search Idx │  │
│  │(PostgreSQL│  │    │  │(PostgreSQL│  │    │  │(Elasticsearch  │
│  └───────────┘  │    │  └───────────┘  │    │  └────────────┘  │
└─────────────────┘    └─────────────────┘    └──────────────────┘
  ↑ owns and controls     ↑ owns and controls    ↑ owns and controls

❌ Shared Database (anti-pattern):
┌─────────────────┐    ┌─────────────────┐
│  Order Service  │    │  User Service   │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    ▼
           ┌──────────────┐
           │  Shared DB   │  ← schema coupling, connection pool contention,
           └──────────────┘    can't change schema independently
```

## Database Technology per Service

Because each service owns its DB, you can pick the best tool per use case:

| Service | DB choice | Reason |
|---------|----------|--------|
| User Service | PostgreSQL | Relational, ACID, mature |
| Session/Cache | Redis | Sub-ms key-value lookup |
| Product Catalogue | MongoDB | Flexible schema (varying attributes per category) |
| Search Service | Elasticsearch | Full-text, faceted search |
| Analytics | ClickHouse/BigQuery | Columnar, fast aggregations |
| Social Graph | Neo4j | Relationship traversal |
| Time-Series Metrics | InfluxDB / TimescaleDB | Time-series queries |

---

## The Cross-Service Query Problem

The hardest consequence: you can no longer `JOIN orders JOIN users`.

### Strategy 1: API Composition (synchronous)
```
Client requests: "Get order with customer name"

Order Service:
  1. Fetch order from Orders DB
  2. Call User Service API: GET /users/{userId}
  3. Merge: { ...order, customerName: user.name }
  4. Return to client

Pros:  simple; always fresh data
Cons:  synchronous coupling; latency = order fetch + user fetch
       if User Service is down → order display fails
```

See: [[solution-arch/patterns/api-composition]]

### Strategy 2: CQRS Read Model (async materialised view)
```
User Service publishes: UserNameChanged { userId, newName }
Order Service consumes event → updates local orders_with_users read table

Client: reads from orders_with_users (pre-joined, fast)

Pros:  no runtime coupling; fast reads; resilient
Cons:  eventual consistency (stale until event processed);
       data duplication; projection logic to maintain
```

### Strategy 3: Shared Read Store (for reporting only)
```
Each service pushes data to a shared data warehouse (via Kafka → data pipeline)
Reporting queries hit the warehouse, not production DBs

Pros:  ad-hoc queries; no impact on production DBs
Cons:  data warehouse is NOT a back door to cross-service writes
       read-only; latency (minutes, not real-time)
```

---

## Distributed Transactions

Without a shared DB, ACID transactions across services are impossible. Use:

- **Saga Pattern** for write operations that span multiple services (see [[solution-arch/patterns/saga]])
- **Outbox Pattern** for reliable event publishing within one service (see [[solution-arch/patterns/outbox]])
- **Eventual consistency** by design: accept that cross-service data will be briefly inconsistent

---

## Private Table / Schema (stepping stone)
For teams migrating from a shared DB, start with logical isolation before physical:

```
Phase 1 (Logical):
  orders.*    → owned by Order Service only (by convention + enforcement)
  users.*     → owned by User Service only
  Same DB instance, schema-level separation

Phase 2 (Physical):
  Separate DB instances per service
  Sync via CDC or events during transition
  Cut over: service reads from own DB
```

---

## Common interview angles
- "Why does microservices architecture require a database per service?" (coupling, independent deploy, polyglot persistence)
- "How do you query data that spans two services without a shared DB?" (API composition, CQRS read model, event-driven replication)
- "How do you handle transactions that span multiple services?" (Saga pattern)
- "What is a distributed monolith?" (services deployed separately but sharing a DB — worst of both worlds)
- "How do you migrate from a shared DB to per-service DBs?" (strangler fig data strategy: logical isolation → sync → cut over)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
