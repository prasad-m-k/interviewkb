# Hash Map Internals

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/hash-collision-resolution]]

## What it is
A hash map (dict, HashMap, unordered_map) provides O(1) average-case get/set/delete by hashing keys to array indices. Understanding the internals — load factor, rehashing, hash function quality — is what separates a senior who can reason about production performance from one who just uses the API.

## The core machinery

```
key → hash function → raw hash → compress to index → bucket
```

**Compression**: `index = hash(key) % capacity`. Capacity is usually a power of 2, so this becomes a bitmask: `index = hash(key) & (capacity - 1)` — faster.

**Why prime or power-of-2 capacity?** Powers of 2 make the bitmask trick cheap. Some implementations use primes because `hash % prime` distributes more uniformly when the hash function has patterns (e.g., all even values).

## Load factor

```
load_factor = num_entries / capacity
```

The load factor controls the tradeoff between memory and collision rate.

- **Too low** (e.g., 0.1): wastes memory, few collisions
- **Too high** (e.g., 0.9): many collisions, O(n) operations in the worst case
- **Typical threshold**: 0.75 (Java HashMap default); Python resizes at 2/3

When load factor exceeds the threshold → **rehash**.

## Rehashing

Rehashing allocates a new, larger array (typically 2× capacity) and reinserts every existing entry.

```
old array (capacity 8) → threshold exceeded → new array (capacity 16) → reinsert all keys
```

**Why is this O(n) and not a problem?** Because it happens infrequently — the array doubles each time, so the total work across all insertions is O(n) amortized (same argument as dynamic array doubling — see [[dsa/concepts/dynamic-array-amortized]]).

**The catch**: a single insert that triggers rehash is O(n) worst-case latency. In latency-sensitive systems (real-time, trading, game loops), this spike matters. Solutions:
- Pre-size the map: `HashMap(expectedSize / loadFactor)` to avoid rehashing entirely
- Incremental rehashing: Redis does this — it maintains two hash tables and migrates entries gradually

## Python dict internals (CPython 3.6+)
Python's dict is open-addressed with a compact key-value store:
- Entries array stores `(hash, key, value)` in insertion order
- Indices array maps hash → position in entries array
- Resizes at 2/3 load factor; new capacity = `next power of 2 ≥ 4/3 * current size`
- Hash randomization: `PYTHONHASHSEED` is randomized per process to prevent hash-flooding DoS attacks

## Java HashMap internals (Java 8+)
- Separate chaining, but with a twist: when a chain length exceeds **8**, it converts to a **red-black tree** (treeification) — O(log n) worst case per bucket instead of O(n)
- When the tree shrinks below **6** entries, it converts back to a linked list
- Default load factor: 0.75; default initial capacity: 16

## Hash function quality

A hash function must be:
1. **Deterministic**: same key always → same hash
2. **Uniform**: output distributes evenly across the range
3. **Fast**: O(1) to compute

Bad hash functions cause clustering. Example: a hash that returns `len(key)` maps all 5-letter words to the same bucket.

**Avalanche effect**: a good hash ensures that flipping a single bit in the input changes ~50% of output bits. MurmurHash, xxHash, and SipHash all have this property.

## When HashMap assumptions break

| Scenario | Problem | Fix |
|---|---|---|
| Adversarial key input (hash flooding) | Attacker crafts keys that all collide → O(n) per op → DoS | Randomized hash seed (Python, Rust default) |
| Mutable keys | Mutating a key after insertion corrupts the map | Only use immutable keys (strings, ints, tuples in Python) |
| Custom `equals` without `hashCode` (Java) | Two equal objects hash differently → key not found | Always override both together |
| Huge map with frequent deletes | Tombstone accumulation (open addressing) degrades performance | Trigger rehash explicitly |

## Common interview angles
- "What's the time complexity of HashMap operations?" — O(1) average, O(n) worst case; explain why
- "How does Java HashMap avoid O(n) worst case?" — treeification to red-black tree at chain length 8
- "What is load factor and why does it matter?" — controls collision rate; rehash at threshold
- "Can you use a mutable object as a HashMap key?" — no, mutations corrupt the structure; explain why
- "How would you pre-size a HashMap?" — `new HashMap<>(n * 4/3 + 1)` to avoid rehashing for n entries

## Sources
*(none yet)*
