# CQRS + Event Sourcing (Combined)

**Topic:** [[solution-arch/topics/data-architecture]]
**Related concepts:** [[solution-arch/concepts/cqrs]], [[solution-arch/concepts/event-sourcing]], [[solution-arch/concepts/message-queues]]

## What it solves
Maximum flexibility in a system that requires strong audit trails, multiple query views, and temporal queries. CQRS and Event Sourcing complement each other naturally: the event log is the write model; projections from the event log populate one or more read models.

## Full Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Write Side                                   │
│                                                                     │
│  Client ──Command──▶ Command Handler ──validates──▶ Domain Object  │
│                                │                         │          │
│                         Emit Event                       │          │
│                                │                         │          │
│                                ▼                         │          │
│                    ┌────────────────────┐                │          │
│                    │    Event Store     │                │          │
│                    │  (append-only log) │◀───────────────┘          │
│                    │  [OrderCreated]    │                           │
│                    │  [PaymentDone]     │                           │
│                    │  [Shipped]         │                           │
│                    └────────┬───────────┘                           │
└─────────────────────────────┼───────────────────────────────────────┘
                              │ events stream
                  ┌───────────┼────────────────────────┐
                  ▼           ▼                         ▼
         ┌──────────────┐ ┌──────────────┐    ┌──────────────────┐
         │ Projection 1 │ │ Projection 2 │    │  Projection 3    │
         │ (Order view) │ │(Dashboard)   │    │ (Search index)   │
         └──────┬───────┘ └──────┬───────┘    └───────┬──────────┘
                │                │                     │
         ┌──────▼──┐      ┌──────▼──┐          ┌──────▼──────┐
         │ Read DB │      │  Redis  │          │Elasticsearch│
         │(Postgres│      │(counters│          │(search)     │
         │ orders) │      │/summary)│          │             │
         └─────────┘      └─────────┘          └─────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                        Read Side                                    │
│                                                                     │
│  Client ──Query──▶ Query Handler ──▶ Read Store (any of the above) │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Characteristics

```
Write Side:
  ✅ ACID via event store (append-only, no concurrent update conflicts)
  ✅ Domain model enforces invariants on each command
  ✅ Event store = ground truth; never mutated
  ✅ Full history; supports temporal queries ("state as of T")

Read Side:
  ✅ Each read model optimised for its query pattern
  ✅ Can have many independent read models (no compromise schema)
  ✅ Read models can be rebuilt by replaying event log
  ⚠️  Eventual consistency (projection lags behind event store)
```

## Temporal Query (Time Travel)

```python
# What was order 123's state on 2026-01-15?
events = event_store.get_events(aggregate_id="order-123",
                                 until=datetime(2026, 1, 15))
order = Order.replay(events)  # replay only events up to that date
return order.status
```

This is only possible with event sourcing — state-based systems lose historical states.

## Rebuilding a Read Model

```
If read model is corrupted, stale, or schema changed:
  1. Drop the read model (e.g., truncate orders_view table)
  2. Replay all events from event store from the beginning
  3. Projection rebuilds the read model

  Read side is fully derived — it can always be rebuilt.
  Write side (event store) is the irreplaceable source of truth.
```

## Signal phrases
- "Need multiple views of the same data"
- "Audit log is a hard requirement"
- "Time-travel queries"
- "Write and read have very different scaling needs"
- "Complex domain with business rules enforced on writes"

## Complexity
Very high. Only justified when the benefits (audit, temporal, multiple views) are genuinely required. Most systems don't need this — use a simple RDBMS with read replicas instead.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
