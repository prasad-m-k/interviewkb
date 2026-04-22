# Heap (Priority Queue)

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/hash-map]], [[dsa/patterns/two-heaps]]
**Internals (senior-level):** [[dsa/concepts/heap-internals]]

## What it is
A heap is a complete binary tree that satisfies the heap property: every parent is ≤ its children (min-heap) or ≥ its children (max-heap). Supports efficient access to the minimum (or maximum) element.

## How it works
- Stored as an array: parent of index `i` is at `(i-1)//2`; children are at `2i+1` and `2i+2`
- **Push**: append to end, then sift up (swap with parent while heap property violated)
- **Pop**: swap root with last element, remove last, then sift down

Python's `heapq` is a **min-heap**. For a max-heap, negate values: push `-val`, pop and negate.

## Complexity
| Operation | Time |
|---|---|
| Push | O(log n) |
| Pop min/max | O(log n) |
| Peek min/max | O(1) |
| Heapify (build from list) | O(n) |
| Space | O(n) |

## When to use
- **Top K elements**: push all, maintain heap of size K → O(n log k)
- **Merge K sorted lists/streams**: use min-heap to always pull the smallest next element
- **Scheduling with priorities**: task scheduler, Dijkstra's algorithm
- **Streaming median**: two heaps partition the stream (see [[dsa/patterns/two-heaps]])
- **Kth largest/smallest**: maintain a heap of size K

## Common interview angles
- "Find the K largest elements" → min-heap of size K (pop when size > K)
- "Find the K most frequent" → heap keyed on frequency
- "Merge K sorted arrays" → min-heap with (value, list_index, element_index)
- "Median of a stream" → max-heap (lower half) + min-heap (upper half)

## Examples
```python
import heapq

# Min-heap
h = []
heapq.heappush(h, 3)
heapq.heappush(h, 1)
heapq.heappush(h, 2)
heapq.heappop(h)  # returns 1

# Max-heap (negate)
max_h = []
heapq.heappush(max_h, -5)
heapq.heappush(max_h, -3)
-heapq.heappop(max_h)  # returns 5

# Heapify in-place O(n)
nums = [3, 1, 4, 1, 5]
heapq.heapify(nums)

# Top K largest
heapq.nlargest(3, nums)   # [5, 4, 3]
heapq.nsmallest(3, nums)  # [1, 1, 3]

# Heap of tuples: sorted by first element
heapq.heappush(h, (priority, item))
```

## Sources
- [[DSA overview]]
