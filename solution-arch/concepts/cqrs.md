# CQRS — Command Query Responsibility Segregation

**Topic:** [[solution-arch/topics/data-architecture]]
**Related:** [[solution-arch/concepts/event-sourcing]], [[solution-arch/patterns/cqrs-event-sourcing]], [[solution-arch/concepts/message-queues]]

## What it is
CQRS separates the **write path** (Commands — change state) from the **read path** (Queries — read state) into distinct models, often backed by different stores optimised for each purpose.

## How it works

```
                    ┌──────────────────────────────────────────┐
                    │              Application                 │
                    └──────────┬─────────────────┬────────────┘
                               │                 │
                         Commands            Queries
                    (PlaceOrder, etc)    (GetOrderStatus)
                               │                 │
                    ┌──────────▼──────┐  ┌───────▼──────────┐
                    │  Command Model  │  │   Query Model    │
                    │  (Write Side)   │  │   (Read Side)    │
                    └──────────┬──────┘  └───────▲──────────┘
                               │                 │
                    ┌──────────▼──────┐  ┌───────┴──────────┐
                    │  Write Store    │  │   Read Store     │
                    │  (PostgreSQL)   │  │  (Elasticsearch  │
                    └──────────┬──────┘  │   / Redis / DDB) │
                               │         └──────────────────┘
                               │ events / sync
                               └────────────────▶ (async projection)
```

## Why separate reads and writes?

**The problem without CQRS:**
- One domain model must satisfy both write (complex transactions) and read (complex queries with joins/aggregations)
- Adding a read-optimised index breaks the domain model
- Scaling reads and writes independently is hard when they share the same model and DB

**With CQRS:**
- Write model: normalised, enforces invariants, handles transactions
- Read model: denormalised, pre-aggregated, optimised for query patterns (can be Elasticsearch, a materialised view, Redis, etc.)
- Read and write sides scale independently

## Complexity

```
Without CQRS (simple):
  API ──▶ Service ──▶ DB  (single model for reads + writes)

With CQRS:
  API ──▶ Command Handler ──▶ Write DB ──events──▶ Projection ──▶ Read DB
      └──▶ Query Handler  ──▶ Read DB
  
  Extra: event bus, projection logic, eventual consistency between models
```

## When to use

✅ Read-heavy workloads with complex query patterns
✅ Different scaling requirements for reads vs writes
✅ Audit log / event history is valuable
✅ Multiple independent read models needed (mobile vs web vs reporting)

❌ Simple CRUD apps — unnecessary complexity
❌ Team unfamiliar with eventual consistency — operational burden

## Eventual consistency in CQRS

```
Client writes ──▶ Command accepted (write DB updated)
                         │
                         │ async projection (may take 100ms–seconds)
                         ▼
                   Read DB updated

Client reads immediately after write ──▶ may see stale data
```

**Mitigation:** Return the write result directly in the command response. Or use read-your-writes consistency (route reads to write DB for the writing user's session for a brief window).

## Common interview angles
- "What problem does CQRS solve?"
- "Does CQRS require event sourcing?" (No — you can sync write to read model via DB triggers, CDC, or domain events without full event sourcing)
- "How do you handle the read model being stale?" (Accept eventual consistency; design UI to show 'processing' state)
- "How do you keep read models consistent after a crash?" (Replay events from write store / event log)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
