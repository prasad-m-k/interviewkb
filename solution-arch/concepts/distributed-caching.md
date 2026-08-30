---
uid: 634049cd-914a-41ad-915e-4f55c29c1749
---

# Distributed Caching

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/caching]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/distributed-consensus]], [[solution-arch/patterns/hot-path-first-design]]
**Deep-dive (production war story):** [[sre/sources/scaling-memcache-facebook]]

> **Scope note:** [[solution-arch/concepts/caching]] covers caching *strategies* (cache-aside, write-through, eviction policies, cache layers) — read that first. This page covers what changes when a cache is no longer one box: **sharding a cache across many nodes, replicating it, and keeping it coherent** at a scale where a single Redis/Memcached instance can't hold the working set or handle the request rate.

## What it is

A distributed cache spreads cached data across multiple nodes so that (a) the total cached dataset can exceed one machine's RAM, and (b) request throughput scales beyond one node's connection/CPU limit. This introduces problems a single-node cache doesn't have: which node owns a given key, what happens when a node dies, and how multiple regions stay roughly in sync.

## How it works

### Sharding a Cache: Consistent Hashing

```
NAIVE sharding — hash(key) % N:
  node = hash("product:123") % 4   → node 2

  Problem: when N changes (a node is added or removed), almost
  EVERY key remaps to a different node — mod-N shifts nearly all
  keys, causing a near-total cache wipe (every key misses, every
  request hits the DB simultaneously) exactly when the cluster is
  already in a fragile state (scaling event or node failure).

CONSISTENT HASHING — the standard fix:
  Both nodes and keys are hashed onto the same circular keyspace
  (a "hash ring"). A key belongs to the first node found walking
  clockwise from the key's position.

        node D           node A
           ●─────────────────●
          /                   \
         /      hash ring      \
        │                       │
         \                     /
          \                   /
           ●─────────────────●
        node C           node B

  key "product:123" hashes to a point between node A and node B
  on the ring → owned by node B (next node clockwise)

  Adding node E only remaps the keys between E and its counter-
  clockwise neighbor — NOT the whole keyspace. Typically only
  ~1/N of keys move when a node joins/leaves, instead of nearly
  all of them.

  Virtual nodes (vnodes): each physical node is placed at MANY
  points on the ring (e.g. 100-200 virtual positions per physical
  node) so that load distributes evenly even with few physical
  nodes, and a single node failure spreads its lost keys across
  many surviving nodes rather than dumping them all onto one
  neighbor.
```

### Client-Side Sharding vs Proxy-Based Routing

```
Client-side sharding (classic Memcached model):
  App server itself runs the consistent-hashing logic and picks
  which Memcached node to talk to directly.

  App Server ──(computes hash, picks node)──▶ Memcached node 3

  + No extra network hop
  - Every client (in every language/service) must implement the
    SAME hashing scheme consistently, or different services will
    disagree on which node owns a key

Proxy-based routing (McRouter / Twemproxy / Redis Cluster's
routing, envoy-based cache proxies):
  A proxy layer sits between clients and cache nodes, owns the
  routing logic centrally.

  App Server ──▶ McRouter/Proxy ──(routes)──▶ Cache node 3

  + Routing logic lives in ONE place — connection pooling, failover,
    and re-sharding logic don't need to be reimplemented per client
    language
  + Proxy can implement pooling (see below) and failover
    transparently to callers
  - Extra network hop; proxy itself must be highly available
    (usually run as a fleet, not a single instance)

This is the exact architecture Meta's McRouter implements at
massive scale — see [[sre/sources/scaling-memcache-facebook]] for
the full production case study.
```

### Replication Topologies

```
Leader-follower (most common):
  Writes go to the leader; followers replicate asynchronously and
  serve reads. Same trade-off as DB read replicas: replication lag
  means followers can serve STALE data briefly after a write.

    Write ──▶ Leader cache node
                  │ async replication
                  ▼
             Follower node(s) ──▶ serve reads

Multi-leader / active-active across regions:
  Each region has its own writable cache; changes propagate to
  other regions asynchronously. Needed for multi-region active-
  active systems, but reintroduces the same conflict-resolution
  questions as any multi-leader replication scheme (see
  [[solution-arch/concepts/distributed-consensus]] for the general
  problem class this is an instance of).

No replication (pure sharding, no redundancy):
  Simplest, cheapest — but a node failure means a hard cache miss
  for every key that node owned, all at once, all hitting the DB
  simultaneously. This is exactly the failure mode gutter pools
  (below) exist to absorb.
```

### Cross-Region Cache Consistency

```
Problem: Primary region writes and invalidates its cache. A
secondary region's cache still holds the OLD value until
replication catches up — a user hitting the secondary region reads
stale data with no indication anything is wrong.

Facebook's solution — "remote markers" (see
[[sre/sources/scaling-memcache-facebook]]):
  1. Primary region write invalidates the key locally
  2. The invalidation ALSO writes a "pending invalidation" marker
     into the secondary region's cache for that key
  3. Any read in the secondary region that sees the marker is
     forced to go to the (replicated) DB instead of trusting a
     stale cached value, until replication catches up and clears
     the marker

This is architecturally the same idea as the invalidation-event
propagation in [[solution-arch/patterns/hot-path-first-design]]'s
outbox-driven cache invalidation — just extended across a
replication boundary instead of within one region.
```

### Failure Handling at Scale: Gutter Pools

```
Naive failure handling: node dies → rehash its keys onto surviving
nodes (via consistent hashing) → but ALL those keys are now cache
MISSES simultaneously → a stampede of DB queries right when the
cluster is already degraded.

Gutter pool pattern: keep a small pool of spare cache nodes on
standby. When a node fails, traffic for its keys routes to the
gutter pool instead of triggering a rehash. The gutter absorbs
the miss load temporarily (populating itself from the DB as
requests come in) without redistributing load onto the ALREADY-
BUSY surviving primary nodes. When the failed node recovers, gutter
entries simply expire — no rebalancing step needed.
```

### Thundering Herd / Cache Stampede at Distributed Scale

```
Same problem as single-node caching (see "cache stampede" in
[[solution-arch/concepts/caching]]) but at distributed scale, a
popular key's expiry can trigger THOUSANDS of app servers missing
simultaneously — not a handful.

Lease-based fix (Facebook's approach):
  Key expires → first server to miss gets a short-lived lease token
  and is the ONLY one allowed to query the DB and repopulate the
  cache. Every other server that misses the same key during the
  lease window is told to wait/retry shortly, instead of also
  querying the DB.

  Without leases: N servers miss → N simultaneous DB queries
  With leases:    N servers miss → 1 DB query, N-1 short waits

Complementary/simpler alternatives when a full lease mechanism is
overkill:
  - Mutex/distributed lock per key (Redis SETNX) — same idea,
    simpler to implement, less battle-tested at extreme scale
  - Probabilistic early expiry — recompute slightly before actual
    TTL, with probability increasing as expiry nears, spreading
    refreshes out instead of all hitting at the exact expiry instant
```

## Complexity

```
Consistent hashing lookup:      O(log n) with a sorted ring + binary
                                 search (n = number of virtual nodes)
Rebalancing on node join/leave:  ~O(K/N) keys move (K = total keys,
                                 N = number of nodes) — vs O(K) for
                                 naive mod-N hashing
```

## When to use

```
Use distributed caching (sharded across many nodes) when:
  ✅ Working set exceeds single-node memory capacity
  ✅ Request rate exceeds a single node's connection/CPU capacity
  ✅ You're already at the scale where [[solution-arch/patterns/hot-path-first-design]]
     has identified specific hot, read-heavy endpoints that a single
     cache node can't keep up with

A single cache node (or simple leader-follower pair) is enough when:
  ✅ Working set and request rate comfortably fit one instance —
     the operational complexity of sharding/consistent hashing/
     gutter pools isn't justified yet. Revisit this decision as part
     of the endpoint classification step, not preemptively.
```

## Common interview angles

```
Q: "Why not just use hash(key) % N to pick a cache node?"
A: Because changing N (scaling the cluster, or a node failing)
   remaps almost every key at once under modulo hashing, causing a
   near-total cache wipe exactly when the system is already
   stressed. Consistent hashing bounds the remapped fraction to
   roughly 1/N.

Q: "A popular product page's cache key just expired and you're
    getting paged for DB overload. What happened and how do you fix
    it?"
A: Cache stampede/thundering herd — many servers missed the same
   hot key simultaneously and all queried the DB at once. Fix with
   a lease mechanism (first miss gets exclusive rights to repopulate,
   others wait) or a distributed lock per key; see Facebook's lease
   approach in [[sre/sources/scaling-memcache-facebook]].

Q: "How do you keep a secondary region's cache from serving stale
    data after a primary-region write?"
A: Remote invalidation markers — the primary's write propagates a
   "pending invalidation" marker to the secondary region's cache
   synchronously (fast, small message) even though the actual data
   replication is asynchronous (slower); reads that see the marker
   fall through to the database instead of trusting the stale cached
   value until replication catches up.

Q: "Client-side sharding vs a routing proxy (like McRouter) — what's
    the trade-off?"
A: Client-side avoids an extra network hop but requires every
   calling service (potentially in different languages) to implement
   identical hashing/failover logic — a consistency risk. A routing
   proxy centralizes that logic once, at the cost of an extra hop
   and needing the proxy layer itself to be highly available.
```

## Examples

```
E-commerce product catalog cache (ties to the hot-path case study
in [[solution-arch/patterns/hot-path-first-design]]):
  - product:{id} keys sharded via consistent hashing across a
    Redis Cluster (or Memcached pool behind McRouter)
  - Small pool for globally-hot keys (bestsellers, homepage
    featured items) shared by all web servers
  - Regional pools for long-tail catalog items, reducing cross-pool
    invalidation traffic
  - Gutter pool absorbs any single node's failure without a
    stampede on the primary DB
```

## Related: Full Platform-Team Scenario

For a worked scenario applying zone-aware replication, gutter pools, and cost-driven tiering to a real Tier-0 platform-team problem, see [[solution-arch/scenarios/design-tiered-caching-platform]].

## Sources
- [[sre/sources/scaling-memcache-facebook]]
- [[solution-arch/concepts/caching]]
- [[solution-arch/sources/designing-data-intensive-applications]]
