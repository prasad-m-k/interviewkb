# Binary Tree Right Side View

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** [[dsa/trees/patterns/tree-views]] (last node per level in BFS)
**Companies:** Amazon, Apple, Facebook, Google

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a binary tree, imagine standing on the right side. Return the values of the nodes you can see, ordered from top to bottom.

```
         1            ←  see 1
        / \
       2   3          ←  see 3
        \
         5            ←  see 5

Right side view: [1, 3, 5]
```

```
         1            ←  see 1
        / \
       2   3          ←  see 3
      / \
     4   5            ←  see 5  (3 has no children, so 4 and 5 are both visible; rightmost = 5)

Right side view: [1, 3, 5]
```

## Approach

BFS level-order. At each level, the last node dequeued is the rightmost — the one visible from the right.

## Solution (Python)

```python
from collections import deque

def rightSideView(root):
    if not root: return []
    result = []
    queue = deque([root])
    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == level_size - 1:           # last node at this level
                result.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
    return result
```

**DFS alternative** (right-first traversal):
```python
def rightSideView(root):
    result = []
    def dfs(node, depth):
        if not node: return
        if depth == len(result):              # first visit to this depth
            result.append(node.val)           # right-first → rightmost seen first
        dfs(node.right, depth + 1)            # right before left!
        dfs(node.left,  depth + 1)
    dfs(root, 0)
    return result
```

## Complexity

Time: O(n) | Space: O(w) BFS where w = max level width, or O(h) DFS

## Key insight

Right side view = last element per level (BFS) = right-subtree-first DFS where you record on the first visit to each depth.

The tricky case: a node's left child extends deeper than the right child. You still see the left child from the right if the right subtree doesn't reach that depth.

```
      1
     /
    2
   /
  3

Right side view: [1, 2, 3]  (no right children — left arm is visible from right)
```

## Variants / follow-ups

- **Left Side View:** → [[dsa/trees/problems/left-side-view]] — same, but record first node per level (BFS) or left-first DFS
- **Average of Levels (LC 637):** `sum(level) / len(level)` for each level
- **Maximum Width (LC 662):** index nodes within each level; width = max_idx - min_idx + 1

## Sources

- [[dsa/trees/patterns/tree-views]]
- [[dsa/trees/Tree overview]]
