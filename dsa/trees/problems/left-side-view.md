# Binary Tree Left Side View

**Difficulty:** Easy–Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** [[dsa/trees/patterns/tree-views]] (first node per level in BFS)
**Companies:** Amazon, Microsoft, Adobe

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a binary tree, imagine standing on the left side. Return the values of the nodes you can see, ordered from top to bottom.

```
         1            ←  see 1
        / \
       2   3          ←  see 2  (leftmost at this level)
        \
         5            ←  see 5  (only node at depth 2)

Left side view: [1, 2, 5]
```

```
         1            ←  see 1
        / \
       2   3          ←  see 2
          / \
         4   5        ←  see 4  (leftmost at depth 2, even though it's in right subtree)

Left side view: [1, 2, 4]
```

## Approach

BFS level-order; record the **first** node at each level.

## Solution (Python)

```python
from collections import deque

def leftSideView(root):
    if not root: return []
    result = []
    queue = deque([root])
    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == 0:                        # first node at this level
                result.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
    return result
```

**DFS alternative** (left-first, record on first visit to each depth):
```python
def leftSideView(root):
    result = []
    def dfs(node, depth):
        if not node: return
        if depth == len(result):              # first time at this depth
            result.append(node.val)
        dfs(node.left,  depth + 1)            # left before right!
        dfs(node.right, depth + 1)
    dfs(root, 0)
    return result
```

## Complexity

Time: O(n) | Space: O(w) BFS / O(h) DFS

## Key insight

The only difference from right side view:
- Left view → record `i == 0` (first per level) → go left-first in DFS
- Right view → record `i == level_size - 1` (last per level) → go right-first in DFS

## Tricky case

```
      1
       \
        2
         \
          3

Left side view: [1, 2, 3]   ← no left children, right arm is visible from left
```

The left view doesn't mean "only left subtree nodes." It means the leftmost node at each depth, regardless of which subtree it lives in.

## Sources

- [[dsa/trees/patterns/tree-views]]
- [[dsa/trees/problems/right-side-view]]
