# Microservices Decomposition Patterns

**Topic:** [[solution-arch/topics/microservices]]
**Related concepts:** [[solution-arch/concepts/service-discovery]], [[solution-arch/patterns/database-per-service]], [[solution-arch/patterns/strangler-fig]]

## What it solves
How to draw the right boundaries between microservices. Wrong decomposition is the most common and most expensive microservices mistake — fixing it means rewriting services and migrating data.

---

## Pattern 1: Decompose by Business Capability

Each service owns one business capability — a thing the business does.

```
Business capabilities of an e-commerce company:
  ┌──────────────────────────────────────────────┐
  │                 E-Commerce                   │
  │                                              │
  │  Account     Order     Product    Shipping   │
  │  Management  Management Catalogue Management │
  │                                              │
  │  Payment     Inventory  Notification Review  │
  │  Processing  Management Service    Service   │
  └──────────────────────────────────────────────┘

  ↓ each becomes a service

  Account Service    → registration, login, profile
  Order Service      → place, track, cancel orders
  Product Service    → catalogue, pricing, descriptions
  Shipping Service   → fulfilment, tracking, carriers
  Payment Service    → charge, refund, payment methods
  Inventory Service  → stock levels, reservations
  Notification Svc   → email, SMS, push
  Review Service     → product reviews, ratings
```

**Rule:** A service changes when the business capability it represents changes. If two things always change together, they belong in the same service.

---

## Pattern 2: Decompose by Subdomain (DDD)

Domain-Driven Design identifies bounded contexts within a domain. Each bounded context becomes a service.

```
Domain: E-Commerce

Core Subdomains (competitive advantage — invest here):
  Order Management    → complex business rules; own this
  Pricing Engine      → dynamic pricing algorithms; own this

Supporting Subdomains (needed but not differentiating):
  Notification        → use a vendor (SendGrid); thin wrapper service
  Authentication      → use Auth0 / Keycloak; don't build from scratch

Generic Subdomains (commodity — buy/use off-shelf):
  Payments            → use Stripe; don't build
  Shipping            → use FedEx/UPS API; thin adapter
```

DDD tools for finding boundaries:
- **Event Storming:** workshop to discover domain events and aggregates
- **Context Map:** diagram showing how bounded contexts relate (shared kernel, anti-corruption layer, etc.)
- **Ubiquitous Language:** each bounded context has its own language (e.g., "Order" means different things in Sales vs. Warehouse)

---

## Pattern 3: Decompose by Volatility

Services that change frequently together are candidates to be one service. Services that change independently should be separate.

```
High volatility (changes weekly): Recommendation engine, Pricing, A/B tests
  → Make these services; can deploy without touching others

Low volatility (changes rarely): Authentication, Address validation
  → Keep simple; may not justify full microservice overhead
```

---

## Anti-Patterns: Wrong Decomposition

```
❌ Decompose by noun (entity):
  "User Service" owns ALL user data for ALL contexts
  → Order context needs user address; Payment context needs billing info
  → Service becomes a "data service" that everyone calls for everything
  → God service: high coupling, becomes a bottleneck

❌ Decompose by technical tier:
  Data Service / Business Service / Presentation Service
  → Every feature change touches all three layers
  → Services are not independently deployable (co-deploy required)
  → Distributed monolith

❌ Premature decomposition:
  Splitting before domain is understood → wrong boundaries
  → Expensive to fix: data migration + API contract changes
  → Start modular monolith; split when boundaries prove stable
```

---

## Strangler Fig for Decomposition

When migrating from a monolith, extract services using the strangler fig pattern:

```
1. Identify bounded context to extract (start with leaf/low-coupling)
2. Build new service alongside monolith
3. Route traffic to new service via API Gateway facade
4. Validate: shadow traffic, compare responses
5. Cut over: 100% traffic to new service
6. Remove code from monolith
```

See: [[solution-arch/patterns/strangler-fig]]

---

## Sizing Guidelines

```
"How big should a microservice be?"

Too small (nanoservice):
  - Constant cross-service calls for simple operations
  - High coordination overhead
  - Transactions always span multiple services

Too big (mini-monolith):
  - Can't deploy without coordinating with other teams
  - Can't scale one part independently
  - Same team scaling problems as the original monolith

Right size:
  - One team can fully own, understand, and maintain it
  - Can be rewritten in 2 weeks (Sam Newman's heuristic)
  - Can be deployed independently
  - Most operations touch only one service's data
```

---

## Signal phrases
- "How do you split a monolith into microservices?"
- "How do you decide the right service boundaries?"
- "What is a bounded context?"
- "What is Domain-Driven Design and how does it apply to microservices?"

## Sources
- [[solution-arch/sources/clean-architecture]]
- [[solution-arch/sources/designing-data-intensive-applications]]
