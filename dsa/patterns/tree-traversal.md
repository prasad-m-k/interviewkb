# Tree Traversal

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/binary-tree]], [[dsa/concepts/deque]]

## What it solves
Processing or querying every node of a tree — finding paths, computing aggregates, serializing structure, detecting properties (balanced, valid BST), or finding relationships between nodes (LCA, diameter).

## Template / skeleton

**DFS — recursive (most common in interviews):**
```python
def dfs(node):
    if not node:
        return base_value        # define what "nothing" returns

    left_result = dfs(node.left)
    right_result = dfs(node.right)

    # Combine results and return
    return combine(node.val, left_result, right_result)
```

**DFS — returning two values (e.g., "is valid AND height"):**
```python
def check(node):
    if not node:
        return True, 0           # (is_valid, height)
    left_ok, lh = check(node.left)
    right_ok, rh = check(node.right)
    curr_ok = left_ok and right_ok and abs(lh - rh) <= 1
    return curr_ok, 1 + max(lh, rh)
```

**BFS — level-order with level grouping:**
```python
from collections import deque

def level_order(root):
    if not root: return []
    result = []
    queue = deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):      # snapshot: process exactly this level
            node = queue.popleft()
            level.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(level)
    return result
```

**DFS — iterative inorder (avoids recursion limit):**
```python
def inorder_iterative(root):
    stack, result, curr = [], [], root
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        result.append(curr.val)
        curr = curr.right
    return result
```

## Signal phrases
- "level order / level by level" → BFS with level snapshot
- "path from root to leaf", "path sum" → DFS preorder
- "bottom-up aggregation", "diameter", "height", "balanced" → DFS postorder (return values from children)
- "sorted output from BST" → inorder
- "serialize / reconstruct" → preorder with null markers
- "lowest common ancestor" → DFS; LCA is the node where p and q split

## Complexity
O(n) time for all traversals | O(h) space for DFS recursion (h = height), O(n) space for BFS queue

## Problems using this pattern
- [[dsa/problems/binary-tree-level-order-traversal]]
- [[dsa/problems/serialize-deserialize-binary-tree]]
- [[dsa/problems/lowest-common-ancestor]]

## Sources
- [[DSA overview]]
