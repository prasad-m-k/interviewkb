---
uid: 4a2c8e91-6b3d-4f7a-9e1c-5d8b7f3a2c60
---

# Inbox Pattern (Transactional Inbox / Idempotent Consumer)

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/idempotency]], [[solution-arch/patterns/outbox]], [[solution-arch/patterns/saga]]

## What it solves
Message brokers give **at-least-once delivery**: a message can be redelivered after a consumer crash, a broker rebalance, or a network timeout that hides a successful ack. If the consumer just reprocesses the message — charge the card again, ship the order twice — a duplicate delivery becomes a duplicate side effect. The Inbox pattern makes the consumer side of messaging safe against that redelivery, the same way the outbox pattern makes the producer side safe against dual-write.

```
WITHOUT Inbox (unsafe):
  Message "evt_001" arrives → process (charge card) → ack
  Broker redelivers "evt_001" (ack was lost, or consumer crashed before ack)
  Message "evt_001" arrives again → process again → charged twice!
```

## Template / Skeleton

```
WITH Inbox (safe):
  BEGIN TRANSACTION
    SELECT 1 FROM inbox WHERE event_id = 'evt_001' FOR UPDATE
    IF found: ROLLBACK, ack message, skip — already processed
    IF not found:
      INSERT event_id INTO inbox_table
      Apply business effect (charge card, update order, ...)
  COMMIT                              ← dedup record + business effect atomic
  ack message
```

```
┌──────────────────────────────────────┐
│ Consumer                             │
│                                      │
│ Kafka/RabbitMQ ──▶ Message "evt_001" │
│         │                            │
│         ▼                            │
│     ┌───────────────────────────┐    │
│     │ inbox table (event_id PK) │    │ ◀── dedup check + business write, same transaction
│     │ orders table              │    │
│     └───────────────────────────┘    │
└──────────────────────────────────────┘
```

The dedup check and the business-effect write happen in the **same local transaction**, so either both land or neither does — there is no window where the event is marked processed but the effect never ran, or vice versa.

## Inbox Table Schema

```sql
CREATE TABLE inbox (
  event_id     VARCHAR(100) PRIMARY KEY,
  event_type   VARCHAR(100) NOT NULL,
  received_at  TIMESTAMPTZ  DEFAULT NOW(),
  processed_at TIMESTAMPTZ
);

-- event_id as PK does double duty: uniqueness constraint IS the dedup check
```

The PK insert doubles as the atomicity guard — a second insert of the same `event_id` fails the constraint, so the transaction rolls back and nothing downstream re-runs.

## Outbox + Inbox Together

Outbox solves reliable **publishing** (DB write + event never diverge). Inbox solves reliable **consumption** (redelivery never causes duplicate effects). Chained end to end, they give effectively-exactly-once processing across a fully at-least-once transport:

```
Service A                                      Service B
┌──────────────────────────┐                   ┌──────────────────────────────┐
│ DB write + outbox insert │   ── Kafka ──▶    │ Inbox dedup check + DB write │
│ (1 transaction)          │  (at-least-once)  │ (1 transaction)              │
└──────────────────────────┘                   └──────────────────────────────┘

reliable publish                               idempotent consume
```

Neither side alone is enough: outbox without inbox still lets broker-level redelivery hit the consumer twice; inbox without outbox still risks the producer's DB write and publish diverging.

## Signal phrases
- "Idempotent consumer"
- "Exactly-once processing over at-least-once delivery"
- "Prevent duplicate message processing"
- "Message redelivery / broker rebalance safety"
- "Transactional inbox"

## Complexity
Low–moderate. Requires: inbox table with a unique/PK constraint on the dedup key, dedup check + business write in one local transaction, a retention/cleanup policy so the table doesn't grow unbounded (expire rows past the broker's redelivery window, e.g. 7 days).

## Trade-offs

| | Inbox (DB dedup) | In-memory/cache dedup (e.g. Redis SETNX) |
|--|-------------------|-------------------------------------------|
| Durability | Survives consumer restart | Lost on cache eviction/restart unless persisted |
| Atomicity with business write | Yes — same transaction | No — separate systems, own race window |
| Extra infra | None (same DB) | Requires a cache with TTL management |
| Best for | Effects that also write to the same DB | High-throughput checks where a rare duplicate is tolerable |

## Common interview angles
- "How do you handle duplicate messages from Kafka/SQS?" (dedup key + unique constraint, checked in the same transaction as the effect)
- "Isn't 'exactly-once' impossible in distributed systems?" (True end-to-end; inbox achieves *effectively* exactly-once by making at-least-once delivery + idempotent processing compose to the same outcome)
- "Where does the dedup key come from?" (Producer-assigned event ID, ideally set in the same transaction as the outbox insert, so it's stable across retries)
- "What happens if the inbox table grows forever?" (TTL/cleanup job pruning rows older than the broker's max redelivery window)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
