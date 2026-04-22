# Idempotency

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/message-queues]], [[solution-arch/patterns/outbox]], [[solution-arch/concepts/acid-vs-base]]

## What it is
An operation is **idempotent** if performing it multiple times produces the same result as performing it once. Critical in distributed systems where retries, at-least-once delivery, and network failures cause duplicate requests.

## Why it matters

```
Client ──POST /payment──▶ Service ──▶ DB (write succeeds)
                          │
                     Response lost (network failure)
                          │
Client ──POST /payment──▶ Service (again, retry)
                          │
                     Charged twice! (if not idempotent)
```

Without idempotency, retries cause duplicate charges, duplicate orders, duplicate emails, etc.

## How it works

### Idempotency Keys

```
Client generates a unique key per logical operation:
  idempotency-key: "order-request-abc123"

POST /payments
  { idempotency-key: "order-request-abc123", amount: 99.00 }

Server behaviour:
  1. Check: has "order-request-abc123" been processed?
  2. Yes  → return stored response (no re-processing)
  3. No   → process payment, store result against key, return response
```

```
              ┌─────────────────────────────────┐
              │     Idempotency Store (Redis)    │
Request ─────▶│                                 │
  + Key        │  Key found? ──YES──▶ Return     │
               │                  cached result  │
               │  Key not found?                 │
               │    ──▶ Process                  │
               │    ──▶ Store result             │
               │    ──▶ Return result            │
               └─────────────────────────────────┘
```

### HTTP Method Idempotency

| Method | Idempotent? | Safe? | Notes |
|--------|-------------|-------|-------|
| GET | ✅ Yes | ✅ Yes | Read only |
| HEAD | ✅ Yes | ✅ Yes | Like GET, no body |
| PUT | ✅ Yes | ❌ No | Replace resource |
| DELETE | ✅ Yes | ❌ No | Already-deleted = same result |
| PATCH | ❌ No* | ❌ No | Depends on operation |
| POST | ❌ No | ❌ No | Creates new resource each time |

*PATCH can be made idempotent with conditional updates.

### Making POST Idempotent

```
POST /orders   { idempotency_key: "user123-cart456" }
  
Service:
  BEGIN TRANSACTION
    SELECT * FROM idempotency_keys WHERE key = 'user123-cart456' FOR UPDATE
    IF found: RETURN stored_response
    IF not found:
      CREATE order
      INSERT INTO idempotency_keys (key, response, expires_at)
      RETURN new_response
  COMMIT
```

Lock prevents race condition where two identical requests arrive simultaneously.

## Idempotent Consumer Pattern (Message Queue)

```
Message arrives: { event_id: "evt_001", type: "OrderShipped" }

Consumer:
  1. Check dedup table: has "evt_001" been processed?
  2. Yes  → skip (ack message, don't reprocess)
  3. No   → process event
          → INSERT into dedup table (key=evt_001)
          → ack message
```

Dedup table entries expire after message retention window (e.g., 7 days).

## Common interview angles
- "Why is idempotency important in distributed systems?" (Networks fail; retries happen; without idempotency, retries cause duplicates)
- "How do you implement an idempotent payment API?" (Idempotency key per request; check-then-act in a transaction)
- "Is HTTP DELETE idempotent?" (Yes — deleting an already-deleted resource still leaves it deleted)
- "What is the difference between idempotency and safety?" (Safe = no state change; idempotent = same result on repeat; GET is both; PUT is idempotent but not safe)
- "How do you handle idempotency across microservices?" (Propagate the idempotency key downstream; each service checks its own dedup store)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
