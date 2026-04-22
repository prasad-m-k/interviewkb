# Saga Pattern

**Topic:** [[solution-arch/topics/data-architecture]], [[solution-arch/topics/integration-patterns]]
**Related concepts:** [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/message-queues]], [[solution-arch/patterns/outbox]]

## What it solves
In microservices, each service owns its own database. Traditional ACID transactions don't span service boundaries. The Saga pattern manages **distributed transactions** as a sequence of local transactions, each with a **compensating transaction** to undo the effect if a later step fails.

## Two Saga Styles

### Choreography (Event-Based)

Services communicate via events. Each service listens for events and publishes events when done. No central coordinator.

```
Order Service        Payment Service       Inventory Service     Shipping Service
     │                     │                      │                     │
     │── OrderCreated ─────▶│                      │                     │
     │                      │── PaymentDone ───────▶│                     │
     │                      │                      │── Reserved ──────────▶│
     │                      │                      │                     │── ShipmentBooked
     │                      │                      │                     │
     │                      Failure: PaymentFailed  │                     │
     │◀── OrderCancelled ───│                      │                     │
```

**Pros:** Loose coupling; no central SPOF; natural event audit log.
**Cons:** Hard to track overall saga state; cyclic dependencies possible; debugging is hard.

### Orchestration (Central Coordinator)

A Saga Orchestrator tells each service what to do and handles failures centrally.

```
                    ┌─────────────────────────────┐
                    │      Saga Orchestrator       │
                    │                             │
                    │  Step 1: Create Order  ─────┼──▶ Order Service
                    │  Step 2: Take Payment  ─────┼──▶ Payment Service
                    │  Step 3: Reserve Stock ─────┼──▶ Inventory Service
                    │  Step 4: Book Shipment ─────┼──▶ Shipping Service
                    │                             │
                    │  If Step 3 fails:           │
                    │    Compensate Step 2: ──────┼──▶ Payment Service (refund)
                    │    Compensate Step 1: ──────┼──▶ Order Service (cancel)
                    └─────────────────────────────┘
```

**Pros:** Clear saga state visible in one place; easier to debug; simpler compensations.
**Cons:** Orchestrator can become a God object; single point of control (not SPOF if replicated).

## Compensating Transactions

Each forward step has a compensating transaction that undoes it:

| Step | Forward | Compensating |
|------|---------|-------------|
| 1 | Create Order | Cancel Order |
| 2 | Charge Payment | Refund Payment |
| 3 | Reserve Inventory | Release Reservation |
| 4 | Book Shipment | Cancel Shipment |

**Key rule:** compensating transactions must be **idempotent** — they may be called more than once.

## Failure Scenarios

```
Happy path:    1 → 2 → 3 → 4 → Done

Step 3 fails:  1 → 2 → 3(fail) → compensate(2) → compensate(1) → Failed

Compensation fails: 
  → Retry compensation with backoff
  → If still fails → manual intervention alert (stuck saga)
  → Saga must be resumable from last successful step
```

## Saga State Machine (Orchestrator)

```
         ┌──────────────────────────────────────────┐
         │            Saga State                    │
PENDING ─▶ ORDER_CREATED ─▶ PAYMENT_TAKEN ─▶ STOCK_RESERVED ─▶ COMPLETED
              │                  │                 │
              │                  ▼                 ▼
              ▼           COMPENSATING ◀── COMPENSATING
          CANCELLED             │
                                ▼
                          CANCELLED
```

State is persisted (DB) so saga can resume after coordinator restart.

## Signal phrases
- "Distributed transaction across services"
- "Eventual consistency with rollback capability"
- "Order fulfilment pipeline"
- "Booking system (flight + hotel + car)"
- "Cannot use 2PC across services"

## Complexity
High. Requires: durable state, compensating transactions, idempotency, retry logic.

**Alternative to avoid Sagas:** design bounded contexts to minimise cross-service transactions. If Order and Payment must be atomic, consider if they belong in the same service.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
