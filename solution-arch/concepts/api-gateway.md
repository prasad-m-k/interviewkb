# API Gateway

**Topic:** [[solution-arch/topics/integration-patterns]], [[solution-arch/topics/security-architecture]]
**Related:** [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/load-balancing]], [[solution-arch/patterns/bff]]

## What it is
An API Gateway is a single entry point for all client requests. It handles cross-cutting concerns — authentication, rate limiting, routing, logging, SSL termination — so that individual services don't have to.

## How it works

```
                     ┌───────────────────────────────────┐
                     │           API Gateway             │
Mobile App ─────────▶│                                   │──▶ User Service
                     │  1. SSL Termination               │
Web App    ─────────▶│  2. AuthN (verify JWT)            │──▶ Order Service
                     │  3. Rate Limiting                 │
Partner API─────────▶│  4. Route by path/header          │──▶ Product Service
                     │  5. Request transformation        │
                     │  6. Response transformation       │──▶ Payment Service
                     │  7. Logging + tracing             │
                     │  8. Caching (optional)            │
                     └───────────────────────────────────┘
```

## Core Responsibilities

| Responsibility | Detail |
|---------------|--------|
| **Routing** | `/api/users` → User Service; `/api/orders` → Order Service |
| **Authentication** | Verify JWT, API key, or OAuth token |
| **Rate Limiting** | Per-client or per-endpoint throttling |
| **SSL Termination** | Handle HTTPS; backend uses HTTP |
| **Load Balancing** | Distribute to service instances |
| **Request Transformation** | Add headers, rewrite paths, convert JSON/XML |
| **Response Aggregation** | Merge responses from multiple services (see BFF) |
| **Caching** | Cache GET responses at gateway layer |
| **Observability** | Centralised access logging, metrics, tracing injection |

## Gateway vs Service Mesh vs Load Balancer

```
Load Balancer:    Routes traffic to instances of one service
                  Layer 4 or 7; no business logic

API Gateway:      Routes external client requests to many services
                  North-south traffic (client → cluster)
                  AuthN, rate limiting, API management

Service Mesh:     Routes service-to-service traffic
                  East-west traffic (service → service)
                  mTLS, circuit breaking, observability
```

They are complementary — most production systems use all three.

## BFF (Backend for Frontend)

One API Gateway per client type, each returning exactly what that client needs.

```
Mobile App   ──▶ Mobile BFF   ──▶ (User + Order + Product APIs, aggregated)
Web App      ──▶ Web BFF      ──▶ (Same services, different response shape)
Partner API  ──▶ Partner BFF  ──▶ (Public API surface, versioned)
```

Avoids one bloated gateway trying to satisfy all clients. Each BFF can evolve independently.

## Gateway Pattern Trade-offs

| Concern | Risk | Mitigation |
|---------|------|-----------|
| Single Point of Failure | Gateway down = all services unavailable | Multi-instance + health check |
| Bottleneck | All traffic funnelled through one place | Horizontal scale; avoid heavy logic |
| Coupling | Gateway knows all service routes | Service registration / discovery |
| Security boundary | Gateway is the exposed attack surface | WAF in front; keep it thin |

## Common interview angles
- "Why do you need an API Gateway if you have a load balancer?"
- "What is the difference between an API Gateway and a service mesh?"
- "When would you implement authentication in the gateway vs. in each service?" (Gateway: coarse-grained; service: fine-grained resource-level authZ)
- "What is a BFF pattern and why would you use it?"
- "How do you prevent the gateway from becoming a monolith?" (Keep it as configuration + routing; push logic into services)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
