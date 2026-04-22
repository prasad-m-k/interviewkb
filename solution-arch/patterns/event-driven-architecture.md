# Event-Driven Architecture (EDA)

**Topic:** [[solution-arch/topics/integration-patterns]], [[solution-arch/topics/architectural-styles]]
**Related concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/event-sourcing]], [[solution-arch/patterns/saga]]

## What it solves
Tight coupling between services forces synchronous, brittle integrations. EDA lets services communicate via events through a broker — producers emit events; consumers react independently. Producer and consumer don't know about each other.

## Core Concepts

```
Event:     Something that happened — immutable fact
           { type: "OrderPlaced", orderId: "123", userId: "u1", at: "..." }

Command:   Intent to do something — may be rejected
           { type: "PlaceOrder", items: [...] }

Query:     Request for current state — no side effects
           { type: "GetOrderStatus", orderId: "123" }
```

In EDA: Commands produce Events. Events trigger more Commands in other services.

## Architecture

```
           ┌──────────────────────────────────────────────┐
           │              Event Broker (Kafka)             │
           │                                              │
           │  Topic: orders       Topic: inventory        │
           │  [OrderPlaced]       [StockReserved]         │
           │  [OrderCancelled]    [StockReleased]         │
           └────────────────────────────────────┬─────────┘
                    ▲               ▲            │
                    │               │            ▼
          ┌─────────┴──┐   ┌────────┴────┐   ┌──────────────┐
          │   Order    │   │  Inventory  │   │ Notification │
          │  Service   │   │  Service    │   │ Service      │
          │            │──▶│             │   │              │
          │ Emits:     │   │ Listens:    │   │ Listens:     │
          │ OrderPlaced│   │ OrderPlaced │   │ OrderPlaced  │
          │            │   │ Emits:      │   │ OrderShipped │
          └────────────┘   │ StockResvd  │   └──────────────┘
                           └─────────────┘
```

## Event Patterns

### Event Notification
Thin event — "something happened", consumer fetches details if needed.
```json
{ "type": "UserRegistered", "userId": "u123" }
// Consumer calls User Service to get full user data
```
Pros: Small events. Cons: extra round trip; consumer needs to call back.

### Event-Carried State Transfer
Fat event — carries all state needed by consumers.
```json
{ "type": "UserRegistered", "userId": "u123", "email": "...", "name": "...", "plan": "..." }
// Consumer has all it needs — no callback required
```
Pros: Decoupled; no round trips. Cons: Large events; schema coupling.

### Event Sourcing
Events are the persistent record (see [[solution-arch/concepts/event-sourcing]]).

## Choreography vs Orchestration

```
Choreography (EDA pure):
  Service A emits → Service B reacts → Service C reacts
  No central coordinator; services decide what to do based on events

Orchestration (Workflow engine):
  Coordinator: "Service A, do this. Now Service B, do that."
  Central control; easier to understand flow
```

## EDA Trade-offs

| Benefit | Cost |
|---------|------|
| Loose coupling (services independent) | Eventual consistency |
| Natural audit log (event stream) | Hard to trace request flow |
| Elastic fan-out (N consumers, no code change) | Debugging complexity |
| Producer unaffected by consumer failures | No immediate error feedback |
| Replay capability (re-consume old events) | Consumer idempotency required |
| Independent scaling of producers/consumers | Ordering challenges |

## Signal phrases
- "Need to notify multiple services when X happens"
- "Services should be independent and deployable separately"
- "Need audit trail of all changes"
- "Workflow spanning multiple services without tight coupling"

## When to use EDA

✅ Fan-out: one event triggers many independent reactions
✅ Audit/compliance: need permanent record of what happened
✅ Async processing: email, SMS, report generation
✅ Decoupled teams: each service owned by different team

❌ Strong consistency required (distributed tx) → Saga with careful design
❌ Immediate response needed (user-facing sync API) → REST/gRPC + async side effects

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
