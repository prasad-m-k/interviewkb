# Solution Architecture — Knowledge Base Index
Last updated: 2026-08-31 (rev. 7)

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
- [[solution-arch/topics/ai-solution-architecture]] — AI SA role/skill map, maturity model, prompting vs RAG vs fine-tuning vs agentic decision framework
- [[solution-arch/topics/agentic-ai-architecture]] — Agent loop, design patterns, multi-agent systems, memory, guardrails, evals
- [[solution-arch/topics/openai-platform-architecture]] — Chat Completions vs Responses vs Assistants API, tool calling, fine-tuning, Azure OpenAI
- [[solution-arch/topics/llm-application-architecture]] — Enterprise RAG integration, context engineering, LLMOps, streaming, caching
- [[solution-arch/topics/ai-governance-responsible-ai]] — AI risk categories, EU AI Act, model risk management, governance stack
- [[solution-arch/topics/enterprise-architecture-frameworks]] — EA vs SA vs TA, TOGAF, cloud Well-Architected Frameworks, build vs buy
- [[solution-arch/topics/cost-architecture-finops]] — FinOps lifecycle, cloud cost levers, LLM/token cost optimization

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
- [[solution-arch/concepts/vector-databases]] — HNSW/IVF ANN indexes, pgvector vs dedicated vector DB, re-embedding
- [[solution-arch/concepts/federated-query-engines]] — F1 Query-style distributed SQL: table interleaving, CBO pushdown, hedged requests, DAG execution over relational + lake + vector sources
- [[solution-arch/concepts/prompt-engineering-and-context-design]] — Context window budget, few-shot/CoT/structured output, cache-friendly ordering
- [[solution-arch/concepts/function-calling-and-tool-use]] — Tool schema design, validation, idempotent side-effecting tools
- [[solution-arch/concepts/ai-guardrails-and-safety]] — Input/output guardrails, prompt injection defense-in-depth
- [[solution-arch/concepts/model-context-protocol-mcp]] — N×M integration problem, MCP client/server/host architecture
- [[solution-arch/concepts/llm-observability-and-evals]] — Tracing, LLM-as-judge, trajectory evals, eval-gated deploys
- [[solution-arch/concepts/network-architecture-fundamentals]] — DNS/GSLB, CDN cache-key design, HTTP/2 vs HTTP/3, VPC segmentation (SA design view; handshake mechanics in [[sre/concepts/networking-fundamentals]])
- [[solution-arch/concepts/rest-api-design-principles]] — Resource modeling, HTTP verb safety/idempotency, status codes, Richardson Maturity Model, versioning, cursor pagination, error format
- [[solution-arch/concepts/http-status-codes]] — Full 1xx-5xx reference with examples: 401 vs 403, 400 vs 422, 502 vs 504, 301/302 vs 307/308 redirect semantics
- [[solution-arch/concepts/http-headers]] — Caching/conditional-request, CORS preflight, security (HSTS/CSP), auth, proxy (X-Forwarded-*), and tracing headers with examples
- [[solution-arch/concepts/sql-fundamentals]] — Joins (with worked examples), WHERE vs HAVING, finding duplicates, composite keys, N+1 problem, basic SQL interview question bank
- [[solution-arch/concepts/distributed-caching]] — Consistent hashing, client-side vs proxy-based sharding, replication topologies, cross-region remote markers, lease-based stampede protection
- [[solution-arch/concepts/azure-ai-content-safety]] — Harm categories/severity, Prompt Shields, groundedness detection, protected material detection; Azure OpenAI's default content filter #ResponsibleAI
- [[solution-arch/concepts/responsible-ai-sdlc-governance]] — Governing AI-generated requirements/design docs/code in the SDLC (distinct from product-facing RAI) #ResponsibleAI

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
- [[solution-arch/patterns/agentic-workflow-patterns]] — Prompt chaining, routing, parallelization, orchestrator-worker, evaluator-optimizer, autonomous agent
- [[solution-arch/patterns/multi-agent-orchestration]] — Supervisor, peer handoff, blackboard topologies; when multi-agent is worth it
- [[solution-arch/patterns/human-in-the-loop]] — Risk-tiered approval gates for high-stakes agent actions
- [[solution-arch/patterns/rag-enterprise-integration]] — Access-control-aware RAG, freshness strategy (enterprise-integration view; ML view in [[ml/patterns/rag-pattern]])
- [[solution-arch/patterns/ai-gateway-pattern]] — LLM traffic chokepoint: routing, guardrails, cost metering, provider failover
- [[solution-arch/patterns/hot-path-first-design]] — Read/write + display/decision-read classification, endpoint classification matrix, outbox-driven cache invalidation, 5-step methodology

## Scenarios (Interview)
- [[solution-arch/scenarios/interview-questions]] — 30 scenario-based SA interview questions with answers (incl. microservices Q21–Q30)
- [[solution-arch/scenarios/design-url-shortener]] — Full system design walkthrough
- [[solution-arch/scenarios/design-rate-limiter]] — Distributed rate limiter design
- [[solution-arch/scenarios/design-tiered-caching-platform]] — Tier-0 multi-tier caching platform (in-memory + distributed, zone-aware replication, cost efficiency at extreme scale)
- [[solution-arch/scenarios/design-notification-system]] — Multi-channel fan-out at scale
- [[solution-arch/scenarios/monolith-to-microservices]] — Migration strategy and pitfalls
- [[solution-arch/scenarios/high-availability-platform]] — 99.99% uptime design decisions
- [[solution-arch/scenarios/design-enterprise-rag-system]] — Access-control-aware enterprise RAG assistant, full walkthrough
- [[solution-arch/scenarios/design-agentic-customer-support-system]] — Multi-agent support system with refund guardrails and HITL
- [[solution-arch/scenarios/ai-solution-architect-interview-questions]] — 28 Q&A: Agentic AI, OpenAI platform, RAG, MCP, governance, cost
- [[solution-arch/scenarios/ai-data-platform-system-design]] — AI Foundations track: RAG metadata store, ML feature store, F1 Query-style federated engine, zero-downtime embedding-schema migration, LLM workload isolation

## Companies
- [[solution-arch/companies/microsoft-coreai-responsible-ai]] — Principal SWE, Responsible AI (CoreAI): role snapshot, interview loop, practice questions, culture signals #ResponsibleAI

## Diagrams
- [[c4-model]] — Context, Container, Component, Code diagram levels
- [[common-system-patterns]] — ASCII reference diagrams for common architectures
- [[agentic-ai-reference-diagrams]] — ASCII diagrams: agent loop, RAG pipeline, multi-agent, AI gateway, guardrails, HITL, MCP, LLMOps

## Reference
- [[solution-arch/flashcards]] — Anki-ready flashcard deck (40+ cards, incl. AI Solution Architecture section)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]] — Kleppmann; storage, replication, consensus
- [[solution-arch/sources/clean-architecture]] — Martin; dependency rules, boundaries, use cases
- [[solution-arch/sources/building-effective-agents-anthropic]] — Anthropic; workflow vs agent distinction, composable agentic patterns
- [[solution-arch/sources/hot-path-first-system-design]] — Newsletter; hot-path-first methodology, read/write and display/decision-read separation
