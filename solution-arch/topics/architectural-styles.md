# Architectural Styles

**Related:** [[solution-arch/overview]], [[solution-arch/topics/nfr-quality-attributes]], [[solution-arch/patterns/event-driven-architecture]]

Architectural style = the top-level structural pattern: how components are divided, how they communicate, and what constraints apply.

---

## Monolithic

All functionality in a single deployable unit.

```
┌─────────────────────────────────┐
│           MONOLITH              │
│  ┌────────┐  ┌───────────────┐  │
│  │  UI    │  │  Business     │  │
│  │ Layer  │  │  Logic        │  │
│  └────────┘  └───────┬───────┘  │
│                      │          │
│             ┌────────▼───────┐  │
│             │   Data Layer   │  │
│             └────────────────┘  │
└─────────────────────┬───────────┘
                      │
               ┌──────▼──────┐
               │  Single DB  │
               └─────────────┘
```

**Pros:** Simple to develop, test, deploy early on. No network calls between components.
**Cons:** Scales as a unit; deploy everything for one change; team coupling; language lock-in.
**When:** Early stage product, small team, unclear domain boundaries.

---

## Microservices

Application decomposed into small, independently deployable services, each owning its data.

```
          API Gateway
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
┌────────┐ ┌──────┐ ┌─────────┐
│ User   │ │Order │ │Payment  │
│Service │ │Svc   │ │Service  │
└───┬────┘ └──┬───┘ └────┬────┘
    │         │           │
┌───▼──┐  ┌───▼──┐  ┌────▼──┐
│UserDB│  │OrdDB │  │PayDB  │
└──────┘  └──────┘  └───────┘
```

**Pros:** Independent deploy, scale, and failure. Team autonomy. Polyglot.
**Cons:** Distributed systems complexity (network, consistency, observability). Operational overhead.
**When:** Large teams, distinct bounded contexts, need independent scaling.

---

## Event-Driven Architecture (EDA)

Services communicate through events on a message broker. Producers don't know consumers.

```
Producer ──▶ Event Broker ──▶ Consumer A
  (Order)    (Kafka/Rabbit)   (Inventory)
                    │
                    └──────▶ Consumer B
                              (Notification)
```

**Pros:** Loose coupling, natural async, elastic fan-out, audit log.
**Cons:** Eventual consistency, hard to trace flows, no clear error owner.
**When:** High fan-out, audit requirements, workflow orchestration across services.

---

## Serverless / FaaS

Business logic as stateless functions triggered by events. Infrastructure fully managed.

```
HTTP Request ──▶ Function Runtime ──▶ Managed DB
Cron Event  ──▶ Function Runtime ──▶ Managed Queue
File Upload ──▶ Function Runtime ──▶ Object Store
```

**Pros:** Zero infra management; pay-per-invocation; infinite scale (per provider).
**Cons:** Cold starts; stateless (state must live outside); vendor lock-in; 15-min max execution.
**When:** Intermittent workloads, event processing, glue code between managed services.

---

## Service-Oriented Architecture (SOA)

Predecessor to microservices. Services share a common Enterprise Service Bus (ESB).

```
Service A ─┐
Service B ─┤─▶ Enterprise Service Bus (ESB) ─┐
Service C ─┘    (routing, transformation,    ├─▶ Service D
                 orchestration, logging)      └─▶ Service E
```

**Cons:** ESB becomes a bottleneck and single point of failure. Heavy XML/WS-* protocols.
**vs Microservices:** SOA: smart pipes (ESB), dumb endpoints. Microservices: dumb pipes (queue/HTTP), smart endpoints.

---

## Layered (N-Tier)

Strict horizontal layers: Presentation → Business Logic → Data Access → Database.

Each layer only calls the one directly below. Classic web app pattern.

**Pros:** Familiar, clear separation, easy to test each layer.
**Cons:** Tends toward "big ball of mud"; adding a feature touches all layers; doesn't map to domain.

---

## Hexagonal (Ports and Adapters)

Core domain logic at the centre with no framework dependencies. External systems plug in via adapters.

```
               ┌──────────────────────┐
  HTTP ──────▶ │  Input Adapter       │
               │    ┌────────────┐    │
  gRPC ──────▶ │    │   DOMAIN   │    │ ──▶ DB Adapter ──▶ PostgreSQL
               │    │   CORE     │    │
  Queue ─────▶ │    └────────────┘    │ ──▶ Email Adapter ──▶ SMTP
               │  Output Adapter      │
               └──────────────────────┘
```

**Benefit:** Core logic is testable without infrastructure. Swap adapters without touching the domain.

---

## Comparison Matrix

| Style | Coupling | Scalability | Complexity | Best team size |
|-------|----------|-------------|------------|---------------|
| Monolith | High | Unit-level | Low | 1–8 |
| Layered | Medium | Unit-level | Low | 5–15 |
| SOA | Medium | Service | High | 20+ |
| Microservices | Low | Per-service | Very high | 10+ per service |
| EDA | Very low | Per-consumer | High | Any |
| Serverless | Very low | Infinite | Medium | Any |
| Hexagonal | Low (internal) | Depends | Medium | Any |

## Sources
- [[solution-arch/sources/clean-architecture]]
- [[solution-arch/sources/designing-data-intensive-applications]]
