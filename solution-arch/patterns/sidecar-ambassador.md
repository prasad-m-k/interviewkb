# Sidecar & Ambassador Patterns

**Topic:** [[solution-arch/topics/integration-patterns]]
**Related concepts:** [[solution-arch/concepts/service-mesh]], [[solution-arch/concepts/api-gateway]]

## What it solves
Cross-cutting concerns (logging, auth, TLS, circuit breaking, retry logic, service discovery) scattered across every service in a microservices system. The sidecar/ambassador patterns attach a proxy helper container that handles these concerns, keeping service code clean.

---

## Sidecar Pattern

A helper container deployed alongside the main application container in the same pod/host. They share the same network namespace (same localhost).

```
┌─────────────────────────────────────────────────┐
│                    Pod / VM                     │
│                                                 │
│  ┌──────────────────┐  ┌──────────────────────┐ │
│  │  Main App        │  │   Sidecar Proxy      │ │
│  │  (business logic)│  │   (Envoy / Nginx)    │ │
│  │                  │  │                      │ │
│  │  Listens :8080   │  │  - mTLS              │ │
│  │                  │◀─┤  - Circuit breaking  │ │
│  │  No knowledge    │  │  - Retry logic       │ │
│  │  of TLS,         │  │  - Metrics           │ │
│  │  retries, etc.   │  │  - Distributed trace │ │
│  └──────────────────┘  └──────────────────────┘ │
│        localhost:8080          :15001 (inbound)  │
└─────────────────────────────────────────────────┘
```

**Service mesh** (Istio, Linkerd) is a system-wide application of the sidecar pattern — every pod gets a sidecar injected automatically.

**Use cases:**
- TLS termination and mTLS between services
- Circuit breaking without touching app code
- Distributed tracing (inject trace headers)
- Access logging
- Metric scraping

---

## Ambassador Pattern

The sidecar acts specifically as a proxy for **outbound** calls — handling service discovery, routing, and protocol translation on behalf of the main app.

```
┌──────────────────────────────────────────────────┐
│                  Pod / VM                        │
│                                                  │
│  ┌──────────────┐     ┌────────────────────────┐ │
│  │   App        │     │   Ambassador Proxy     │ │
│  │              │     │                        │ │
│  │ calls        │────▶│ - Service discovery    │──▶ Service A
│  │ localhost    │     │ - Load balancing       │──▶ Service B
│  │ :9090        │     │ - Auth token injection │
│  │              │     │ - Protocol translation │
│  │ (unaware of  │     │   (REST → gRPC)        │
│  │ real targets)│     │                        │ │
│  └──────────────┘     └────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**Use cases:**
- Abstracting service discovery from app code
- Injecting auth headers automatically
- Protocol bridging (legacy app speaks HTTP/1.1; backend speaks gRPC)
- Shared retry / timeout policy for all outbound calls

---

## Adapter Pattern (Sidecar variant)

Translates the output of the main app into a standardised format expected by the platform.

```
┌──────────────────────────────────────────────────┐
│   App emits logs in custom format               │
│   ┌────────────┐   ┌──────────────────────────┐ │
│   │ Legacy App │──▶│  Adapter Sidecar         │──▶ Centralised log platform
│   │ log format │   │  (translate to JSON/     │    (expects JSON)
│   │ is custom  │   │   structured format)     │ │
│   └────────────┘   └──────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

---

## Comparison

| Pattern | Direction | Responsibility | Example |
|---------|-----------|---------------|---------|
| Sidecar | Both | General proxy, cross-cutting | Envoy, Linkerd-proxy |
| Ambassador | Outbound | Proxy for outbound calls | Envoy as egress proxy |
| Adapter | Outbound | Format translation | Log/metric adapter |

---

## Signal phrases
- "Avoid code duplication for cross-cutting concerns across services"
- "Retrofit observability/security without changing service code"
- "Service mesh infrastructure"
- "Legacy app that can't be changed needs new capability"

## Complexity
Each sidecar adds a container, CPU/memory overhead (~10–50 MB), and one more hop in the network path. For high-throughput services, measure the latency impact.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
