# Outbox Pattern (Transactional Outbox)

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/idempotency]], [[solution-arch/patterns/saga]]

## What it solves
The dual-write problem: after updating a database, you want to publish an event to a message broker. But the DB write and the message publish are **two separate operations** — if the app crashes between them, one succeeds and the other doesn't.

```
WITHOUT Outbox (unreliable):
  BEGIN TRANSACTION
    INSERT order into DB       ← succeeds
  COMMIT
  Publish OrderCreated to Kafka ← crashes here → event never published
  
  Result: order exists in DB, but downstream services never notified.
```

## Template / Skeleton

```
WITH Outbox (reliable):
  BEGIN TRANSACTION
    INSERT order INTO orders_table
    INSERT event INTO outbox_table   ← same transaction!
  COMMIT                             ← both succeed or both fail

  Outbox Relay (separate process):
    Poll outbox_table for unpublished events
    Publish event to Kafka
    Mark event as published in outbox_table
```

```
┌─────────────────────────────────────────────────────────┐
│                         Database                        │
│  ┌─────────────────┐          ┌───────────────────────┐ │
│  │  orders table   │          │     outbox table      │ │
│  │  id | status    │          │  id | event | status  │ │
│  │  1  | created   │          │  1  | {...} | pending │ │
│  └─────────────────┘          └──────────┬────────────┘ │
└─────────────────────────────────────────┼───────────────┘
                                          │ poll
                                ┌─────────▼────────┐
                                │  Outbox Relay    │──▶ Kafka/RabbitMQ
                                │  (CDC or poller) │
                                └──────────────────┘
```

## Two Implementation Approaches

### Polling Relay
```
Every 100ms:
  SELECT * FROM outbox WHERE status = 'pending' LIMIT 100
  For each event: publish to message broker
  UPDATE outbox SET status = 'published' WHERE id = event.id
```
Simple. Small latency overhead (poll interval). Works with any DB.

### Change Data Capture (CDC)
```
DB transaction log (WAL in PostgreSQL / binlog in MySQL)
    │ stream
    ▼
CDC Tool (Debezium, AWS DMS, Maxwell)
    │
    ▼
Message Broker (Kafka, RabbitMQ)
```
Near-zero latency. No polling load on DB. But requires CDC infrastructure.

## Outbox Table Schema

```sql
CREATE TABLE outbox (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type   VARCHAR(100) NOT NULL,
  aggregate_id VARCHAR(100) NOT NULL,
  payload      JSONB NOT NULL,
  status       VARCHAR(20)  DEFAULT 'pending',
  created_at   TIMESTAMPTZ  DEFAULT NOW(),
  published_at TIMESTAMPTZ
);

-- Index for efficient polling
CREATE INDEX idx_outbox_pending ON outbox(status, created_at)
  WHERE status = 'pending';
```

## Signal phrases
- "Reliable event publishing"
- "At-least-once event delivery without 2PC"
- "Guaranteed DB + message broker consistency"
- "Transactional outbox"

## Complexity
Moderate. Requires: outbox table, relay process, deduplication on consumers (relay may publish twice if it crashes after publish but before marking published).

## Trade-offs

| | Polling | CDC |
|--|---------|-----|
| Latency | Poll interval (~100ms) | Near-zero |
| Infra | Just a scheduler | Debezium + Kafka Connect |
| DB load | Extra poll queries | Reads WAL (low overhead) |
| Reliability | Depends on scheduler uptime | Depends on CDC pipeline |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
