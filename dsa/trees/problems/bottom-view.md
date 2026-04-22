# Bottom View of Binary Tree

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** [[dsa/trees/patterns/tree-views]] (BFS + horizontal distance, last per HD)
**Companies:** Amazon, Microsoft, Flipkart

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given a binary tree, print the nodes visible when the tree is viewed from the bottom (looking upward), left to right.

```
           1
         /   \
        2     3
       / \   / \
      4   5 6   7

HD:  -2 -1  0  1  2
      4   2  5  6→ 3  7
              ↑
        (5 and 6 are at HD=0 and HD=1 in level 2 — they're deeper than 1 and 3)

Wait: 5 has HD=0 (right child of 2 which has HD=-1 → -1+1=0)
      6 has HD=0 (left child of 3 which has HD=1 → 1-1=0)
      BFS order at level 2: 4(-2), 5(0), 6(0), 7(2)
      6 is processed after 5 so hd_map[0] = 6 (last overwrites)

      7 has HD=2 (right child of 3)

Bottom view: [4, 2, 5, 6, 3, 7]
```

Wait — let me recheck:
```
HD assignments:
  1 → 0
  2 → -1,  3 → 1
  4 → -2,  5 → 0,  6 → 0,  7 → 2

BFS processes: 1(0), 2(-1), 3(1), 4(-2), 5(0), 6(0), 7(2)

hd_map after each step:
  1: {0:1}
  2: {0:1, -1:2}
  3: {0:1, -1:2, 1:3}
  4: {0:1, -1:2, 1:3, -2:4}
  5: {0:5, -1:2, 1:3, -2:4}         ← 5 overwrites 1 at HD=0
  6: {0:6, -1:2, 1:3, -2:4}         ← 6 overwrites 5 at HD=0
  7: {0:6, -1:2, 1:3, -2:4, 2:7}

Sorted by HD: [-2,-1,0,1,2] → [4, 2, 6, 3, 7]
```

**Bottom view: [4, 2, 6, 3, 7]**

(Note: at HD=0, both 5 and 6 are tied in depth. BFS processes left-to-right so 6 wins. Some problems say "either is acceptable" for ties.)

## Approach

BFS from root, carrying horizontal distance. **Always overwrite** `hd_map[hd]` — the last node processed at any HD is the deepest visible one (since BFS processes top-to-bottom).

## Solution (Python)

```python
from collections import deque

def bottomView(root):
    if not root: return []
    hd_map = {}                           # HD → last node value seen (deepest wins)
    queue = deque([(root, 0)])

    while queue:
        node, hd = queue.popleft()
        hd_map[hd] = node.val            # always overwrite
        if node.left:  queue.append((node.left,  hd - 1))
        if node.right: queue.append((node.right, hd + 1))

    return [hd_map[hd] for hd in sorted(hd_map)]
```

## Complexity

Time: O(n log n) | Space: O(n)

## Key insight

One-line difference from top view:

```python
# Top view:    if hd not in hd_map: hd_map[hd] = node.val   (first = topmost wins)
# Bottom view: hd_map[hd] = node.val                         (last = deepest wins)
```

BFS visits nodes level by level (top to bottom), so overwriting gives the deepest node at each HD.

## Tricky tie case

```
         1
        / \
       2   3
        \ /
         4  ← HD=0, depth=2
         5  ← HD=0? No.
```

Actually:
```
         1            (HD=0)
        / \
       2   3          (HD=-1, HD=1)
        \ /
        [impossible — node can only have one parent]
```

Real tie example:
```
         1
        / \
       2   3
      /     \
     4       5        All unique HDs here

         1
        / \
       2   3
        \ /
     Can't happen in a tree
```

True tie: when two nodes at the same depth both map to the same HD. BFS left-to-right ordering determines the winner (rightmost wins since it's processed last).

## Variants / follow-ups

- **Top View:** → [[dsa/trees/problems/top-view]]
- **Vertical Order Traversal (LC 987):** return ALL nodes at each HD, sorted by depth then value — harder

## Sources

- [[dsa/trees/patterns/tree-views]]
- [[dsa/trees/Tree overview]]
