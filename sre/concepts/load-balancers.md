---
tags:
  - sre
  - networking
  - kubernetes
  - interview-prep
---

# Concept: Load Balancers

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/networking-fundamentals]], [[devops/concepts/kubernetes-architecture]], [[devops/patterns/zero-downtime-deployment]]

---

## What a Load Balancer Does

A load balancer sits in front of a pool of servers and distributes incoming requests across them. It solves three problems simultaneously:

```
Without LB:                        With LB:
                                              ┌──▶ Server 1  (healthy)
Client ──────────▶ Server 1       Client ──▶ LB ──▶ Server 2  (healthy)
                  (single SPOF)              └──▶ Server 3  (unhealthy ✗)
                                                    (removed from pool)
```

1. **Horizontal scale** — spread load across N servers
2. **High availability** — route around failed/unhealthy servers
3. **Operational flexibility** — add/remove servers without client changes

---

## L4 vs L7: The Fundamental Split

The most important LB concept for interviews. Everything else follows from this.

```
OSI Stack                    L4 Load Balancer        L7 Load Balancer
─────────────────────────    ────────────────────    ────────────────────
L7  Application (HTTP)                               ✓ reads HTTP headers,
L6  Presentation (TLS)                                 path, cookies, body
L5  Session                                          ✓ can terminate TLS
L4  Transport (TCP/UDP)      ✓ routes on IP:port     ✓ also does this
L3  Network (IP)             ✓ reads src/dst IP
L2  Data Link
L1  Physical
```

### Network Load Balancer (L4)

Routes based on IP address and TCP/UDP port only. Does **not** read HTTP content.

```
Client TCP connection ──▶ NLB ──▶ Backend (TCP forwarded or new connection)

NLB sees:  src_ip=1.2.3.4, dst_ip=10.0.0.1, dst_port=443
NLB does:  pick a backend by IP hash / round-robin
NLB does NOT see: HTTP method, URL path, Host header, cookies
```

**Modes:**
- **TCP passthrough (DSR):** NLB rewrites dst IP to backend; backend responds directly to client. NLB never sees the response. Ultra-low latency but backend must handle TLS.
- **TCP proxy:** NLB terminates the client TCP connection, opens a new TCP connection to the backend. Allows connection pooling but adds 1 RTT.

**AWS NLB characteristics:**
- Static IP per AZ (predictable for firewall rules)
- Preserves source IP (client IP visible to backend)
- Ultra-low latency (~100μs overhead)
- TLS passthrough or TLS termination at NLB
- No HTTP-level features

**Use L4 when:**
- Non-HTTP traffic (databases, SMTP, custom TCP protocols)
- You need static IPs for firewall whitelisting
- Ultra-low latency is critical
- You need to preserve the client IP at the transport layer

### Application Load Balancer (L7)

Reads HTTP/HTTPS content. Can make routing decisions on any HTTP attribute.

```
Client HTTPS request ──▶ ALB (terminates TLS, reads HTTP) ──▶ Backend

ALB sees:
  method: POST
  path: /api/v2/payments
  Host: api.example.com
  Cookie: session=abc123
  User-Agent: Mozilla/5.0

ALB can route based on any of the above.
```

**Routing capabilities:**

```
Rule                              Route to
──────────────────────────────────────────────────────────────
path starts with /api/            → API backend target group
path starts with /static/         → S3 / CDN origin
Host header = app.example.com     → App servers
Host header = admin.example.com   → Admin servers (different TG)
Header X-Version: v2              → Canary backend (10% traffic)
Cookie: beta_user=true            → New version backend
Method = OPTIONS                  → Return 200 directly (CORS preflight)
```

**AWS ALB characteristics:**
- Path-based and host-based routing
- TLS termination (offloads crypto from backends)
- HTTP/2 and WebSocket support
- WAF integration (block SQLi, XSS, bad bots)
- Sticky sessions (via cookie `AWSALB`)
- Target types: instances, IP addresses, Lambda functions, containers

**Use L7 when:**
- HTTP/HTTPS traffic (the common case for web apps)
- You need path-based routing (microservices behind one LB)
- You need canary/A/B routing by header or cookie
- TLS termination at the LB layer
- WAF, auth gateway, request inspection needed

### L4 vs L7 comparison

| | L4 (NLB) | L7 (ALB) |
|---|---|---|
| **Routing basis** | IP + port | IP, port, HTTP headers, path, method, cookies |
| **TLS** | Passthrough or terminate | Terminate (can also re-encrypt to backend) |
| **Protocols** | Any TCP/UDP | HTTP, HTTPS, HTTP/2, WebSocket, gRPC |
| **Performance** | ~100μs overhead | ~1–5ms overhead (HTTP parsing) |
| **Source IP** | Preserved (passthrough) or X-Forwarded-For (proxy) | X-Forwarded-For header |
| **Sticky sessions** | IP hash | Cookie-based |
| **Cost (AWS)** | Lower | Higher (LCU pricing) |
| **Health check** | TCP connect | HTTP endpoint (checks app, not just port) |

---

## Load Balancing Algorithms

### Round Robin

```
Request 1  ──▶ Server A
Request 2  ──▶ Server B
Request 3  ──▶ Server C
Request 4  ──▶ Server A  (cycle repeats)
```

**Best for:** homogeneous servers with similar request costs. Fails when requests have wildly different processing times — one slow request leaves a server "busy" while round-robin keeps sending it more.

### Weighted Round Robin

```
Server A: weight 3  ──▶ gets 3 out of every 5 requests
Server B: weight 2  ──▶ gets 2 out of every 5 requests

Use when: servers have different CPU/RAM capacity
```

### Least Connections

```
Current state:
  Server A: 10 active connections
  Server B:  3 active connections  ← next request goes here
  Server C:  7 active connections
```

**Best for:** long-lived connections (WebSocket, database, streaming) where connection count is a real proxy for server load. Worse than round-robin for very short requests (adds overhead to track connection counts).

### Least Response Time

Routes to the backend with the lowest combination of: active connections + average response time. Most accurate for heterogeneous workloads.

```
Server A: 5 connections, avg 200ms  → score ≈ 1000
Server B: 2 connections, avg 50ms   → score ≈ 100  ← pick this
Server C: 8 connections, avg 10ms   → score ≈ 80   ← or this
```

### IP Hash (Sticky by IP)

```
hash(client_ip) % num_backends → always routes same client to same server
```

**Use when:** server-side session state (not stored in shared cache). Problem: if a server goes down, all its sessions are lost. Prefer shared session storage (Redis) over IP hash.

### Consistent Hashing

```
Virtual ring (0 to 2^32):
  Server A at position 10
  Server B at position 40
  Server C at position 70

Client with hash 25 → clockwise to Server B
Client with hash 55 → clockwise to Server C
Client with hash 85 → clockwise to Server A (wraps around)

Adding Server D at position 55:
  Only clients between 40 and 55 move from B to D
  All other assignments unchanged
```

**Critical property:** when a server is added or removed, only `1/N` of keys need to remapped (vs. `(N-1)/N` for modulo hashing). Used in: distributed caches (Cassandra, Redis Cluster), CDN routing, microservice sharding. Virtual nodes smooth out uneven distribution.

### Random with Two Choices (Power of Two Choices)

```
1. Pick two random backends
2. Of the two, choose the one with fewer active connections
```

Achieves near-optimal load balance in O(1) with no central state. Used in Nginx and Envoy's `RANDOM` policy.

---

## Health Checks

Load balancers continuously check backend health. Unhealthy backends are removed from the pool; they're re-added when healthy again.

```
Passive (circuit breaker):
  LB observes real traffic — if 5xx rate > 50% → mark unhealthy
  No extra traffic, but detection is slower

Active:
  LB sends probe requests to each backend every N seconds
  TCP probe: can I open a connection?   (checks: port is listening)
  HTTP probe: GET /health → 200?        (checks: app is responding)
  HTTPS probe: also validates TLS cert  (checks: cert is valid)
```

**nginx upstream health check:**
```nginx
upstream backend {
    server app1:8080;
    server app2:8080;
    server app3:8080;
    # passive: mark down after 3 fails, try again after 30s
    max_fails=3 fail_timeout=30s;
}
```

**Health check endpoint best practices:**
```python
@app.route("/health/live")    # liveness: is the process alive?
def liveness():
    return "ok", 200          # never check dependencies here

@app.route("/health/ready")   # readiness: can it serve traffic?
def readiness():
    db.ping()                 # check DB connectivity
    cache.ping()              # check Redis
    return "ok", 200          # or 503 if dependency is down
```

---

## Database / Data Layer Load Balancing

Databases are stateful — you can't naively round-robin all queries. The data layer has specialized patterns.

### Read/Write Split

```
Application
    │
    ├──── writes ────▶ Primary (1 node, read-write)
    │                       │
    └──── reads  ────▶ Read Replica Pool  ──▶ Replica 1
                       (LB across replicas) ──▶ Replica 2
                                            ──▶ Replica 3

Replication lag is the key risk:
  A write to primary may not be visible on replicas for 10ms–1s.
  "Read your own writes" consistency: route reads immediately after
  a write back to primary for that session.
```

### Connection Pooling (PgBouncer, ProxySQL)

```
Without pool:
  20 app pods × 20 connections each = 400 DB connections
  PostgreSQL default max: 100–200 → FATAL: too many connections

With PgBouncer (transaction pooling):
  20 app pods × 20 "connections" → PgBouncer → 20 real DB connections
  1 real connection serves many app transactions sequentially
  Reduces DB connection overhead by 10–20×
```

**PgBouncer modes:**
| Mode | Connection reuse | Limitation |
|---|---|---|
| Session | 1 real conn per app session | No multiplexing |
| Transaction | Reuse after each transaction | Can't use `SET`/prepared statements across txns |
| Statement | Reuse after each statement | Very aggressive; rarely used |

**ProxySQL (MySQL):** adds query routing on top of pooling — route `SELECT` to replicas, `INSERT/UPDATE` to primary, based on SQL parsing.

### Horizontal Sharding

```
Shard key: user_id
  user_id 0–999,999    → Shard 0  (DB cluster A)
  user_id 1M–1,999,999 → Shard 1  (DB cluster B)
  user_id 2M–2,999,999 → Shard 2  (DB cluster C)

Routing layer (application or proxy like Vitess):
  hash(user_id) % num_shards → shard index → DB cluster
```

**Challenges:**
- Cross-shard queries (joins across shards) require scatter-gather
- Re-sharding is painful (consistent hashing helps)
- Hot shards (celebrity user problem) — use virtual shards or random salt

### Distributed SQL / NewSQL (CockroachDB, Spanner, Vitess)

Built-in sharding + replication + LB at the database layer. The application connects to a stateless SQL proxy that routes internally. For SRE interviews, know this exists and roughly how it works.

---

## Kubernetes Load Balancing

### Service types

```
ClusterIP (default):
  Virtual IP inside the cluster only
  Accessible only from within the cluster
  Backed by iptables/IPVS rules on every node

NodePort:
  Opens a port (30000–32767) on every node
  External: NodeIP:NodePort → ClusterIP → Pod
  Rarely used in production (non-standard ports, all nodes exposed)

LoadBalancer:
  Provisions a cloud LB (AWS NLB/ALB, GCP TCP LB)
  External IP → cloud LB → NodePort → ClusterIP → Pod
  One LB per Service = expensive at scale

Ingress:
  One L7 LB for all Services
  Route by host + path → different Services
  Cost-effective: 1 LB, N services
```

### Ingress + Ingress Controller

```
External request:
  https://api.example.com/payments  ──▶
  Cloud LB (one, shared)  ──▶
  Ingress Controller (nginx/envoy pod)  ──▶  reads Ingress rules
  Ingress rule: host=api.example.com, path=/payments → svc/payment-svc
  payment-svc ClusterIP  ──▶  kube-proxy iptables  ──▶  Pod

Ingress resource (declarative routing):
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls-cert     # cert-manager manages this
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /payments
            pathType: Prefix
            backend:
              service:
                name: payment-svc
                port: {number: 80}
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: user-svc
                port: {number: 80}
```

### kube-proxy: how ClusterIP routing actually works

```
Every node runs kube-proxy (or Cilium eBPF, which replaces it)

kube-proxy watches Endpoints (pod IPs for each Service)
When a pod sends traffic to ClusterIP:

iptables mode:
  Packet hits PREROUTING chain
  iptables rule: dst=ClusterIP:port → DNAT to one of [pod1:port, pod2:port, pod3:port]
  Selection: probability rules (1/3, then 1/2, then 1) = uniform random

IPVS mode (better at scale):
  Uses Linux IPVS (kernel LB) instead of iptables chains
  O(1) lookup vs O(n) iptables chain traversal
  Supports LB algorithms: round-robin, least-conn, destination-hash
  Recommended for clusters with > 1000 Services

Cilium eBPF mode:
  Replaces kube-proxy entirely
  LB logic in eBPF programs in the kernel
  Fastest path; also adds network policy, observability
```

### Service topology / traffic policies

```yaml
spec:
  # internalTrafficPolicy: Cluster (default) | Local
  # Local = only route to pods on the SAME node (avoids cross-node hops)
  # Use for: DaemonSet services, node-local caches
  internalTrafficPolicy: Local

  # externalTrafficPolicy: Cluster (default) | Local
  # Local = preserve client IP, avoid double-hop
  # Trade-off: uneven load (pods not on all nodes get no traffic)
  externalTrafficPolicy: Local
```

### Gateway API (next-gen Ingress)

Kubernetes 1.24+ — more expressive than Ingress, supports L4 + L7, multi-tenancy:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs: [{name: prod-gateway}]
  hostnames: [api.example.com]
  rules:
    - matches:
        - path: {type: PathPrefix, value: /payments}
          headers:
            - {name: X-Version, value: v2}
      backendRefs:
        - name: payment-svc-v2
          port: 80
          weight: 10
        - name: payment-svc-v1
          port: 80
          weight: 90
```

---

## Common Interview Questions

**Q: "When would you use an NLB vs an ALB?"**
NLB for non-HTTP protocols (database, SMTP, custom TCP), static IPs, ultra-low latency, or TLS passthrough to backends. ALB for HTTP/HTTPS routing, microservices with path-based routing, TLS termination, WAF integration, canary deployments.

**Q: "Explain consistent hashing and why it matters."**
With modulo hashing (`hash(key) % N`), adding 1 server remaps `(N-1)/N` of all keys — devastating for a cache. Consistent hashing maps keys to a ring; adding a server remaps only `1/N` of keys. Critical for distributed caches and CDN routing.

**Q: "How does Kubernetes route traffic to pods?"**
kube-proxy programs iptables (or IPVS) rules on every node. When traffic hits a ClusterIP, DNAT rewrites the destination to a specific pod IP selected randomly from the Endpoints list. The pod IP is then routed across the overlay network (VXLAN/BGP) to the correct node.

**Q: "How do you load balance a database?"**
Read/write split (primary for writes, replicas for reads). Connection pooling via PgBouncer to reduce connection count. Sharding for horizontal scale. Watch for: replication lag on reads, hot shards, cross-shard queries.

**Q: "What's the problem with sticky sessions / IP hash?"**
If a backend goes down, all sessions pinned to it are lost. Better: store session state externally (Redis) and make backends stateless — then any backend can serve any request, and true round-robin/least-conn works.

**Q: "What is the difference between least connections and round robin?"**
Round-robin ignores current load — it can send a new request to a server already handling 100 slow requests. Least connections tracks active connections and routes to the least-loaded server. Better for variable-cost workloads (e.g., some API calls are 10ms, others are 5 seconds).

## Sources
- [[sre/concepts/networking-fundamentals]]
- [[devops/concepts/kubernetes-architecture]]
- [[devops/patterns/zero-downtime-deployment]]
- [[devops/topics/containers-orchestration]]
