# Backend for Frontend (BFF)

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related concepts:** [[solution-arch/concepts/api-gateway]], [[solution-arch/patterns/event-driven-architecture]]

## What it solves
Different client types (mobile, web, 3rd-party) have different data requirements from the same backend services. A single general-purpose API becomes a compromise that serves nobody well — over-fetching for mobile, under-fetching for web. BFF provides a dedicated backend aggregation layer per client type.

## Template / Skeleton

```
WITHOUT BFF (single API — compromise):
  Mobile ────────────────────────▶ ┌──────────────┐
  Web App ───────────────────────▶ │ Generic API  │──▶ User Svc
  Partner ───────────────────────▶ │              │──▶ Order Svc
                                   │ Too much for │──▶ Product Svc
                                   │ mobile,      │
                                   │ too little   │
                                   │ for web      │
                                   └──────────────┘

WITH BFF:
  Mobile ──▶ Mobile BFF ──▶ Aggregates User + Order + Product
               (lean payload, mobile-optimised)
                              │ calls
  Web App──▶ Web BFF    ──▶  ├──▶ User Service
               (rich data,   ├──▶ Order Service
                web-optimised)├──▶ Product Service
                              │
  Partner──▶ Partner BFF──▶  └──▶ (public subset, versioned, rate-limited)
               (stable API,
                versioned)
```

## BFF Responsibilities

- Aggregate responses from multiple microservices into one call
- Transform data format per client needs (mobile gets compressed/minimal JSON)
- Handle client-specific auth flows
- Rate limiting per partner
- Translate between client protocol and internal protocol (REST → gRPC)
- Manage client-specific feature flags

## When to create a new BFF

```
Create a BFF when:
  ✅ Client has fundamentally different data needs
  ✅ Client has different auth/security requirements
  ✅ Client is owned by a different team
  ✅ Client API needs to be versioned independently

Don't create a BFF when:
  ❌ Clients need almost identical data
  ❌ Team capacity is thin (each BFF needs maintenance)
  ❌ Simple CRUD app — just use a shared API
```

## BFF Anti-Patterns

```
Fat BFF:
  BFF contains business logic → becomes a second monolith
  Fix: BFF should only aggregate and transform, not contain rules

Generic BFF:
  One BFF for all clients → same problem as no BFF
  Fix: one BFF per distinct client type

BFF as gateway:
  BFF does auth, rate limiting, routing → duplicate of API Gateway
  Fix: put cross-cutting concerns in API Gateway layer; BFF handles aggregation
```

## Signal phrases
- "Mobile needs less data than web"
- "Partner API must be versioned and rate-limited differently"
- "Aggregation of multiple service responses into one call"
- "GraphQL as a BFF" — GraphQL is a natural fit for BFF (client specifies exact fields needed)

## Complexity
Each BFF needs its own deployment, monitoring, and team ownership. Don't create more BFFs than you have teams to own them.

## Sources
- [[solution-arch/sources/clean-architecture]]
