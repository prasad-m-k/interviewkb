# Database Sharding

**Topic:** [[solution-arch/topics/scalability-and-reliability]], [[solution-arch/topics/data-architecture]]
**Related:** [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/distributed-consensus]]

## What it is
Sharding (horizontal partitioning) splits a large dataset across multiple database nodes, where each node holds a subset of the data. Contrast with replication (all nodes hold the same data).

## How it works

```
Without Sharding:
  All 1B users ──▶ Single DB node (bottleneck)

With Sharding:
  Users A–M (500M) ──▶ Shard 1
  Users N–Z (500M) ──▶ Shard 2

  Users 0–333M ──▶ Shard 1 (hash range)
  Users 333M–666M ─▶ Shard 2
  Users 666M–1B ──▶ Shard 3
```

## Sharding Strategies

### Range-Based Sharding
```
key:  A–M  →  Shard 1
      N–Z  →  Shard 2

Pros:  Simple; range scans efficient (e.g., all orders in Jan)
Cons:  Hot spots if keys aren't uniform (e.g., all names start with A)
```

### Hash-Based Sharding
```
shard = hash(user_id) % num_shards

Pros:  Uniform distribution; no hot spots
Cons:  Range queries span all shards; resharding is disruptive
```

### Directory-Based Sharding
```
Shard lookup table:
  user_id 1–1000 → Shard 1
  user_id 1001–2000 → Shard 2
  ...
  (look up table to find shard)

Pros:  Flexible; can move data between shards
Cons:  Lookup table is a SPOF; extra round trip
```

### Consistent Hashing
```
Nodes arranged on a hash ring:
       0
      /  \
  Shard C  Shard A
     |      |
  Shard B---+

key = hash(key) → find nearest node clockwise
Adding/removing a node only moves ~1/N of the keys (vs all keys in modular hashing)
```
Used by: Cassandra, DynamoDB, Redis Cluster.

## Hot Spot Problem

```
Shard key: user_id
Celebrity user has 10M followers → their user_id gets 1000× more reads
→ Shard holding celebrity is overwhelmed

Fix 1: Compound key  user_id + random_suffix  (10 virtual shards per user)
Fix 2: Dedicated shard for known hot keys
Fix 3: Cache the hot key aggressively at application layer
```

## Cross-Shard Queries

```
SELECT * FROM orders WHERE user_id IN (123, 456, 789)
  → user 123 on Shard 1
  → user 456 on Shard 2
  → user 789 on Shard 3

Must query all 3 shards and merge results in application layer.
Joins across shards: extremely expensive or impossible.
```

**Rule:** Design shard key so most queries hit one shard (co-locate related data).

## Resharding

When a shard fills up, data must be moved to new shards. Painful without consistent hashing.

```
Old: 2 shards  → shard = hash(key) % 2
New: 3 shards  → shard = hash(key) % 3
  ALL existing keys map to different shards → must move all data
```

Consistent hashing: adding 1 node moves only `1/N` of data.

## When to shard

1. Vertical scaling (bigger server) is no longer cost-effective
2. Single-node DB write throughput is the bottleneck
3. Dataset doesn't fit on a single disk
4. You've already exhausted: indexing, query optimisation, read replicas, caching

**Don't shard prematurely.** Complexity cost is high. Most systems can go far with read replicas + caching.

## Common interview angles
- "When would you choose sharding over read replicas?"
- "What is consistent hashing and why is it better for sharding than modular hashing?"
- "How do you handle cross-shard transactions?" (Saga pattern, 2PC — both costly; design to avoid)
- "What is the hot key problem and how do you mitigate it?"

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
