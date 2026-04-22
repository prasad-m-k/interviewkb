# Service Mesh

**Topic:** [[solution-arch/topics/microservices]], [[solution-arch/topics/security-architecture]]
**Related:** [[solution-arch/patterns/sidecar-ambassador]], [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/load-balancing]]

## What it is
A service mesh is a dedicated infrastructure layer for handling **service-to-service (east-west) communication** inside a microservices cluster. It adds observability, security (mTLS), and traffic management transparently — without changing application code — by injecting a sidecar proxy alongside each service.

## How it works

```
Without service mesh:
  Service A ──HTTP──▶ Service B   (no encryption, no tracing, no retry)

With service mesh (sidecar per pod):
  ┌─────────────────────────┐      ┌─────────────────────────┐
  │  Pod A                  │      │  Pod B                  │
  │ ┌──────────┐ ┌────────┐ │      │ ┌────────┐ ┌──────────┐ │
  │ │App (8080)│◀│Sidecar │◀┼─mTLS─┼▶│Sidecar │▶│App (8080)│ │
  │ │          │ │(Envoy) │ │      │ │(Envoy) │ │          │ │
  │ └──────────┘ └────────┘ │      │ └────────┘ └──────────┘ │
  └─────────────────────────┘      └─────────────────────────┘
         │                                     │
         └──────────── Control Plane ──────────┘
                      (Istiod / Linkerd)
                      distributes config, certs, policy
```

Traffic from App A exits to its sidecar → encrypted via mTLS → arrives at Pod B's sidecar → forwarded to App B. App code never touches TLS or retry logic.

## Data Plane vs Control Plane

```
Data Plane (sidecars — Envoy, nginx):
  → Intercept all ingress + egress traffic per pod
  → Enforce mTLS, retries, timeouts, circuit breaking
  → Emit metrics and traces per request
  → Applied to: every service pod in the mesh

Control Plane (Istio Istiod / Linkerd control plane):
  → Distributes routing rules to all sidecars
  → Issues and rotates mTLS certificates (SPIFFE/SPIRE)
  → Central config: VirtualService, DestinationRule (Istio CRDs)
  → Aggregates telemetry metadata
```

## Core Capabilities

| Capability | Detail |
|-----------|--------|
| **mTLS** | Automatic mutual TLS between all services; identity via X.509 certs |
| **Traffic management** | Canary routing (10% to v2), retries, timeouts, fault injection |
| **Observability** | Per-request metrics (RPS, latency, error rate) and traces — without code changes |
| **Load balancing** | L7 round-robin, least-request, consistent hashing per service |
| **Circuit breaking** | Outlier detection: eject unhealthy instances from load balancing pool |
| **Access policy** | AuthorizationPolicy: allow/deny traffic between specific services |

## Service Mesh vs API Gateway

```
API Gateway (North-South):             Service Mesh (East-West):
  External clients → cluster              Service → Service (internal)
  AuthN, rate limiting, routing          mTLS, observability, retries
  One per cluster (or per BFF)           One sidecar per pod
  Examples: Kong, AWS ALB, Nginx         Examples: Istio, Linkerd, Consul Connect
```

They are **complementary**, not alternatives. Most production clusters use both.

## Traffic Management Example (Canary via Istio)

```yaml
# Route 90% to v1, 10% to v2 (canary)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  http:
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10
```

No load balancer config change needed — the sidecar handles traffic splitting.

## Complexity and Cost

```
Benefits:
  ✅ Zero code changes for mTLS, retries, tracing
  ✅ Consistent policy across all services
  ✅ Canary/A-B traffic splitting per service
  ✅ Certificate rotation without service restarts

Costs:
  ⚠️  Additional latency: ~1-5ms per hop (sidecar overhead)
  ⚠️  Memory per pod: ~50-100MB for Envoy sidecar
  ⚠️  Operational complexity: Istio has steep learning curve
  ⚠️  CRD proliferation in Kubernetes
```

**Decision:** Use a service mesh when you have 10+ services and need consistent mTLS + observability. For fewer services, handle these in code or via a simpler proxy.

## Common interview angles
- "What is the difference between a service mesh and an API gateway?"
- "How does a service mesh implement mTLS without changing service code?" (sidecar intercepts all traffic; Control Plane distributes certs)
- "What is the data plane vs control plane in Istio?" (data = Envoy sidecars; control = Istiod)
- "How do you do a canary release using Istio?"
- "What overhead does a service mesh add?" (latency ~1-5ms, memory ~50-100MB per pod)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
