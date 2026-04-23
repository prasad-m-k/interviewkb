# Collections Framework — Deep Dive

**Topic:** [[java/topics/core-language]]
**Related:** [[java/concepts/synchronization]], [[java/concepts/java-memory-model]]

## The Collections Hierarchy

```
Iterable
└── Collection
    ├── List
    │   ├── ArrayList
    │   ├── LinkedList
    │   └── CopyOnWriteArrayList
    ├── Set
    │   ├── HashSet (backed by HashMap)
    │   ├── LinkedHashSet (backed by LinkedHashMap)
    │   └── TreeSet (backed by TreeMap / Red-Black Tree)
    └── Queue / Deque
        ├── ArrayDeque
        ├── LinkedList
        ├── PriorityQueue
        └── BlockingQueue (java.util.concurrent)

Map (not extending Collection)
├── HashMap
├── LinkedHashMap
├── TreeMap
├── WeakHashMap
└── ConcurrentHashMap
```

---

## HashMap — Complete Internals

### Structure
```java
// Simplified view of HashMap (Java 8+)
class HashMap<K, V> {
    Node<K,V>[] table;         // array of buckets
    int size;                  // number of key-value pairs
    int threshold;             // resize when size > threshold
    float loadFactor;          // default 0.75
    int modCount;              // tracks structural modifications (for fail-fast)
}

static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;           // linked list within bucket
}
```

### `put(key, value)` — Step by Step
```
1. key == null → store at bucket 0 (HashMap allows one null key)
2. hash = hash(key)          // spreads high bits into low bits
   hash = key.hashCode() ^ (key.hashCode() >>> 16)
3. bucket = hash & (table.length - 1)   // bitwise AND (faster than %)
4. If bucket is empty → insert new Node
5. If bucket has nodes → walk list comparing hash AND key.equals()
   - Found: update value
   - Not found: append new Node
6. If bucket size > TREEIFY_THRESHOLD (8):
   - If table.length < MIN_TREEIFY_CAPACITY (64): resize table instead
   - Else: convert bucket from LinkedList → TreeNode (Red-Black Tree)
7. If ++size > threshold: resize (double table, rehash all entries)
```

### The `hashCode()` Spread (Perturbation)
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
XOR-ing the high 16 bits into the low 16 bits ensures the high bits participate in bucket selection even when `table.length` is small (spread prevents clustering).

### Resizing (Rehashing)
When `size > capacity × loadFactor`:
- New capacity = old capacity × 2 (always power of 2)
- Rehash: every entry moves to either the same index OR `old_index + old_capacity`
- Why? With power-of-2 sizing, the new bit added to the index calculation is either 0 (stays) or 1 (moves by exactly old capacity). No full hash recomputation needed.

### Interview Questions

**Q: Why is HashMap not thread-safe?**
Concurrent `put` operations can cause infinite loops (Java 7) or data loss (Java 8+) during resizing. Use `ConcurrentHashMap`.

**Q: What's the default load factor and why 0.75?**
It balances time (collision avoidance) vs. space (waste). Below 0.75, collisions are rare but memory is wasted. Above 0.75, space is saved but chains get long.

**Q: Why must keys be immutable?**
If a key's `hashCode()` changes after insertion, the entry is in the wrong bucket and can never be found. `String` is a perfect HashMap key: immutable and caches its hash.

---

## LinkedHashMap — Insertion and Access Order

`LinkedHashMap` maintains a doubly-linked list running through all entries in either **insertion order** (default) or **access order**.

```java
// Access-order LinkedHashMap (most-recently-used at tail)
Map<K, V> lru = new LinkedHashMap<K, V>(capacity, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > MAX_CAPACITY;
    }
};
```

**This is the basis of a thread-unsafe LRU cache.** For a thread-safe LRU, wrap with `ConcurrentHashMap` + a separate eviction queue, or use Caffeine.

---

## TreeMap — Red-Black Tree Internals

`TreeMap` stores entries in sorted order by key (natural ordering or `Comparator`). Backed by a Red-Black Tree: guaranteed O(log n) for `get`, `put`, `remove`.

```java
// Range queries — TreeMap's superpower
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "a"); map.put(3, "c"); map.put(5, "e"); map.put(7, "g");

map.subMap(3, 6);           // {3=c, 5=e}  — keys in [3, 6)
map.headMap(5);             // {1=a, 3=c}  — keys < 5
map.tailMap(3);             // {3=c, 5=e, 7=g} — keys >= 3
map.floorKey(4);            // 3  — greatest key <= 4
map.ceilingKey(4);          // 5  — smallest key >= 4
```

**Interview use:** Whenever a problem says "find the nearest X" or "range query on sorted data", think `TreeMap`.

---

## ArrayList vs LinkedList

| Operation | ArrayList | LinkedList |
|---|---|---|
| `get(i)` | O(1) — random access | O(n) — traverse from head |
| `add(end)` | O(1) amortized | O(1) |
| `add(middle)` | O(n) — shift elements | O(n) — find position, O(1) insert |
| `remove(middle)` | O(n) — shift elements | O(n) — find position, O(1) remove |
| Memory | Compact (int[]) — cache-friendly | 24 bytes/node overhead — cache-hostile |
| Iteration | Very fast (CPU prefetch) | Slow (pointer chasing) |

**Rule of thumb:** Use `ArrayList` for almost everything. `LinkedList` only when you need O(1) add/remove at both ends AND never random access (e.g., implementing a Deque). `ArrayDeque` is almost always better than `LinkedList` for queue use cases.

---

## Fail-Fast vs Fail-Safe Iterators

### Fail-Fast (most collections: ArrayList, HashMap)
Throws `ConcurrentModificationException` if the collection is structurally modified while being iterated.

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) list.remove(s); // ConcurrentModificationException!
}

// Correct: use Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // OK
}
```

**How:** `modCount` is incremented on every structural modification. The iterator captures `expectedModCount` at creation and compares on each `next()` call.

### Fail-Safe (java.util.concurrent collections)
Operate on a **snapshot** or use lock-free techniques — never throw `ConcurrentModificationException`.

```java
// CopyOnWriteArrayList: writes copy the entire array; reads use the old array
List<String> list = new CopyOnWriteArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    list.add("d"); // OK — iterating over old snapshot; "d" not visible in this iteration
}
```

**CopyOnWriteArrayList:** Writes are O(n) (copy entire array). Reads are O(1) and never contend. Perfect for **read-heavy** workloads with very infrequent writes (e.g., listener/subscriber lists).

---

## PriorityQueue — Min-Heap

`PriorityQueue` is a min-heap by default. The head is always the smallest element.

```java
// Min-heap (default)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// Custom objects
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]); // sort by index 1

// Key operations (all O(log n) except peek):
pq.offer(5);         // O(log n) — add element, sift up
pq.poll();           // O(log n) — remove min, sift down
pq.peek();           // O(1) — view min without removing
```

**Interview pattern:** Top-K elements. Maintain a min-heap of size K. When a new element > heap min, replace min with new element. Final heap contains top K.

---

## ArrayDeque vs Stack/LinkedList for Deque Operations

`ArrayDeque` is the preferred implementation for both Stack and Queue use cases:

```java
Deque<Integer> deque = new ArrayDeque<>();
deque.push(1);        // stack: add to front
deque.pop();          // stack: remove from front
deque.offerLast(1);   // queue: enqueue at tail
deque.pollFirst();    // queue: dequeue from head
```

**Do not use `Stack` class** (extends `Vector`, synchronized, legacy). Use `ArrayDeque` — it's faster and unsynchronized. For thread-safe use, use `LinkedBlockingDeque`.

---

## WeakHashMap — GC-Friendly Cache

```java
// Keys are weakly referenced — GC can reclaim them
Map<Object, String> cache = new WeakHashMap<>();
Object key = new Object();
cache.put(key, "value");
key = null;          // when GC runs, the entry may be removed from the map
System.gc();
// cache.get() may return null now
```

**Use case:** Caching where you don't want the cache to prevent GC of the key objects. Not suitable as a general cache (GC timing is unpredictable). Use `SoftReference` values in a `HashMap` for more control.

---

## Interview Quick Reference

| Collection | Ordered? | Sorted? | Null Keys? | Null Values? | Thread-Safe? |
|---|---|---|---|---|---|
| `ArrayList` | Yes (insertion) | No | N/A | Yes | No |
| `LinkedList` | Yes (insertion) | No | N/A | Yes | No |
| `HashSet` | No | No | 1 null | N/A | No |
| `LinkedHashSet` | Yes (insertion) | No | 1 null | N/A | No |
| `TreeSet` | Yes (sorted) | Yes | No | N/A | No |
| `HashMap` | No | No | 1 null key | Yes | No |
| `LinkedHashMap` | Yes (insertion/access) | No | 1 null key | Yes | No |
| `TreeMap` | Yes (sorted) | Yes | No | Yes | No |
| `ConcurrentHashMap` | No | No | No | No | Yes |
| `CopyOnWriteArrayList` | Yes | No | N/A | Yes | Yes |

## Sources
- [[java/concepts/jvm-internals]]
- [[java/concepts/synchronization]]
