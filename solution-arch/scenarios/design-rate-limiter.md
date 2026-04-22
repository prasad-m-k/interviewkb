# System Design: Distributed Rate Limiter

**Difficulty:** Medium
**Concepts:** [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/caching]], [[solution-arch/concepts/idempotency]]

---

## Step 1: Requirements

**Functional:**
- Limit requests per client (API key or IP) per time window
- Return HTTP 429 with `Retry-After` header when exceeded
- Support multiple rule types: per-second, per-minute, per-day
- Rules configurable per endpoint and per client tier (free/paid)

**Non-Functional:**
- Decision latency < 2ms P99 (must be fast enough to be inline)
- Works across N API server instances (distributed)
- Accurate (prefer sliding window over fixed window)
- Graceful: if rate limiter store fails, fail open (allow traffic) or fail closed (deny all)

---

## Step 2: Algorithm Selection

```
Token Bucket (chosen):
  Capacity: 100 tokens
  Refill: 10 tokens/sec
  
  On request:
    tokens = get_tokens(client_id)
    if tokens >= 1:
      decrement_tokens(client_id, 1)
      allow
    else:
      deny (429)
  
  Supports bursting; intuitive; widely used (AWS API Gateway)
```

Alternative: Sliding Window Counter (more accurate at boundaries).

---

## Step 3: Architecture

```
                     Client Request
                          │
                 ┌────────▼────────┐
                 │   API Gateway   │  (or middleware in each service)
                 │                 │
                 │  RateLimiter    │◀─────────── Rules Config
                 │  Middleware     │             (endpoint, tier, limit)
                 └────────┬────────┘
                          │
          ┌───────────────┼──────────────┐
          ▼               ▼              ▼
    API Server 1    API Server 2   API Server 3
          │               │              │
          └───────────────┼──────────────┘
                          │
                 ┌────────▼────────┐
                 │  Redis Cluster  │
                 │                 │
                 │ key: rate:{api_key}:{window}
                 │ val: token_count
                 └─────────────────┘
```

---

## Step 4: Redis Implementation (Sliding Window Counter)

```lua
-- Atomic Lua script (runs in single Redis transaction)
local key = KEYS[1]              -- e.g., "rate:apikey123:minute"
local limit = tonumber(ARGV[1])  -- e.g., 100
local window = tonumber(ARGV[2]) -- e.g., 60 (seconds)
local now = tonumber(ARGV[3])    -- current unix timestamp (ms)

-- Remove entries outside the window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

-- Count remaining entries
local count = redis.call('ZCARD', key)

if count < limit then
  -- Add current request timestamp
  redis.call('ZADD', key, now, now)
  redis.call('EXPIRE', key, window)
  return {1, limit - count - 1}  -- {allowed, remaining}
else
  return {0, 0}                  -- {denied, remaining}
end
```

**Why Lua?** Atomicity — read-increment-check must be a single operation to prevent race conditions.

---

## Step 5: Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1714100060

HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1714100060
```

---

## Step 6: Multi-Tier Rules

```
Rule hierarchy (most specific wins):
  1. Client-specific override: api_key:abc123 → 1000 req/min
  2. Tier-based: tier:enterprise → 500 req/min
  3. Endpoint-specific: POST /videos/upload → 10 req/hour
  4. Global default: 100 req/min

Implementation:
  Rules stored in config store (Redis or DB)
  Rate limiter middleware fetches rules at startup, caches in-process
  TTL: refresh every 60 seconds
```

---

## Step 7: Failure Modes

```
Scenario: Redis is unavailable

Option A: Fail Open (allow all traffic)
  Pro: Service stays available; clients unaffected
  Con: Abuse/DDoS proceeds unchecked during Redis outage

Option B: Fail Closed (deny all traffic)
  Pro: Abuse prevented
  Con: All legitimate traffic denied; bad UX

Option C: Local rate limiting (degraded mode)
  Each server instance uses in-memory counter
  Pro: Partial protection; service available
  Con: Inaccurate (N instances = N× the limit effectively)

Recommendation: fail open + alert on-call + circuit breaker on Redis
  Rationale: brief window of unthrottled traffic < service unavailability
```

---

## Step 8: Trade-offs

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Algorithm | Sliding window counter | More accurate than fixed window at boundaries |
| Storage | Redis Cluster | Sub-millisecond latency; atomic Lua scripts |
| Key design | `rate:{identifier}:{window}` | Flexible scope; easy TTL management |
| Scope | Per API key (auth'd) + Per IP (unauth'd) | API key = reliable identity; IP = fallback |
| Redis failure | Fail open | Availability > rate enforcement during brief outage |
| Latency overhead | ~1ms (Redis call) | Acceptable for API gateway inline check |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
