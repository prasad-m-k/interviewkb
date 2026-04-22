# Service Discovery

**Topic:** [[solution-arch/topics/microservices]]
**Related:** [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/service-mesh]]

## What it is
In a microservices system, service instances start and stop dynamically (auto-scaling, rolling deploys, crashes). Service discovery is the mechanism by which a service finds the current network location (IP:port) of another service it needs to call — without hardcoding addresses.

## Why it's needed

```
Static world (hardcoded IPs — doesn't work):
  Order Service config: PAYMENT_URL = "10.0.1.5:8080"
  Payment instance crashes → new instance at 10.0.1.9:8080
  → Order Service still calling dead IP → failures

Dynamic world (service discovery):
  Order Service asks: "Where is Payment Service?"
  Registry answers: "10.0.1.9:8080, 10.0.1.12:8080"
  → Always routes to live instances
```

## Two Patterns

### Client-Side Discovery

The client queries the Service Registry directly, then load-balances and picks an instance itself.

```
  Service A
      │
      │ 1. Query: "where is Service B?"
      ▼
  Service Registry         2. Returns: [10.0.1.5, 10.0.1.9, 10.0.1.12]
  (Consul / Eureka)
      │
      │ 3. Client picks instance (round-robin / random / least-conn)
      ▼
  Service B instance (10.0.1.9:8080)
```

**Pros:** Client controls load-balancing algorithm; no extra hop.
**Cons:** Every service needs a discovery client library; language-coupled; harder to update routing logic.
**Examples:** Netflix Eureka + Ribbon (Java), Consul with client-side SDK.

---

### Server-Side Discovery

Client calls a fixed address (load balancer / router). The router queries the registry and forwards to an instance. Client is unaware of discovery.

```
  Service A
      │
      │ 1. Call: "payment-service:8080" (virtual address)
      ▼
  Load Balancer / Router
  (AWS ALB, Kubernetes Service, Nginx)
      │
      │ 2. Queries registry or uses its own table
      │ 3. Picks instance
      ▼
  Service B instance (10.0.1.9:8080)
```

**Pros:** Client is simple; no SDK needed; polyglot-friendly.
**Cons:** Load balancer is an additional hop; must be highly available.
**Examples:** Kubernetes Services + kube-proxy, AWS ECS Service Discovery, Istio.

---

## Service Registry

The central store of live service instance addresses and their health.

```
Instance registers on startup:
  POST /services  { name: "payment-service", ip: "10.0.1.9", port: 8080, health: "/health" }

Registry monitors health:
  Every 10s: GET 10.0.1.9:8080/health
  200 OK → instance stays in registry
  Timeout → instance removed from registry (deregistered)

Instance deregisters on shutdown:
  DELETE /services/payment-service/10.0.1.9
```

**Self-registration:** Service registers itself on startup (Consul agent pattern).
**Third-party registration:** Orchestrator (Kubernetes) registers/deregisters (Kubernetes Endpoints pattern).

## Kubernetes Service Discovery

Kubernetes is the most common implementation of server-side discovery:

```
Payment Service Pods:
  Pod 1: 172.16.0.1:8080
  Pod 2: 172.16.0.2:8080
  Pod 3: 172.16.0.3:8080
      │
      │ exposed via:
      ▼
Kubernetes Service: payment-service (ClusterIP: 10.96.0.10)
      │
      │ DNS: payment-service.default.svc.cluster.local → 10.96.0.10
      ▼
kube-proxy (iptables/IPVS) load-balances to healthy pods

Order Service calls:
  http://payment-service:8080/charge
  → DNS resolves to ClusterIP
  → kube-proxy routes to a healthy pod
  → Order Service never sees pod IPs
```

## Comparison

| | Client-Side | Server-Side |
|--|-------------|-------------|
| Registry query | Client | Router/LB |
| LB algorithm | Client controls | Router controls |
| Client complexity | Needs SDK | Simple HTTP call |
| Language coupling | Yes (per-language SDK) | None |
| Extra hop | No | Yes (router) |
| Examples | Eureka+Ribbon | K8s Services, Consul |

## Common interview angles
- "How does service A find service B in a microservices system?"
- "Client-side vs server-side service discovery — trade-offs?"
- "How does Kubernetes DNS-based service discovery work?"
- "What happens when a service instance crashes — how does discovery handle it?" (Health check fails → deregistered → removed from routing table)
- "What is the difference between a Kubernetes Service and an Ingress?" (Service = internal cluster routing; Ingress = external HTTP routing into the cluster)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
