# Message Queues & Event Streaming

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/concepts/idempotency]], [[solution-arch/patterns/outbox]]

## What it is
A message queue decouples a producer (sender) from a consumer (receiver) via a durable intermediate buffer. Producers and consumers scale independently and don't need to be running simultaneously.

## Queue vs Pub/Sub vs Stream

```
Queue (Point-to-Point):
  Producer ──▶ [msg][msg][msg] ──▶ Consumer
               Each message consumed once; deleted on ack.
               Multiple consumers compete for messages (work queue).

Pub/Sub (Broadcast):
  Publisher ──▶ Topic ──▶ Subscriber A  (independent copy)
                     └──▶ Subscriber B  (independent copy)
                     └──▶ Subscriber C  (independent copy)
               Each subscriber gets every message.

Event Stream (Kafka model):
  Producer ──▶ Topic/Partition ──▶ Consumer Group
               Log is retained (days/forever).
               Consumers track their own offset.
               Replay possible; multiple independent groups.
```

## Delivery Guarantees

| Guarantee | What it means | Risk |
|-----------|--------------|------|
| **At-most-once** | Message sent once; may be lost | Data loss on crash |
| **At-least-once** | Message retried until acked; may be duplicated | Duplicate processing |
| **Exactly-once** | Delivered and processed exactly once | Complex; high cost |

**Exactly-once in practice:** Use idempotent consumers (deduplicate by message ID) + transactional outbox. True exactly-once end-to-end is very hard.

## Message Ordering

```
No Ordering Guarantee:
  Producer sends: 1, 2, 3
  Consumer receives: 2, 1, 3   (network reordering)

FIFO Queue (SQS FIFO, Kafka within a partition):
  Producer sends: 1, 2, 3
  Consumer receives: 1, 2, 3   (ordered)
  Cost: lower throughput; no parallel processing within a group
```

**Kafka:** ordering guaranteed within a partition. Use a consistent partition key (e.g., user_id) to ensure all events for one user are ordered.

## Dead Letter Queue (DLQ)

```
                              Processing fails 3 times
Consumer ──process──▶ Error ──────────────────────────▶ DLQ
                                                          │
                                                    Alert + inspect
                                                    Replay when fixed
```

DLQ captures messages that permanently fail processing. Essential for debugging and recovery without losing data.

## Kafka Architecture

```
                         Kafka Cluster
┌──────────┐        ┌───────────────────────────────────┐
│Producer A│──────▶ │  Topic: orders                    │
│Producer B│──────▶ │  ┌─────────────┬──────────────┐   │
└──────────┘        │  │ Partition 0  │ Partition 1  │   │
                    │  │ [1][4][7]   │ [2][5][8]    │   │
                    │  └──────┬──────┴──────┬────────┘   │
                    └─────────┼─────────────┼────────────┘
                              │             │
                    ┌─────────▼──┐    ┌─────▼──────────┐
                    │Consumer    │    │Consumer        │
                    │Group A     │    │Group B         │
                    │Instance 0  │    │Instance 0      │
                    └────────────┘    └────────────────┘
```

**Kafka key concepts:**
- **Topic:** named stream of records
- **Partition:** unit of parallelism; ordered log within partition
- **Offset:** consumer's position in a partition (consumer owns this, not broker)
- **Consumer Group:** logical subscriber; partitions split across instances
- **Retention:** messages kept for days/weeks regardless of consumption

## Common interview angles
- "When would you use a queue vs a database?" (queue: async, durable, decoupled; DB: queries, joins, transactions)
- "How do you prevent duplicate processing?" (idempotency key, deduplication table, transactional outbox)
- "What is consumer lag and why does it matter?" (difference between latest offset and consumer's offset — indicates falling behind)
- "How does Kafka differ from RabbitMQ?" (Kafka: log/stream, replay, high throughput; RabbitMQ: traditional queue, routing, complex topologies)
- "What happens when a consumer crashes mid-processing?" (at-least-once: message redelivered; use idempotency to handle)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
