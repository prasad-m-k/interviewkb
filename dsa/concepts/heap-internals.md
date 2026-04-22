# Heap Internals — Binary Heap and O(n) Heapify

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/tree-balancing]]

## What it is
A binary heap is a complete binary tree (all levels full except possibly the last, filled left-to-right) satisfying the heap property: every parent is ≤ its children (min-heap) or ≥ its children (max-heap).

The key trick: a complete binary tree maps perfectly onto an array with no pointers.

```
Min-heap tree:           Array representation:
        1                [1, 3, 2, 7, 4, 5, 6]
       / \               index: 0  1  2  3  4  5  6
      3   2
     / \ / \
    7  4 5  6

For node at index i:
  left child:  2i + 1
  right child: 2i + 2
  parent:      (i - 1) // 2
```

No pointers needed — just arithmetic. This makes heaps extremely cache-friendly.

## Core operations

### Sift Up (used in insert)
Insert at the end of the array, then "bubble up" by comparing with parent and swapping until heap property is restored.
```
insert 0 → append → [1, 3, 2, 7, 4, 5, 6, 0]
  0 < parent(3) → swap → [1, 3, 2, 0, 4, 5, 6, 7]
  0 < parent(1) → swap → [0, 3, 2, 1, 4, 5, 6, 7]
  0 at root → done
```
**Cost**: O(log n) — height of the tree.

### Sift Down (used in extract-min and heapify)
Remove root (min), move last element to root, then "bubble down" by swapping with the smaller child until heap property is restored.
```
extract min → remove 0, put 7 at root → [7, 3, 2, 1, 4, 5, 6]
  7 > min_child(2) → swap → [2, 3, 7, 1, 4, 5, 6]
  7 > min_child(5) → swap → [2, 3, 5, 1, 4, 7, 6]
  7 is a leaf → done
```
**Cost**: O(log n).

## Why heapify is O(n), not O(n log n)

**The naive approach**: insert n elements one by one, each costing O(log n) → total O(n log n). This is how `heappush` × n works.

**Heapify (Floyd's algorithm)**: start from the last non-leaf node (index `n//2 - 1`) and sift down each node toward the root.

```python
def heapify(arr):
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):
        sift_down(arr, i, n)
```

**Why O(n)?** The key insight is that most nodes are near the leaves and have very little work to do. Sift-down cost = height of the subtree rooted at that node.

```
Level   Nodes at level   Max sift-down cost   Total work
  0     n/2 leaves       0                     0
  1     n/4 nodes        1                     n/4
  2     n/8 nodes        2                     n/4
  3     n/16 nodes       3                     3n/16
  ...
  h     1 root           h = log n             log n

Sum = n/4 · 1 + n/8 · 2 + n/16 · 3 + ...
    = n · Σ(k / 2^(k+1)) for k=1 to log n
    = n · 1              (geometric series with ratio < 1 sums to a constant)
    = O(n)
```

Inserting one-by-one is O(n log n) because you're doing expensive work (sift-up) for every node including those near the root. Heapify does cheap work for most nodes (leaves) and expensive work for only a few (near the root).

**Python:** `heapq.heapify(list)` uses Floyd's algorithm — O(n). Always prefer it over pushing n elements individually.

## d-ary heaps
A d-ary heap has d children per node instead of 2.

| | Binary heap (d=2) | d-ary heap (d>2) |
|---|---|---|
| Sift-down cost | O(2 log₂ n) comparisons | O(d log_d n) — shallower tree but more comparisons per level |
| Sift-up cost | O(log₂ n) | O(log_d n) — fewer levels |
| Cache behavior | Poor (sparse) | Better (denser, fewer levels = fewer cache misses) |
| Best for | General purpose | d=4 often used in practice; heaps backed by cache-line-sized nodes |

## Priority queue implementations compared

| Structure | Insert | Extract-min | Build from n | Notes |
|---|---|---|---|---|
| Binary heap | O(log n) | O(log n) | O(n) | Standard choice |
| Sorted array | O(n) | O(1) | O(n log n) | Good if inserts are rare |
| Fibonacci heap | O(1) amortized | O(log n) amortized | O(n) | Theoretically optimal; complex; rarely used in practice |
| Skip list | O(log n) expected | O(log n) | O(n log n) | Used in Redis sorted sets |

## Common interview angles
- "How is a binary heap stored?" — array, with index arithmetic for parent/child (no pointers)
- "Why is `heapq.heapify()` O(n) and not O(n log n)?" — Floyd's algorithm; most nodes are near leaves and have O(1) sift-down cost
- "What's the difference between sift-up and sift-down?" — sift-up for insert, sift-down for extract and heapify; sift-down is the more powerful operation
- "Is a heap a BST?" — No. A heap only enforces parent ≤ children; a BST enforces left < node < right. You cannot do O(log n) search in a heap.
- "When would you use a heap over a sorted array?" — when you have a mix of inserts and extract-mins; sorted array is better if inserts are rare and you just need a sorted order once

## Sources
*(none yet)*
