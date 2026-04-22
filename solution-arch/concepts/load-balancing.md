# Load Balancing

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/caching]]

## What it is
A load balancer distributes incoming requests across multiple backend instances to prevent any single instance from being overwhelmed, improve availability, and enable horizontal scaling.

## How it works

```
                      ┌──────────────────────────────┐
Clients               │        Load Balancer         │
                      │                              │
  User A ────────────▶│  Algorithm:                  │──▶ Server 1 (healthy ✓)
  User B ────────────▶│  • Round Robin               │──▶ Server 2 (healthy ✓)
  User C ────────────▶│  • Least Connections         │──▶ Server 3 (draining ⚠)
  User D ────────────▶│  • IP Hash                   │──✗ Server 4 (down  ✗)
                      │  • Weighted                  │
                      │                              │
                      │  Health checks every 10s     │
                      └──────────────────────────────┘
```

## L4 vs L7 Load Balancers

| | L4 (Transport Layer) | L7 (Application Layer) |
|--|---------------------|----------------------|
| Operates on | TCP/UDP packets | HTTP/HTTPS content |
| Can inspect | IP, port | Headers, URL, cookies, body |
| Performance | Faster (no parsing) | Slower but smarter |
| Features | Forwarding only | Path routing, SSL termination, sticky sessions, auth |
| Examples | AWS NLB, HAProxy (TCP) | AWS ALB, nginx, Envoy |

```
L4: Forward packets by IP:port
  Client:54321 ──TCP──▶ LB ──TCP──▶ Server:8080

L7: Route by content
  GET /api/users  ──HTTP──▶ LB ──▶ Users Service
  GET /api/orders ──HTTP──▶ LB ──▶ Orders Service
  POST /upload    ──HTTP──▶ LB ──▶ Upload Service
```

## Algorithms

| Algorithm | How | Best for |
|-----------|-----|---------|
| Round Robin | Each server in sequence | Uniform request size |
| Weighted Round Robin | More traffic to bigger servers | Heterogeneous capacity |
| Least Connections | Route to server with fewest active connections | Variable request duration |
| IP Hash | hash(client_IP) % n servers | Session affinity without cookies |
| Random | Random server | Simple; good at scale |
| Least Response Time | Route to fastest-responding server | Latency-sensitive |

## Health Checks

```
LB ──GET /health──▶ Server 1  →  200 OK      (healthy, receives traffic)
LB ──GET /health──▶ Server 2  →  timeout     (unhealthy, removed from pool)
LB ──GET /health──▶ Server 3  →  200 OK      (healthy, receives traffic)
```

Passive health checks: remove server after N consecutive failures.
Active health checks: probe endpoint on fixed interval.

## Sticky Sessions (Session Affinity)

Routes the same client to the same server — needed for stateful apps.

```
Without sticky:
  Request 1 → Server A (session stored here)
  Request 2 → Server B (session missing → logged out!)

With sticky (cookie-based):
  Request 1 → Server A → sets LB cookie: srv=A
  Request 2 → reads cookie → Server A (consistent)
```

**Problem:** server death loses all its sessions. **Fix:** externalise session to Redis.

## SSL Termination

```
Client ══TLS══▶ Load Balancer ──HTTP──▶ Backend Servers
                (decrypts here)          (plain text, internal network)

Benefits:
  + Backends don't need TLS certs or processing overhead
  + LB can inspect/modify HTTP headers

Risk: internal traffic unencrypted (mitigate with service mesh mTLS)
```

## Common interview angles
- "What's the difference between L4 and L7 load balancers?"
- "How do you handle session state when load balancing stateful services?"
- "What happens when a load balancer itself becomes a SPOF?" (Use anycast, multiple LBs, or DNS-based LB)
- "How does a load balancer know a backend is unhealthy?"

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
