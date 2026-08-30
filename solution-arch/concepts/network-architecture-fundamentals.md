# Network Architecture Fundamentals (SA View)

**Topic:** [[solution-arch/topics/scalability-and-reliability]], [[solution-arch/topics/security-architecture]]
**Related:** [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/caching]]
**Deep-dive (protocol/CLI level):** [[sre/concepts/networking-fundamentals]], [[sre/concepts/networking-troubleshooting]]

> **Scope note:** [[sre/concepts/networking-fundamentals]] already covers the TCP three-way handshake, TLS 1.2/1.3 handshake mechanics, certificate chains, forward secrecy, and proxy/Zero-Trust internals in full depth — read that page for "walk me through what happens when you hit a URL" and handshake-level questions. This page covers the **design-time decisions** an SA owns when architecting a system's network topology: CDN placement, DNS architecture, protocol version choice, and network segmentation. The two pages are complementary, not overlapping.

## What it is

Network architecture, from an SA's perspective, is the set of decisions about how traffic physically and logically flows from a user to a system's compute — which points sit in front of the origin (CDN, DNS, load balancer), how the network is segmented for blast-radius containment (VPC/subnet design), and which transport protocol version to standardize on. These decisions affect latency, availability, cost, and security posture before a single line of application code runs.

## How it works

### DNS Architecture

```
User's browser
     │
     ▼
Recursive resolver (ISP or 1.1.1.1 / 8.8.8.8)
     │  (not authoritative — walks the hierarchy on cache miss)
     ▼
Root servers  →  TLD servers (.com)  →  Authoritative nameserver
                                        (Route 53 / Cloudflare DNS /
                                         Azure DNS for your domain)
     │
     ▼
Returns IP(s), cached at recursive resolver for the record's TTL
```

```
GSLB (Global Server Load Balancing) — DNS-based traffic steering:

  api.example.com
       │
       ▼
  Authoritative DNS returns a DIFFERENT IP depending on:
    - GeoDNS: nearest region to the resolver's location
    - Latency-based: region with lowest measured latency
    - Weighted: percentage split for canary/blue-green at the DNS layer
    - Failover: healthy region's IP, or standby if primary is down

  This is how multi-region active-active systems route users to the
  nearest healthy region WITHOUT any load balancer in the path yet —
  the routing decision happens at DNS resolution time.
```

**Architectural gotcha: DNS TTL trade-off.** A low TTL (30-60s) makes failover fast but multiplies query volume and load on authoritative DNS; a high TTL (hours) is cheap but means a failover takes as long as the TTL to propagate to clients holding a cached record. Set TTL based on how fast you actually need failover to take effect, not a default.

### CDN Architecture

```
User ──▶ Nearest CDN edge PoP (Cloudflare/Akamai/CloudFront — often
          <50ms away, hundreds of global PoPs)
              │
        ┌─────┴─────┐
     Cache HIT     Cache MISS
        │             │
        ▼             ▼
  Serve from      Origin fetch (to your actual servers/load balancer),
  edge cache      cache the response at the edge for the next request
  (fast, cheap,   from anyone near this PoP
  no origin load)
```

```
What belongs behind a CDN, architecturally:
  ✅ Static assets (JS/CSS/images) — near-100% cacheable
  ✅ API responses that are identical across users for a TTL window
     (e.g. product catalog, public content)
  ✅ TLS termination at the edge — cuts handshake RTT for users far
     from origin (edge is close; origin connection is kept warm/
     reused between edge and origin)

What does NOT belong behind a CDN cache (but may still route
through it for DDoS/WAF benefits):
  ❌ Per-user personalized responses without cache-key discipline
     (a naive CDN config can leak User A's personalized response to
     User B if the cache key doesn't include the auth/session context —
     a real, serious security bug class, not just a performance miss)
  ❌ Real-time/streaming responses (see time-to-first-token concerns
     in [[solution-arch/topics/llm-application-architecture]])
```

### HTTP Protocol Version — An Architecture Decision, Not Just a Config Flag

```
HTTP/1.1
  - One request per TCP connection at a time (head-of-line blocking
    at the app layer); browsers open 6-8 parallel connections per
    host to compensate
  - Still common for simple backend-to-backend calls where the
    overhead doesn't matter

HTTP/2
  - Multiplexes many requests over ONE TCP connection (binary
    framing) — eliminates app-layer head-of-line blocking, reduces
    connection overhead (fewer TLS handshakes)
  - Problem: head-of-line blocking still exists at the TCP layer —
    one lost packet stalls ALL multiplexed streams on that connection,
    because TCP guarantees in-order delivery of the whole stream

HTTP/3 (built on QUIC, over UDP)
  - QUIC implements its own reliability/ordering PER STREAM, not
    per-connection — a lost packet only stalls the one stream it
    belongs to, not all of them
  - Faster connection establishment: QUIC combines the transport
    and TLS 1.3 handshake into fewer round trips than TCP+TLS
    separately (full handshake message flow, plus the HTTP/2
    connection preface and ALPN mechanics, in
    [[sre/concepts/networking-fundamentals]])
  - Connection migration: a QUIC connection survives a client's
    network change (WiFi → cellular) without re-establishing —
    valuable for mobile-heavy traffic

Architecture decision: HTTP/3 is worth adopting at the CDN/edge
layer first (biggest win for users on lossy/mobile networks talking
to your edge) even before backend services adopt it internally,
where HTTP/2 or even gRPC-over-HTTP/2 is usually still the pragmatic
default.
```

### Network Segmentation (VPC Design)

```
┌──────────────────────────────────────────────────────────────┐
│                          VPC (10.0.0.0/16)                       │
├──────────────────────────────────────────────────────────────┤
│  Public subnet (10.0.1.0/24)                                      │
│    - Load balancer, NAT gateway, bastion host                      │
│    - Has a route to an Internet Gateway                             │
├──────────────────────────────────────────────────────────────┤
│  Private subnet — app tier (10.0.10.0/24)                          │
│    - App servers / Kubernetes nodes                                 │
│    - No direct internet route; outbound via NAT gateway in the       │
│      public subnet (can reach out, can't be reached from outside)     │
├──────────────────────────────────────────────────────────────┤
│  Private subnet — data tier (10.0.20.0/24)                          │
│    - Databases, caches                                                │
│    - Security group allows inbound ONLY from app-tier subnet on        │
│      the specific DB port — nothing else, not even other internal      │
│      services                                                          │
└──────────────────────────────────────────────────────────────┘

Security groups (stateful, instance-level) vs NACLs (stateless,
subnet-level): security groups are the primary tool for "which
service can talk to which" (allow rules only, implicit deny);
NACLs are a coarser, defense-in-depth layer at the subnet boundary.
```

```
Multi-VPC / multi-account patterns:

VPC Peering: point-to-point connection between two VPCs — doesn't
  scale well past a handful of VPCs (no transitive routing; N VPCs
  need up to N×(N-1)/2 peering connections)

Transit Gateway / Hub-and-spoke: a central hub VPC that all other
  VPCs connect to — scales to hundreds of VPCs, centralizes routing
  policy and network security team ownership, same "avoid the N×M
  problem" principle as [[solution-arch/concepts/model-context-protocol-mcp]]
  applies to networking here

Private connectivity to cloud services (VPC endpoints / Private
  Link): keep traffic to managed services (S3, the LLM provider's
  API, a SaaS vendor) off the public internet entirely — relevant
  to the OpenAI-direct-vs-Azure-OpenAI data-residency discussion in
  [[solution-arch/topics/openai-platform-architecture]]
```

## Complexity

Not algorithmic. The relevant cost trade-offs are: DNS TTL (failover speed vs query volume/cache efficiency), CDN cache hit rate (origin load/cost vs staleness risk), and network segmentation depth (blast-radius containment vs operational complexity of managing many subnets/security groups).

## When to use

```
Multi-region GSLB + low DNS TTL: when the SLA requires fast
  regional failover (seconds-to-minutes), and query volume/cost at
  low TTL is acceptable.

CDN in front of an API (not just static assets): when a meaningful
  fraction of responses are cacheable and shared across users —
  verify cache-key design excludes per-user data correctly first.

HTTP/3 adoption: prioritize the edge/CDN layer for user-facing
  traffic over mobile/lossy networks before internal service-to-
  service calls, where the win is smaller relative to the migration
  effort.

Multi-VPC with Transit Gateway over flat peering: once you have
  more than ~4-5 VPCs needing to interconnect, or when a platform/
  network team needs centralized routing policy ownership.
```

## Common interview angles

```
Q: "How would you architect for sub-second failover to a secondary
    region?"
A: DNS-based failover alone is too slow for sub-second (bounded by
    TTL + client cache behavior, seconds to minutes even at low
    TTL). Sub-second requires anycast IP addressing (the SAME IP
    announced from multiple regions via BGP, with the network
    routing to the nearest healthy one) rather than relying on DNS
    resolution to change — this is how Cloudflare/major CDNs achieve
    near-instant failover.

Q: "A CDN is serving one user's personalized data to another user.
    What went wrong?"
A: The cache key doesn't include enough context (missing the
    session/auth header or a Vary header) to distinguish requests
    that must NOT share a cached response — a cache-key design bug,
    not a CDN outage. Fix: explicitly scope the cache key, or mark
    the response Cache-Control: private/no-store for anything
    containing per-user data.

Q: "Why would you put HTTP/3 at the edge but keep HTTP/2 internally?"
A: HTTP/3's biggest win (per-stream loss recovery, faster handshake,
    connection migration) matters most on the lossy, high-latency,
    mobile-heavy path between end users and your edge. Internal
    service-to-service traffic on a reliable, low-latency data-center
    network doesn't suffer the same head-of-line-blocking pain, so
    the migration cost of moving every internal service to HTTP/3
    isn't justified by the win there yet.

Q: "Design the network segmentation for a system holding regulated
    (PCI/PHI) data."
A: Isolate the regulated-data tier in its own subnet/VPC with
    security groups allowing inbound ONLY from the specific
    services that need it, no direct internet route at all (not even
    via NAT), and route ALL access through a narrow, audited gateway
    service — the same "least privilege at the network layer"
    principle as tool permission scoping in
    [[solution-arch/concepts/ai-guardrails-and-safety]], applied to
    infrastructure instead of an agent's tool access.
```

## Examples

```
A global e-commerce platform's edge architecture:
  User ──▶ Anycast IP ──▶ Nearest CDN PoP (HTTP/3, TLS 1.3, static
            + cacheable API responses served from edge)
                │ (cache miss)
                ▼
          GSLB-routed to nearest healthy region
                │
                ▼
          Regional load balancer (L7, HTTP/2 to backends)
                │
                ▼
          App tier (private subnet) ──▶ Data tier (private subnet,
                                          most restrictive security
                                          group)
```

## Sources
- [[sre/concepts/networking-fundamentals]]
- [[solution-arch/concepts/load-balancing]]
