# Solution Architecture Knowledge Base — Log

## [2026-04-22] update | Microservices architecture — gap fill

- Created topics: microservices (8 fallacies, principles, communication, data, observability, team topology, pitfalls)
- Created concepts: service-mesh (data/control plane, mTLS, Istio canary), service-discovery (client-side vs server-side, Kubernetes DNS)
- Created patterns: microservices-decomposition (by capability/DDD/volatility; anti-patterns; sizing), database-per-service (polyglot persistence; cross-service query strategies; saga/outbox reference), api-composition (sequential/parallel/scatter-gather; N+1 fix; vs CQRS)
- Updated scenarios: interview-questions — appended Q21–Q30 (distributed monolith, service boundaries, discovery, cross-service data, service mesh, saga walkthrough, choreography vs orchestration, mTLS auth, observability, cascading failure)
- Updated index: registered all 6 new files; updated interview-questions count
- Notes: All new pages interlinked with existing concepts (CQRS, saga, outbox, circuit-breaker, strangler-fig)

## [2026-04-22] ingest | Solution Architecture knowledge base — initial build

- Created: overview, index, log
- Topics (×7): architectural-styles, nfr-quality-attributes, scalability-and-reliability, data-architecture, integration-patterns, security-architecture, cloud-agnostic-principles
- Concepts (×13): cap-theorem, acid-vs-base, load-balancing, caching, message-queues, api-gateway, cqrs, event-sourcing, database-sharding, idempotency, rate-limiting, distributed-consensus (service-mesh stub)
- Patterns (×10): circuit-breaker, bulkhead, saga, outbox, strangler-fig, sidecar-ambassador, event-driven-architecture, bff, cqrs-event-sourcing, blue-green-canary
- Scenarios (×6): interview-questions (20 Q&A), design-url-shortener, design-rate-limiter, design-notification-system, monolith-to-microservices, high-availability-platform
- Diagrams (×2): c4-model (4 levels with ASCII), common-system-patterns (8 ASCII diagrams)
- Reference: flashcards (40+ Anki cards)
- Sources: designing-data-intensive-applications, clean-architecture
- Notes: Cloud-agnostic throughout; ASCII diagrams for all major patterns; interview Q&A covers fundamentals through senior-level
