# Concept: Service Mesh

**Topic:** [[devops/topics/containers-orchestration]]
**Related:** [[devops/concepts/kubernetes-architecture]], [[devops/topics/security-devsecops]], [[devops/concepts/observability-pillars]]

## What it is

A service mesh is an infrastructure layer that handles service-to-service network communication transparently — without changes to application code. A sidecar proxy (typically Envoy) is injected alongside every pod, and all traffic between services flows through these proxies instead of directly.

```
Without mesh:                    With mesh:
Pod A ──────────────▶ Pod B      Pod A → [Envoy] ──────▶ [Envoy] → Pod B
  (app code handles TLS,           (mesh handles TLS, retries,
   retries, timeouts)               timeouts, tracing, metrics)
```

## How it works

### Data plane vs control plane

```
Control Plane (Istiod / Linkerd controller)
  - Distributes configuration to all proxies
  - Manages certificate authority (issues mTLS certs)
  - Handles service discovery

Data Plane (Envoy sidecars)
  - Intercepts all inbound + outbound traffic
  - Enforces policies (mTLS, retries, circuit breaking)
  - Emits metrics + traces for every request
```

### Sidecar injection

```yaml
# Automatic injection: label the namespace
kubectl label namespace prod istio-injection=enabled

# K8s MutatingAdmissionWebhook intercepts pod creation
# → injects initContainer (iptables rules) + Envoy sidecar
# → app pod now has 2 containers: app + envoy
```

### Traffic flow through the mesh

```
Inbound request:
  External → Ingress Gateway (Envoy) → Service → Pod [Envoy sidecar → app]

Service-to-service:
  Pod A [app → Envoy sidecar] → [Envoy sidecar → Pod B app]
  (Envoy on A encrypts, Envoy on B decrypts — mTLS transparent to app)
```

## What a mesh provides

| Capability | How |
|---|---|
| **mTLS** | Envoy handles cert exchange; auto-rotated; app sees plaintext |
| **Distributed tracing** | Envoy adds trace headers (B3/W3C); Jaeger/Tempo aggregates |
| **Traffic splitting** | VirtualService routes X% to v1, Y% to v2 (canary without redeployment) |
| **Circuit breaking** | Envoy tracks failure rate; trips circuit to prevent cascade |
| **Retries + timeouts** | Configured in DestinationRule, not in app code |
| **Observability** | Every request emits golden signals (no app instrumentation needed) |
| **Authorization policies** | Deny service A from calling service B at the mesh level |

## Istio core resources

```yaml
# VirtualService: define routing rules
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-svc
spec:
  hosts: [payment-svc]
  http:
    - match:
        - headers:
            canary: {exact: "true"}
      route:
        - destination: {host: payment-svc, subset: v2}
    - route:
        - destination: {host: payment-svc, subset: v1}
          weight: 90
        - destination: {host: payment-svc, subset: v2}
          weight: 10

# DestinationRule: define subsets + circuit breaking
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-svc
spec:
  host: payment-svc
  subsets:
    - name: v1
      labels: {version: v1}
    - name: v2
      labels: {version: v2}
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
    outlierDetection:        # circuit breaking
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

## When to use

**Use a service mesh when:**
- 10+ microservices and mTLS compliance is required (PCI DSS, SOC 2)
- You need sophisticated L7 traffic management (canary at request level, A/B by header)
- You want observability across all services without instrumenting each one
- You need centralized authorization policies (who can call what)

**Don't use when:**
- < 5 services — overhead not worth it
- Team has no Istio/Linkerd experience — steep learning curve
- Debugging mesh issues requires knowing both app and proxy behavior
- Adds ~25MB RAM per pod for the Envoy sidecar

## Tool comparison

| | Istio | Linkerd | Cilium |
|---|---|---|---|
| **Proxy** | Envoy | Linkerd-proxy (Rust) | eBPF (no sidecar) |
| **Features** | Most complete | Simpler, lighter | Network-level only |
| **Complexity** | High | Medium | Low–Medium |
| **Performance** | Good | Best (Rust proxy) | Best (kernel-level) |
| **Use case** | Enterprise, full features | Developer-friendly | K8s-native, no sidecar |

## Common interview angles

- "How does a service mesh differ from an API gateway?" — API gateway handles north-south (external → cluster) traffic; mesh handles east-west (service → service) traffic inside the cluster.
- "How do you debug a request in a mesh?" — Check the Envoy sidecar logs (`kubectl logs pod -c istio-proxy`); use Jaeger/Zipkin trace; check VirtualService/DestinationRule configs.
- "What's the performance overhead?" — Envoy adds ~0.5ms latency per hop; ~25MB RAM per sidecar. Usually acceptable for the security/observability gain.

## Sources
- [[devops/concepts/kubernetes-architecture]]
- [[devops/topics/security-devsecops]]
