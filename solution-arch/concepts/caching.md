---
uid: 3954fa98-b063-4319-8d66-164eca6112e1
---

# Caching

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/idempotency]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/patterns/hot-path-first-design]]

## What it is
Caching stores the result of expensive operations in a fast-access layer so that subsequent requests can be served without recomputing or re-fetching from the origin.

## Cache Hit vs Miss

```
                ┌──────────────┐
                │    Cache     │
Request ───────▶│              │
                │  Hit?        │
                │  ┌─────────┐ │──── YES ───▶ Return cached value
                │  │  Key    │ │
                │  │  found  │ │
                │  └─────────┘ │──── NO  ───▶ Fetch from origin
                └──────────────┘               │
                       ▲                       │
                       └─── Populate cache ────┘
```

**Cache Hit Rate** = hits / (hits + misses). Target > 95% for effective caching.

## Caching Strategies

### Cache-Aside (Lazy Loading)
Application code manages the cache. Most common pattern.
```
Read:
  1. Check cache
  2. Cache miss → query DB
  3. Populate cache with result
  4. Return result

Write:
  1. Write to DB
  2. Invalidate cache key (or update)
```
Pros: Only requested data is cached. Cache failure is non-fatal.
Cons: First request always slow (cold start). Data can be stale.

### Write-Through
Write to cache and DB together, synchronously.
```
Write: App ──▶ Cache ──▶ DB (synchronous)
Read:  App ──▶ Cache (always warm)
```
Pros: Cache always consistent with DB.
Cons: Write latency increases; cache may hold data never read.

### Write-Behind (Write-Back)
Write to cache immediately; persist to DB asynchronously.
```
Write: App ──▶ Cache (returns immediately)
               Cache ──async──▶ DB
```
Pros: Lowest write latency.
Cons: Risk of data loss if cache crashes before DB write.

### Read-Through
Cache sits in front of DB; cache fetches from DB on miss automatically.
```
Read: App ──▶ Cache
              Cache (miss) ──▶ DB ──▶ Cache ──▶ App
```
Used by: Redis (with RedisGears), ORM-level caches.

## Eviction Policies

| Policy | Evicts | Best for |
|--------|--------|---------|
| **LRU** (Least Recently Used) | Oldest accessed item | General purpose; temporal locality |
| **LFU** (Least Frequently Used) | Least accessed item | Popularity-based access |
| **FIFO** | First inserted | Simple queue-like data |
| **TTL** (Time-To-Live) | Expired items | Data with natural staleness (weather, prices) |
| **Random** | Random item | Simple; avoids scan overhead |

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

Strategies:
1. **TTL expiry** — cache entries expire automatically after N seconds. Simple; allows brief staleness.
2. **Event-driven invalidation** — DB change event triggers cache delete. Complex but accurate.
3. **Write-through** — invalidate on every write. Correct but higher write cost.
4. **Versioned keys** — `product:123:v2` — old keys naturally become unreachable.

## Cache Layers

```
Browser cache (HTTP Cache-Control headers)
      │
      ▼
CDN (edge cache — geographically distributed)
      │
      ▼
API Gateway cache
      │
      ▼
Application-level cache (in-process, e.g. Guava, Caffeine)
      │
      ▼
Distributed cache (Redis, Memcached)
      │
      ▼
Database query cache
      │
      ▼
Database (disk)
```

**Rule:** cache as close to the user as possible for best latency reduction.

## CDN Caching

```
User (Tokyo) ──▶ CDN Edge (Tokyo) ──HIT──▶ Serve asset
                                   ──MISS──▶ Origin (US) ──▶ Cache + Serve
```

Cache-Control headers control CDN behaviour:
```
Cache-Control: public, max-age=86400          # cache 1 day, anywhere
Cache-Control: private, max-age=0             # browser only, don't cache in CDN
Cache-Control: no-store                       # never cache (sensitive data)
Cache-Control: stale-while-revalidate=300     # serve stale for 5min while fetching fresh
```

## Common interview angles
- "How do you handle cache stampede?" (many simultaneous misses hit DB — use mutex lock, probabilistic early expiry, or warming)
- "You have a hot key in Redis — what do you do?" (local in-process cache tier, key sharding with suffix, replicate reads)
- "Difference between write-through and write-behind?"
- "How do you invalidate a cache in a distributed system?"
- "What are the risks of in-process caching vs. distributed caching?"

## Related: Scaling a Cache Beyond One Node

Everything above assumes a single cache instance (or a simple pair). Once the working set or request rate outgrows that, sharding, replication, and cross-region consistency become the relevant problems — see [[solution-arch/concepts/distributed-caching]] for consistent hashing, gutter pools, and lease-based stampede protection at scale.

## Related: Deciding WHERE Caching Belongs at All

Before choosing eviction policies or cache layers, decide which endpoints actually need caching in the first place — see [[solution-arch/patterns/hot-path-first-design]] for the read/write and display/decision classification that answers this.

## Related: Full Platform-Team Scenario

For a worked system-design scenario that puts these strategies to use in a Tier-0, multi-tier, cost-constrained platform (the kind of problem an internal caching/data-stores team owns, not just a caller of the cache), see [[solution-arch/scenarios/design-tiered-caching-platform]].

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
- [[solution-arch/sources/hot-path-first-system-design]]
