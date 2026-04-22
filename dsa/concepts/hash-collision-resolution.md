# Hash Collision Resolution

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/hash-map-internals]]

## What it is
A collision occurs when two distinct keys hash to the same bucket. Since hash functions map an infinite key space to a finite array, collisions are inevitable. The collision resolution strategy determines how the structure stores and retrieves these keys.

## The two families

### 1. Separate Chaining
Each bucket holds a pointer to a linked list (or another structure) of all entries that hash to that bucket.

```
index 3 → [ ("cat", 42) → ("act", 7) → None ]
index 4 → [ ("dog", 99) → None ]
```

- **Lookup**: hash the key → walk the chain comparing keys → O(1) average, O(n) worst case (all keys in one chain)
- **Insert**: hash → prepend to chain → O(1)
- **Delete**: hash → find in chain → unlink → O(1) average
- **Load factor tolerance**: degrades gracefully above load factor 1.0 — chains just get longer
- **Used by**: Java `HashMap`, Python `dict` (historically), most standard library maps

**The "bad hash function" trap**: if all n keys collide into one bucket, every lookup is O(n). A good hash function distributes keys uniformly. This is why Java `HashMap` applies a secondary hash (`hash ^ (hash >>> 16)`) to reduce sensitivity to poor `hashCode()` implementations.

### 2. Open Addressing
All entries live in the array itself — no pointers, no external chains. On collision, probe for the next available slot according to a probing sequence.

**Linear probing**: on collision at index `i`, try `i+1`, `i+2`, `i+3`, ...
```
insert "cat" → hash → index 3 (occupied) → try 4 (empty) → store at 4
lookup "cat" → hash → index 3, check key (not "cat") → probe 4, check key ("cat") → found
```

**Quadratic probing**: try `i+1²`, `i+2²`, `i+3²`, ... — reduces primary clustering

**Double hashing**: use a second hash function to compute the step size: `i + k * hash2(key)` — eliminates clustering patterns entirely

**Tombstone problem**: deleting a key in open addressing is tricky. You can't just clear the slot — that breaks the probe chain for later keys. Instead, mark the slot with a *tombstone* sentinel. Tombstones count as occupied during probing but as empty for insertion. Too many tombstones degrade performance; rehashing clears them.

```
delete "cat" at index 4: mark as TOMBSTONE
lookup "dog" (which was stored at index 5): hash → 3 (miss) → 4 (tombstone, keep probing) → 5 (found)
```

## Chaining vs Open Addressing

| | Separate Chaining | Open Addressing |
|---|---|---|
| Memory | Extra pointer overhead per entry | Dense array — cache-friendly |
| Load factor | Works above 1.0 | Must stay below ~0.7 |
| Cache behavior | Poor (pointer chasing) | Good (sequential probing) |
| Deletion | Simple | Requires tombstones |
| Worst case | O(n) — all keys in one chain | O(n) — full table, all probes |
| Used by | Java HashMap, most std libs | Python dict (since 3.6), C++ `dense_hash_map`, Robin Hood hashing |

## Robin Hood hashing (bonus — impresses interviewers)
A variant of open addressing where during insertion, if the new key has traveled farther from its home bucket than the key currently sitting in the probed slot, they swap. This "steals from the rich, gives to the poor" — it minimizes the maximum probe length across all keys, which dramatically improves worst-case lookup. Used in Rust's `HashMap`.

## Common interview angles
- "What happens on a hash collision?" — describe chaining or open addressing
- "How does Python's dict handle collisions?" — open addressing with a pseudorandom probe sequence (not linear)
- "What's the worst-case for a HashMap?" — O(n), when all keys collide (bad hash function or adversarial input)
- "How do you make a hash map resistant to adversarial inputs?" — randomize the hash seed at startup (Python does this: `PYTHONHASHSEED`)

## Sources
*(none yet)*
