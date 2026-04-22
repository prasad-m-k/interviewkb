# Top View of Binary Tree

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** [[dsa/trees/patterns/tree-views]] (BFS + horizontal distance, first per HD)
**Companies:** Amazon, Microsoft, Flipkart, Adobe

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given a binary tree, print the nodes visible when the tree is viewed from the top (directly above), left to right.

```
           1
         /   \
        2     3
       / \   / \
      4   5 6   7

HD:  -2 -1  0  1  2
      4   2  1  3  7
              ↑
        (5 and 6 also at HD=0 and HD=1 but hidden behind 1 and 3)

Top view: [4, 2, 1, 3, 7]
```

**Trickier example:**
```
         1
        / \
       2   3
        \
         4
          \
           5
            \
             6

HD:  -1  0  1
      2   1  3
          but 4 has HD=0, 5 has HD=1, 6 has HD=2

Top view: [2, 1, 3, 6]
(4 is hidden behind 1; 5 is hidden behind 3; 6 is new HD=2)
```

## Approach

BFS from root, carrying each node's horizontal distance. For each HD, record only the **first** node seen (topmost in BFS order = closest to root).

```python
HD rules:
  root        → HD = 0
  node.left   → parent.HD - 1
  node.right  → parent.HD + 1
```

## Solution (Python)

```python
from collections import deque

def topView(root):
    if not root: return []
    hd_map = {}                           # HD → first node value (topmost)
    queue = deque([(root, 0)])            # (node, horizontal_distance)

    while queue:
        node, hd = queue.popleft()
        if hd not in hd_map:             # first time seeing this HD
            hd_map[hd] = node.val
        if node.left:  queue.append((node.left,  hd - 1))
        if node.right: queue.append((node.right, hd + 1))

    return [hd_map[hd] for hd in sorted(hd_map)]
```

## Traced execution (first tree above)

```
Queue: [(1,0)]
  Pop (1,0): hd_map={0:1}; push (2,-1), (3,1)
  Pop (2,-1): hd_map={0:1,-1:2}; push (4,-2), (5,0)
  Pop (3,1):  hd_map={0:1,-1:2,1:3}; push (6,0), (7,2)
  Pop (4,-2): hd_map={0:1,-1:2,1:3,-2:4}; no children
  Pop (5,0):  hd=0 already in map → skip
  Pop (6,0):  hd=0 already in map → skip (both 5 and 6 hidden)
  Pop (7,2):  hd_map={..., 2:7}

Sorted by HD: [-2,-1,0,1,2] → [4,2,1,3,7]  ✓
```

## Complexity

Time: O(n log n) — BFS is O(n); sorting HDs is O(k log k) where k ≤ n  
Space: O(n)

## Key insight

Top view uses **horizontal distance**, not depth. BFS guarantees nodes are processed in top-to-bottom order, so the first node at any HD is always the topmost (visible) one.

The HD can range from `-(n-1)` to `+(n-1)` in a skewed tree. A sorted dictionary or `sorted(hd_map)` at the end is cleaner than tracking min/max HD during traversal.

## Comparison: Top vs. Bottom view

```python
# Top view:    if hd not in hd_map: hd_map[hd] = node.val    (first wins)
# Bottom view: hd_map[hd] = node.val                          (last wins)
```

## Variants / follow-ups

- **Bottom View:** → [[dsa/trees/problems/bottom-view]] — always overwrite; deepest per HD wins
- **Vertical Order Traversal (LC 987):** group all nodes by HD, sort within each HD by depth then value; harder variant
- **What if two nodes at same HD and same depth?** → Vertical order traversal tiebreaks by value; top/bottom view typically says "any one is acceptable"

## Sources

- [[dsa/trees/patterns/tree-views]]
- [[dsa/trees/Tree overview]]
