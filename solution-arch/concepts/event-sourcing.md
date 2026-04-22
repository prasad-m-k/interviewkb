# Event Sourcing

**Topic:** [[solution-arch/topics/data-architecture]]
**Related:** [[solution-arch/concepts/cqrs]], [[solution-arch/patterns/cqrs-event-sourcing]], [[solution-arch/concepts/message-queues]]

## What it is
Instead of storing the current state of an entity, event sourcing stores the **full sequence of events** that led to the current state. The current state is derived by replaying events from the beginning (or from a snapshot).

## How it works

```
Traditional (State-based):
  Order record: { id: 1, status: "shipped", amount: 99.00 }
  (history lost — we only know current state)

Event Sourced:
  Event log:
  [1] OrderCreated    { id: 1, amount: 99.00,  at: T1 }
  [2] PaymentReceived { id: 1, txn: "xyz",      at: T2 }
  [3] OrderShipped    { id: 1, tracking: "UPS", at: T3 }
  
  Current state = replay all events ──▶ { status: "shipped", ... }
```

## Event Store Architecture

```
Command ──▶ Domain Logic ──▶ Emit Event ──▶ Event Store (append-only log)
                                                  │
                                          ┌───────▼───────┐
                                          │  Subscribers  │
                                          │  (projections)│
                                          └───────┬───────┘
                                                  │
                                      ┌───────────▼──────────┐
                                      │    Read Models       │
                                      │  (DB, cache, index)  │
                                      └──────────────────────┘
```

## Snapshots

Replaying 10,000 events every time is expensive. Snapshots capture state periodically:

```
Snapshot at event #5000:
  Store: { state_at_5000 }

On load:
  1. Load snapshot (event 5000)
  2. Replay events 5001 → now
  (much faster than replaying from event 1)
```

## Benefits

- **Full audit trail:** every state change recorded with who/when/why
- **Temporal queries:** "What was the order status on Tuesday?" → replay to that point
- **Event replay:** rebuild read models, fix bugs by reprocessing events
- **Debugging:** exact sequence of events that led to any state
- **Natural fit with CQRS:** event log is the write side; projections build read models

## Costs

- **Query complexity:** can't `SELECT * WHERE status = 'shipped'` directly — need projections
- **Storage growth:** append-only log grows indefinitely (mitigated by archival/snapshots)
- **Schema evolution:** changing event schemas is hard — need versioned events, upcasters
- **Eventual consistency:** projections update asynchronously — stale reads possible

## When to use

✅ Audit trail is a hard requirement (finance, compliance, medical)
✅ Business processes are naturally event-driven (orders, state machines)
✅ Need temporal queries or time-travel debugging
✅ Complex workflows with compensation logic

❌ Simple CRUD with no audit requirements — massive over-engineering
❌ Team unfamiliar with eventual consistency

## Common interview angles
- "What is event sourcing and how is it different from a change log / audit table?"
- "How do you handle schema changes in event sourcing?" (Event versioning: v1_OrderCreated → v2_OrderCreated; upcaster transforms old format at read time)
- "What is the difference between event sourcing and CQRS?" (Orthogonal concepts: can use either without the other; often combined)
- "How do you query state in an event-sourced system?" (Via projections / read models built from the event stream)
- "How do you avoid unbounded event log growth?" (Snapshots, archival, compaction)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
