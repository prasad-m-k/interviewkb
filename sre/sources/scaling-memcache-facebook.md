# Scaling Memcache at Facebook

**Type:** Academic Paper / Conference Paper
**Authors:** Nishtala, Fugal, Grimm, Kwiatkowski, Lee, Li, McElroy, Paleczny, Peek, Saab, Stafford, Tung, Venkataramani
**Venue:** USENIX NSDI 2013
**Ingested:** 2026-04-22
**Topics covered:** [[sre/topics/system-design]], [[sre/concepts/load-balancers]]

## Summary

This paper describes how Facebook scaled Memcache to handle billions of requests per second across multiple datacenters serving 1 billion users. It is one of the most influential papers on large-scale caching and distributed systems, and it directly informed the design of what became known as **McRouter** and the broader Meta caching infrastructure.

The paper is essential reading for anyone interviewing for Meta PE/SRE because interviewers explicitly expect candidates to know the concepts it describes. Saying "I've read the Memcache paper" and being able to discuss its key contributions signals seriousness.

## Key Takeaways

- **Single server → Cluster → Region → Multiple Regions:** Facebook's journey in scaling Memcache was incremental. Each phase solved the bottleneck of the previous phase. This "phase-by-phase scaling" narrative is a useful interview framework.
- **Lease mechanism:** The "thundering herd" problem occurs when a key expires and thousands of servers simultaneously miss cache and query the DB. Facebook solved this with **leases**: the first server to miss gets a lease token and fetches from DB; all others wait. A lease is a 10-second token. This is O(1) messages, not O(N) DB queries.
- **Pooling:** Not all keys are equal. "Small pools" for keys accessed by all web servers (everyone shares) vs. "regional pools" for keys that only a subset of servers need. Segmenting reduces cross-pool invalidation storms.
- **McRouter:** A proxy layer between web servers and Memcache clusters. McRouter handles routing, connection pooling (reduces # of connections per Memcache server), replication, and failover. The web server doesn't know which Memcache server to talk to — McRouter abstracts it.
- **Invalidation vs. update:** Facebook never updates a cached value when DB changes. It **deletes (invalidates)** the cached key. The next request will miss cache and reload from the updated DB. Why? Concurrent updates can create race conditions where the cache has a stale write-order.
- **Gutter pools:** When a Memcache server fails, rather than rehashing all keys (causing a cache stampede on the DB), traffic is redirected to a small "gutter pool" of spare servers. The gutter absorbs the miss load temporarily. When the original server recovers, gutter keys expire naturally.
- **Across regions (consistency):** Facebook uses a primary-region / secondary-region model. Writes go to the primary. Secondary regions get a replication stream. But secondary regions can have stale Memcache entries. Solution: "remote markers" — when a primary write invalidates a key, it also marks the key as "pending invalidation" in secondary regions, preventing stale reads until the replication catches up.

## What it Updated

- Informed [[sre/companies/meta]] — McRouter section; two-tier cache; thundering herd prevention; TAO vs Memcache distinction
- Informed [[sre/topics/system-design]] — cache design patterns; lease mechanism; invalidation vs. update; gutter pools

## Key Concepts for Interview Recall

### The Thundering Herd Problem and Leases
```
Without leases:
  Key expires → 10,000 web servers miss cache simultaneously
  → 10,000 DB queries at once → DB falls over

With leases:
  Key expires → server A misses, gets a 10-second lease token
  → servers B-Z also miss, but are told "wait, key has lease"
  → B-Z retry after 500ms; by then A has populated the cache
  → 1 DB query instead of 10,000
```

### McRouter Architecture
```
Web servers
    ↓
McRouter (handles routing, pools, failover)
    ↓
Memcache cluster A (pool for hot keys)
Memcache cluster B (pool for cold keys)
Gutter pool (spare capacity for failures)
```

### Delete vs. Update
```
WRONG:  DB write → cache write (race: two concurrent writes = wrong order)
RIGHT:  DB write → delete cache key (next read is a miss, refreshes from DB)
```

### What Facebook Numbers Tell You
At the time of the paper (2013), numbers (for interview context and "napkin math" training):
- 1 billion users
- Hundreds of millions of items in Memcache
- Billions of reads/writes per second
- ~28 millisecond average query latency reduction from caching
- 99% cache hit rate on hot keys

## Interview Questions This Paper Helps Answer

- "How do you design a distributed cache that avoids thundering herd?" → Lease tokens.
- "How do you invalidate cache when a DB record changes?" → Delete the key, not update.
- "What happens if a Memcache server dies?" → Gutter pool absorbs the miss; rehashing is avoided.
- "How do you scale a cache across multiple datacenters?" → Primary writes + replication + remote markers.
- "What is McRouter and why does Meta use it?" → Proxy for routing, pooling, replication, failover.

## Sources
- Nishtala et al., "Scaling Memcache at Facebook," USENIX NSDI 2013
- [[sre/companies/meta]]
