# System Design: URL Shortener

**Difficulty:** Medium
**Concepts:** [[solution-arch/concepts/caching]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/load-balancing]]

---

## Step 1: Requirements

**Functional:**
- Shorten a long URL → short code (e.g., `sho.rt/abc123`)
- Redirect short URL to original
- Custom aliases (optional)
- Link expiry (optional)
- Analytics: click count, top referrers (optional)

**Non-Functional:**
- 100M URLs created per day → ~1,200 writes/sec
- 10B redirects per day → ~116,000 reads/sec (read-heavy, ~10,000:1)
- Low latency: redirect < 10ms P99
- High availability: 99.99%
- Storage: ~500 bytes per URL × 100M/day × 365 days × 5 years ≈ 90 TB

---

## Step 2: High-Level Design

```
                         Client
                           │
                  ┌────────▼────────┐
                  │  CDN / WAF      │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  Load Balancer  │
                  └────────┬────────┘
                     ┌─────┴──────┐
                     ▼            ▼
              ┌──────────┐  ┌──────────┐
              │ Write    │  │ Read     │
              │ Service  │  │ Service  │
              │ (create) │  │(redirect)│
              └──────┬───┘  └──┬───────┘
                     │         │
             ┌───────▼─────────▼──────┐
             │     Cache (Redis)      │
             │  shortCode → longURL   │
             └────────────┬───────────┘
                          │ cache miss
                 ┌────────▼─────────┐
                 │   URL Database   │
                 │  (PostgreSQL or  │
                 │   Cassandra)     │
                 └──────────────────┘
```

---

## Step 3: Short Code Generation

### Option A: Counter + Base62 encoding
```
Global counter (atomic): 1, 2, 3, 4, ...
Encode to base62: a–z, A–Z, 0–9 (62 chars)

counter = 1,000,000
base62(1,000,000) = "4c92"   (6 chars → 62^6 = 56B unique codes)

Problem: counter is a SPOF and bottleneck
Fix: each service node gets a range (1–1M, 1M–2M, ...)
     or: use Snowflake-style distributed ID
```

### Option B: MD5/SHA256 hash of long URL
```
hash = MD5(longURL) → 128-bit
Take first 8 chars → "3f7b2a9c"

Problem: hash collision if two URLs produce same prefix
Fix: check DB; if collision, append incrementing suffix
     MD5("https://example.com" + "1") → try again
```

### Option C: Random code (preferred for simplicity + security)
```
import secrets
code = secrets.token_urlsafe(6)  # 6 chars = ~47 bits of entropy
Check uniqueness in DB; retry if collision (probability ~0 at scale)
```

---

## Step 4: Redirect Flow (Critical Path — must be fast)

```
GET sho.rt/abc123
  │
  ▼
Read Service
  │
  ▼ check Redis
Redis hit (95% of requests):
  → Return 301/302 redirect (< 1ms)

Redis miss (5%):
  → Query DB for abc123
  → Populate Redis with TTL
  → Return redirect

HTTP 301 vs 302:
  301 Permanent: browser caches → no future requests (lower server load)
                 But: analytics miss subsequent visits
  302 Temporary: browser always requests → analytics capture all visits
                 Cost: more requests to server
  Decision: use 302 for analytics; 301 for pure performance
```

---

## Step 5: Database Schema

```sql
CREATE TABLE urls (
  short_code   VARCHAR(10)  PRIMARY KEY,
  long_url     TEXT         NOT NULL,
  user_id      VARCHAR(36),
  created_at   TIMESTAMPTZ  DEFAULT NOW(),
  expires_at   TIMESTAMPTZ,
  click_count  BIGINT       DEFAULT 0
);

CREATE INDEX idx_urls_user ON urls(user_id);
```

**Sharding:** shard on `short_code` hash (read and write access both use short code).

---

## Step 6: Analytics (Click Tracking)

```
Option A: Synchronous counter update
  UPDATE urls SET click_count = click_count + 1 WHERE short_code = 'abc123'
  → Slow (write on every redirect); lock contention at high QPS

Option B: Async via message queue
  Redirect Service ──click event──▶ Kafka ──▶ Analytics Consumer ──▶ ClickHouse
  → Zero impact on redirect latency
  → Near-real-time analytics (seconds delay)
  → ClickHouse handles billions of events efficiently

Option C: Approximate count in Redis
  INCR redirect:abc123 in Redis (atomic, < 1ms)
  Batch flush to DB every minute
  → Simple; eventually consistent
```

---

## Step 7: Trade-offs and Extensions

| Decision | Choice | Reason |
|----------|--------|--------|
| Cache eviction | LRU + TTL | Popular short codes stay cached; old ones expire |
| DB choice | Cassandra (at scale) | Write-optimised; no joins needed; sharding built-in |
| Redirect status | 302 | Preserve analytics |
| Code generation | Random + DB uniqueness check | Simple, collision-resistant |
| Expiry | TTL on Redis entry aligned with DB expiry | Expired links return 404 |

**Rate limiting:** 10 URLs per minute per IP to prevent abuse. Token bucket in Redis.

**Custom aliases:** check uniqueness at write time; if taken, return 409 Conflict.

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
