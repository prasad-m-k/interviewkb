# Designing Data-Intensive Applications

**Type:** book
**Author:** Martin Kleppmann
**Ingested:** 2026-04-22
**Topics covered:** [[solution-arch/topics/data-architecture]], [[solution-arch/topics/scalability-and-reliability]], [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/event-sourcing]], [[solution-arch/concepts/cqrs]]

## Summary

The definitive reference for backend and distributed systems architecture. Covers storage engines, replication, partitioning, distributed transactions, stream processing, and batch processing from first principles. Exceptionally balanced — presents trade-offs rather than prescriptions.

Part 1 (Foundations): Data models, storage engines, encoding. Explains B-trees vs LSM-trees, column-oriented storage, schema evolution.

Part 2 (Distributed Data): Replication (single-leader, multi-leader, leaderless), partitioning (range vs hash), transactions (ACID, isolation levels, 2PC), distributed consensus (linearisability, Raft, Paxos, quorum).

Part 3 (Derived Data): Batch processing (MapReduce), stream processing (event sourcing, CQRS, stream joins), system-of-record vs derived data.

## Key Takeaways

- Data systems are fundamentally about reliability, scalability, and maintainability — NFRs before design
- Replication lag is the root of most distributed consistency problems — understand sync vs async replication
- CAP theorem is a useful framing but PACELC is more complete for everyday trade-offs
- Transactions do not require a single database — sagas and outbox patterns approximate ACID across services
- Event sourcing is powerful but not a silver bullet — it adds complexity that must be justified
- "Exactly-once" is a lie without careful design — at-least-once + idempotency is the practical answer
- Stream processing and batch processing are converging (Lambda/Kappa architecture)

## What it updated
- Formed basis for: [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/event-sourcing]], [[solution-arch/concepts/cqrs]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/message-queues]]
- Informed all scenario walkthroughs in [[solution-arch/scenarios/]]
