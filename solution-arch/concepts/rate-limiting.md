# Rate Limiting

**Topic:** [[solution-arch/topics/security-architecture]], [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/idempotency]]

## What it is
Rate limiting controls how many requests a client can make in a given time window. Protects services from abuse, DDoS, and unintentional overload.

## Algorithms

### Token Bucket
```
Bucket capacity: 100 tokens
Refill rate:     10 tokens/second

  [■■■■■■■■■■] ← full bucket (100 tokens)

  Request arrives → consume 1 token
  Bucket refills at 10 tok/sec

  Burst allowed: up to 100 requests instantly
  Sustained rate: 10 req/sec
```
Best for: allowing occasional bursts while enforcing average rate. Used by AWS API Gateway.

### Leaky Bucket
```
Requests ──▶ [■■■■■■■■] queue ──▶ processed at fixed rate
             (bucket)           (e.g., 10 req/sec, drains constantly)

  If queue full: reject/drop new requests
  Output rate is always constant (no bursting)
```
Best for: smoothing out bursty traffic to downstream services.

### Fixed Window Counter
```
Window: 00:00–01:00
  Counter resets to 0 at each minute boundary
  Requests count up; reject when limit exceeded

Problem: boundary burst
  00:59: 100 requests (limit reached)
  01:00: 100 requests (new window — reset!)
  → 200 requests in 2 seconds at boundary
```

### Sliding Window Log
```
Keep timestamp log of each request.
  On new request: remove timestamps older than window
  Count remaining timestamps
  If count < limit: allow; add timestamp
  Else: reject

Accurate but memory intensive (stores all timestamps).
```

### Sliding Window Counter
```
Hybrid: estimate sliding window using fixed window data.

Current window count + (prev window count × overlap fraction)
  
  Window [00:00–01:00]: 80 requests
  Window [01:00–02:00]: 40 requests (so far, 30sec in)
  
  Estimated count = 40 + (80 × 0.5) = 80  (50% of prev window overlaps)
```
Best for: efficient approximation at scale. Used by Redis + Cloudflare.

## Distributed Rate Limiting

Single-instance rate limiting is trivial (in-memory counter). The challenge is across many instances:

```
Approach 1: Centralised store (Redis)
  Each service instance ──▶ Redis INCR + TTL
  + Accurate across instances
  - Redis becomes a hot dependency; adds latency (~1ms per request)

Approach 2: Sticky routing
  Client always hits same instance (via consistent hashing at LB)
  Each instance rate-limits locally
  + No shared state
  - Breaks if client rotates IPs or LB rebalances

Approach 3: Local approximation
  Each instance has local counter; periodically sync to central store
  + Low latency (mostly local)
  - Brief overcounting window between syncs
```

## Rate Limit Response

```
HTTP 429 Too Many Requests
  Retry-After: 30
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 0
  X-RateLimit-Reset: 1714100060
```

Client should honour `Retry-After` and back off with jitter.

## Scoping Rate Limits

| Scope | Granularity | Use case |
|-------|------------|---------|
| Per IP | IP address | Basic DDoS protection |
| Per API key | Client application | Partner API fairness |
| Per user | Authenticated user ID | Per-user quotas |
| Per endpoint | Route + method | Protect expensive operations |
| Global | Entire service | System protection |

Layered approach: IP-level at edge (CDN/WAF) + user-level at API Gateway + endpoint-level in service.

## Common interview angles
- "Design a distributed rate limiter." (Redis sliding window; token bucket; discuss tradeoffs)
- "What happens if Redis (rate limit store) goes down?" (Fail open: allow all traffic but risk abuse; Fail closed: reject all; Circuit breaker: switch to local approximate limiting)
- "Why is fixed window rate limiting problematic?" (Boundary burst doubles effective rate)
- "How does Stripe/Twilio implement rate limiting?" (Token bucket per API key in Redis)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
