# LRU Cache

**Difficulty:** Medium
**Topic:** [[dsa/topics/data-structures]]
**Pattern:** HashMap + Doubly Linked List
**Companies:** Amazon, Google, Meta, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Design a data structure that follows the Least Recently Used (LRU) cache eviction policy.

Implement `LRUCache(capacity)`, `get(key)` → value or -1, and `put(key, value)`. Both operations must run in **O(1)** time.

When the cache exceeds capacity, evict the least recently used item.

## Approach
O(1) `get` requires a HashMap for instant key lookup. O(1) eviction requires knowing which item was used least recently and removing it without scanning — that's a Doubly Linked List, where you can move a node to the "most recently used" end and remove the "least recently used" end in O(1) given a pointer to the node.

Combine them: HashMap maps key → DLL node. DLL orders nodes by recency (head = LRU, tail = MRU). Two dummy sentinel nodes (head, tail) eliminate all edge cases.

**On `get(key)`:** move the node to the tail (MRU end). Return its value.

**On `put(key, value)`:** if key exists, update and move to tail. If new, create node, insert at tail, add to map. If over capacity, remove the node at `head.next` (LRU), delete from map.

## Solution (Python)
```python
class Node:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.cache = {}          # key -> Node
        self.head = Node()       # dummy LRU sentinel
        self.tail = Node()       # dummy MRU sentinel
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: Node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _insert_tail(self, node: Node):
        node.prev = self.tail.prev
        node.next = self.tail
        self.tail.prev.next = node
        self.tail.prev = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self._remove(self.cache[key])
        self._insert_tail(self.cache[key])
        return self.cache[key].val

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self.cache[key] = node
        self._insert_tail(node)
        if len(self.cache) > self.cap:
            lru = self.head.next
            self._remove(lru)
            del self.cache[lru.key]
```

## Complexity
Time: O(1) get and put | Space: O(capacity)

## Key insight
The dummy head/tail sentinels are not optional style — they eliminate all nil-pointer edge cases in `_remove` and `_insert_tail`. Without them, every operation needs special-case checks for empty list, single node, removing the head, removing the tail.

## Variants / follow-ups
- **LFU Cache** (Least Frequently Used): add a frequency layer — a HashMap of frequency → DLL. Harder; O(1) still achievable.
- **Thread-safe LRU**: wrap operations with a mutex; or use a concurrent hash map + lock striping.
- **TTL Cache**: add expiry timestamps; background sweep or lazy expiry on access.
- **Interviewers ask**: "How would you handle concurrent access?" → Lock the whole structure (simple) or use striped locks per bucket (scalable).

## Sources
- [[DSA overview]]
