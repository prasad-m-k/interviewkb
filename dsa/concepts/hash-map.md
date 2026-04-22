# Hash Map

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/heap]], [[dsa/problems/lru-cache]], [[dsa/problems/top-k-frequent-elements]]
**Internals (senior-level):** [[dsa/concepts/hash-collision-resolution]], [[dsa/concepts/hash-map-internals]]

## What it is
A hash map (dictionary) maps keys to values using a hash function to compute an array index. Provides O(1) average-case insert, lookup, and delete.

## How it works
1. Compute `hash(key) % capacity` to get a bucket index
2. Handle collisions via chaining (linked list per bucket) or open addressing
3. Resize (rehash) when the load factor exceeds a threshold (~0.75)

Python's `dict` is implemented as an open-addressing hash table. Keys must be hashable (immutable).

## Complexity
| Operation | Average | Worst |
|---|---|---|
| Insert | O(1) | O(n) |
| Lookup | O(1) | O(n) |
| Delete | O(1) | O(n) |
| Space | O(n) | O(n) |

Worst case occurs with hash collisions (adversarial inputs). In practice, treat as O(1).

## When to use
- Frequency counting: `collections.Counter(items)`
- Seen-before check: `if x in seen`
- Group-by: `collections.defaultdict(list)`
- Complement lookup: "two sum" pattern — store `target - num` as you iterate
- Combined with doubly linked list for O(1) ordered-eviction (LRU Cache)

## Common interview angles
- **Two Sum**: single pass with complement map
- **Group Anagrams**: key = sorted(word), value = [words]
- **Subarray Sum Equals K**: prefix sum + frequency map
- **LRU Cache**: HashMap + DLL for O(1) get and put

## Examples
```python
# Frequency count
from collections import Counter
count = Counter([1, 2, 2, 3, 3, 3])  # {3: 3, 2: 2, 1: 1}

# Complement lookup (Two Sum)
def two_sum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        if target - n in seen:
            return [seen[target - n], i]
        seen[n] = i

# defaultdict avoids KeyError on missing keys
from collections import defaultdict
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)
```

## Sources
- [[DSA overview]]
