# Solution Architecture — Knowledge Base Index
Last updated: 2026-04-22

## Overview
- [[solution-arch/overview]] — Scope, quality attributes, architect's decision framework

## Topics
- [[solution-arch/topics/architectural-styles]] — Monolith, microservices, event-driven, serverless, SOA
- [[solution-arch/topics/nfr-quality-attributes]] — Availability, scalability, reliability, security, maintainability
- [[solution-arch/topics/scalability-and-reliability]] — Scaling models, replication, consensus, failure modes
- [[solution-arch/topics/data-architecture]] — Storage types, consistency models, data mesh, lakehouse
- [[solution-arch/topics/security-architecture]] — Zero trust, IAM, encryption, threat modeling
- [[solution-arch/topics/integration-patterns]] — Sync vs async, API styles, messaging, ETL/ELT
- [[solution-arch/topics/cloud-agnostic-principles]] — 12-Factor, IaC, GitOps, portability
- [[solution-arch/topics/microservices]] — 8 fallacies, principles, communication, data management, team topology

## Concepts
- [[solution-arch/concepts/cap-theorem]] — Consistency, availability, partition tolerance trade-off
- [[solution-arch/concepts/acid-vs-base]] — Strong vs eventual consistency; when to use each
- [[solution-arch/concepts/load-balancing]] — L4 vs L7, algorithms, health checks, sticky sessions
- [[solution-arch/concepts/caching]] — Strategies, eviction policies, cache invalidation, CDN
- [[solution-arch/concepts/message-queues]] — Queue vs pub/sub, ordering, at-least-once, exactly-once
- [[solution-arch/concepts/api-gateway]] — Routing, auth, rate limiting, transformation, BFF
- [[solution-arch/concepts/service-mesh]] — Sidecar proxy, mTLS, observability, traffic management
- [[solution-arch/concepts/service-discovery]] — Client-side vs server-side, Kubernetes DNS, health checks
- [[solution-arch/concepts/database-sharding]] — Horizontal partitioning, shard keys, resharding
- [[solution-arch/concepts/event-sourcing]] — Append-only log as source of truth; replay
- [[solution-arch/concepts/cqrs]] — Command Query Responsibility Segregation; read/write split
- [[solution-arch/concepts/idempotency]] — Designing idempotent APIs and operations
- [[solution-arch/concepts/rate-limiting]] — Token bucket, leaky bucket, sliding window algorithms
- [[solution-arch/concepts/distributed-consensus]] — Paxos, Raft, leader election, split-brain

## Patterns
- [[solution-arch/patterns/circuit-breaker]] — Fail fast; prevent cascading failure
- [[solution-arch/patterns/bulkhead]] — Isolate failure to a resource pool
- [[solution-arch/patterns/saga]] — Distributed transaction via choreography or orchestration
- [[solution-arch/patterns/outbox]] — Reliable event publishing without 2PC
- [[solution-arch/patterns/strangler-fig]] — Incremental monolith migration
- [[solution-arch/patterns/sidecar-ambassador]] — Proxy patterns for cross-cutting concerns
- [[solution-arch/patterns/event-driven-architecture]] — Event sourcing, streaming, reactive systems
- [[solution-arch/patterns/bff]] — Backend for Frontend; per-client API aggregation
- [[solution-arch/patterns/cqrs-event-sourcing]] — Combined read/write split with event log
- [[solution-arch/patterns/blue-green-canary]] — Zero-downtime deployment strategies
- [[solution-arch/patterns/microservices-decomposition]] — Decompose by capability, DDD bounded context, volatility
- [[solution-arch/patterns/database-per-service]] — Service data ownership, polyglot persistence, cross-service queries
- [[solution-arch/patterns/api-composition]] — Aggregating cross-service queries; N+1 problem; vs CQRS

## Scenarios (Interview)
- [[solution-arch/scenarios/interview-questions]] — 30 scenario-based SA interview questions with answers (incl. microservices Q21–Q30)
- [[solution-arch/scenarios/design-url-shortener]] — Full system design walkthrough
- [[solution-arch/scenarios/design-rate-limiter]] — Distributed rate limiter design
- [[solution-arch/scenarios/design-notification-system]] — Multi-channel fan-out at scale
- [[solution-arch/scenarios/monolith-to-microservices]] — Migration strategy and pitfalls
- [[solution-arch/scenarios/high-availability-platform]] — 99.99% uptime design decisions

## Diagrams
- [[solution-arch/diagrams/c4-model]] — Context, Container, Component, Code diagram levels
- [[solution-arch/diagrams/common-system-patterns]] — ASCII reference diagrams for common architectures

## Reference
- [[solution-arch/flashcards]] — Anki-ready flashcard deck (40+ cards)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]] — Kleppmann; storage, replication, consensus
- [[solution-arch/sources/clean-architecture]] — Martin; dependency rules, boundaries, use cases
