# Data Structures

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/topics/algorithms]]

High-level survey of data structures that appear in FAANG interviews for senior MLOps engineers.

> **Senior-level internals:** Junior candidates know the API. Senior candidates know *why* operations have the complexity they do and what breaks under load. Interviewers probe this with "how does X work under the hood?" or "what if you have 10 million keys?" Each section below links to a deep-dive internals page.

---

## HashMap / HashSet

O(1) average lookup, insert, delete. The single most-used structure in interview problems.

- Python: `dict`, `set`, `collections.defaultdict`, `collections.Counter`
- When to reach for it: any time you need "have I seen this before?" or frequency counting
- Pitfall: worst-case O(n) with hash collisions — mention it but don't dwell on it

**See:** [[dsa/concepts/hash-map]] | **Internals:** [[dsa/concepts/hash-collision-resolution]], [[dsa/concepts/hash-map-internals]]
**Problems:** [[dsa/problems/lru-cache]], [[dsa/problems/top-k-frequent-elements]], [[dsa/problems/design-hit-counter]]

---

## Heap (Priority Queue)

O(log n) insert and extract-min/max. Maintains a partial order — not fully sorted.

- Python: `heapq` is a min-heap; negate values for max-heap
- When to reach for it: "Top K", streaming statistics, merge sorted sources, scheduling
- `heapq.nlargest(k, iterable)` / `heapq.nsmallest(k, iterable)` for one-shot queries

**See:** [[dsa/concepts/heap]] | **Internals:** [[dsa/concepts/heap-internals]]
**Problems:** [[dsa/problems/find-median-from-data-stream]], [[dsa/problems/top-k-frequent-elements]], [[dsa/problems/merge-k-sorted-lists]], [[dsa/problems/task-scheduler]]

---

## Deque (Double-Ended Queue)

O(1) append and pop from both ends. Foundation for monotonic queue patterns.

- Python: `collections.deque`
- When to reach for it: sliding window with O(1) max/min, BFS queues
- Monotonic deque: maintain invariant (increasing or decreasing) by popping from the back before appending

**See:** [[dsa/concepts/deque]]
**Problems:** [[dsa/problems/sliding-window-maximum]], [[dsa/problems/design-hit-counter]]

---

## Linked List (Doubly)

O(1) insert/delete at any position given a pointer. Rarely built from scratch — usually combined with a HashMap.

- When to reach for it: LRU Cache (O(1) move-to-front requires both pointer access and O(1) lookup)
- Python: implement with a `Node` class; use dummy head/tail sentinels to avoid edge cases

**Problems:** [[dsa/problems/lru-cache]], [[dsa/problems/merge-k-sorted-lists]]

---

## Graph (Adjacency List)

Vertices + directed or undirected edges. Represented as `List[List[int]]` or `dict[int, list[int]]`.

- When to reach for it: dependencies, connectivity, shortest paths
- Build from edge list: `for u, v in edges: graph[u].append(v)`

**See:** [[dsa/concepts/graph]]
**Problems:** [[dsa/problems/course-schedule-ii]], [[dsa/problems/number-of-islands]]

---

## Array / Matrix

The default starting point. Contiguous memory, O(1) index access.

- Matrix problems: think of each cell as a graph node with up/down/left/right neighbors
- In-place marking (e.g., set visited cells to '0') avoids O(m*n) visited set

**Internals:** [[dsa/concepts/dynamic-array-amortized]] — why `append` is O(1) amortized
**Problems:** [[dsa/problems/number-of-islands]], [[dsa/problems/sliding-window-maximum]]

---

## Trees (BST / Balanced)

O(log n) search, insert, delete when balanced. Used in sorted maps, range queries, order statistics.

- A plain BST degrades to O(n) on sorted inserts — balance is essential
- AVL trees: strict balance, better reads; Red-Black trees: looser balance, better writes
- Databases use B-trees (not binary trees) to minimize disk I/O per lookup

**Internals:** [[dsa/concepts/tree-balancing]] — BST → AVL → Red-Black → B-Tree; the database B-tree question
