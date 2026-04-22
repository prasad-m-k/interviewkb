# Binary Tree Level Order Traversal

**Difficulty:** Medium
**Topic:** [[dsa/topics/data-structures]]
**Pattern:** [[dsa/patterns/tree-traversal]] (BFS)
**Companies:** Apple, Amazon, Microsoft, Google

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a binary tree, return the level-order traversal of its nodes' values as a list of lists — each inner list contains all values at that depth.

Example: tree `[3,9,20,null,null,15,7]` → `[[3],[9,20],[15,7]]`

## Approach
BFS with a level-size snapshot. A plain BFS queue processes nodes in level-order naturally. To group by level, capture the queue size at the start of each iteration — that's exactly how many nodes belong to the current level. Process that many, then the remaining queue entries are the next level.

The `for _ in range(len(queue))` inner loop is the key pattern — it snapshots the level boundary without any extra data structure.

## Solution (Python)
```python
from collections import deque
from typing import List, Optional

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val; self.left = left; self.right = right

def levelOrder(root: Optional[TreeNode]) -> List[List[int]]:
    if not root:
        return []
    result = []
    queue = deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):      # process exactly this level
            node = queue.popleft()
            level.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(level)
    return result
```

## Complexity
Time: O(n) | Space: O(n) — the queue holds at most one full level; widest level of a complete binary tree has n/2 nodes

## Key insight
`len(queue)` is evaluated once before the inner loop begins — it captures the exact number of nodes at the current level. Nodes added to the queue during this loop (the next level's children) don't affect `range(len(queue))` since `range` was already computed.

## Variants / follow-ups
- **Level Order II**: return levels bottom-up → reverse the result list at the end.
- **Zigzag Level Order**: alternate left-to-right and right-to-left per level → use a flag; reverse `level` list on odd-numbered levels.
- **Right Side View**: return the last element of each level → `level[-1]`.
- **Average of Levels**: return mean of each level → `sum(level) / len(level)`.
- **Maximum Width of Binary Tree**: level width = rightmost index − leftmost index + 1; number nodes within each level using `(parent_index * 2 + offset)`.
- **Interviewers ask**: "How would you print the tree level by level without knowing the height in advance?" → The BFS approach handles this naturally.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
