# C4 Model — Architecture Diagrams

**Related:** [[solution-arch/overview]], [[solution-arch/diagrams/common-system-patterns]]

The C4 model (Simon Brown) provides four levels of abstraction for describing software architecture. Each level zooms in one step deeper.

---

## Level 1: System Context Diagram

Highest level — what does the system do and who uses it?

```
┌─────────────────────────────────────────────────────────────┐
│                     System Context                          │
│                                                             │
│   ┌───────────┐         ┌─────────────────────────────┐    │
│   │  Customer │         │       E-Commerce            │    │
│   │  (Person) │────────▶│       Platform              │    │
│   └───────────┘  shops  │   [Software System]         │    │
│                         └───────────┬─────────────────┘    │
│   ┌───────────┐                     │                       │
│   │  Admin    │──────manages──────▶ │                       │
│   │  (Person) │                     │                       │
│   └───────────┘           ┌─────────┴──────────────────┐   │
│                           │                            │   │
│                  ┌────────▼──────┐      ┌──────────────▼─┐ │
│                  │ Payment       │      │  Shipping      │ │
│                  │ Gateway       │      │  Provider      │ │
│                  │ [External]    │      │  [External]    │ │
│                  └───────────────┘      └────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Purpose:** Non-technical stakeholders can understand this. No implementation detail.

---

## Level 2: Container Diagram

Zooms into the system — shows deployable units (containers = process/app/DB, not Docker).

```
┌───────────────────────────────────────────────────────────────────────┐
│                          E-Commerce Platform                          │
│                                                                       │
│  ┌─────────────────┐     ┌─────────────────┐  ┌──────────────────┐   │
│  │   Web App       │     │  Mobile App     │  │  Admin SPA       │   │
│  │  [React SPA]    │     │  [React Native] │  │  [React SPA]     │   │
│  └────────┬────────┘     └────────┬────────┘  └────────┬─────────┘   │
│           │                       │                    │             │
│           └───────────────────────┼────────────────────┘             │
│                                   │ HTTPS                            │
│                          ┌────────▼────────┐                         │
│                          │   API Gateway   │                         │
│                          │  [nginx/Kong]   │                         │
│                          └────────┬────────┘                         │
│                                   │                                  │
│             ┌─────────────────────┼───────────────────┐             │
│             │                     │                   │             │
│   ┌─────────▼──────┐   ┌──────────▼──────┐  ┌────────▼──────────┐  │
│   │  User Service  │   │  Order Service  │  │  Product Service  │  │
│   │  [Go + gRPC]   │   │  [Java/Spring]  │  │  [Node.js REST]   │  │
│   └─────────┬──────┘   └──────────┬──────┘  └────────┬──────────┘  │
│             │                     │                   │             │
│   ┌─────────▼──────┐   ┌──────────▼──────┐  ┌────────▼──────────┐  │
│   │   Users DB     │   │   Orders DB     │  │  Products DB      │  │
│   │  [PostgreSQL]  │   │  [PostgreSQL]   │  │  [MongoDB]        │  │
│   └────────────────┘   └─────────────────┘  └───────────────────┘  │
│                                                                       │
│                        ┌───────────────────┐                         │
│                        │   Message Broker  │                         │
│                        │   [Kafka]         │                         │
│                        └───────────────────┘                         │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Level 3: Component Diagram

Zooms into one container — shows internal components.

```
┌──────────────────────────────────────────────────────────────┐
│                       Order Service                          │
│                                                              │
│  ┌────────────────────┐         ┌──────────────────────────┐ │
│  │   REST Controller  │──────▶  │  Order Application       │ │
│  │   (HTTP handler)   │         │  Service (use cases)     │ │
│  └────────────────────┘         └─────────┬────────────────┘ │
│                                           │                  │
│                                  ┌────────┼──────────┐       │
│                                  ▼        ▼          ▼       │
│                          ┌──────────┐ ┌───────┐ ┌────────┐  │
│                          │ Order    │ │Payment│ │Publish │  │
│                          │Repository│ │Client │ │Events  │  │
│                          │ (DB)     │ │(gRPC) │ │(Kafka) │  │
│                          └──────────┘ └───────┘ └────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Level 4: Code Diagram

Class/function level. Usually just use your IDE for this. Skip in interviews.

---

## Interview Usage

In system design interviews, draw Level 1 first (context), then Level 2 (containers). Level 3 only if asked about a specific service's internals.

```
Interview sequence:
  1. Requirements + NFRs (5 min)
  2. Level 1: System boundary, external actors (2 min)
  3. Level 2: Key containers + data stores + messaging (10 min)
  4. Deep-dive: one component in Level 3 detail (5 min)
  5. Trade-off discussion (5 min)
```

---

## Notation Conventions

```
┌─────────────────────────┐
│ Name                    │  ← Solid box = internal to system
│ [Technology]            │
│ Description             │
└─────────────────────────┘

┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
│ External System         │  ← Dashed box = external to system
│ [SaaS / 3rd party]      │
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘

Person ──[rel: description]──▶ System

Database ──[reads/writes]──▶ Storage
```

## Sources
- [[solution-arch/sources/clean-architecture]]
