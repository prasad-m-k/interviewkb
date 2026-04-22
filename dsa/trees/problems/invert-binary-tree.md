# Invert Binary Tree

**Difficulty:** Easy
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** DFS postorder (swap children after recursing)
**Companies:** Google, Amazon, Apple, Meta

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a binary tree, invert it (mirror it left-to-right) and return the root.

```
Input:               Output:
       4                    4
      / \                  / \
     2   7                7   2
    / \ / \              / \ / \
   1  3 6  9            9  6 3  1
```

## Approach

Swap left and right children at every node. Since you need both children before swapping, this is naturally postorder. However, preorder also works — the order doesn't matter because we're replacing, not reading.

## Solution (Python)

```python
def invertTree(root):
    if not root:
        return None
    root.left, root.right = invertTree(root.right), invertTree(root.left)
    return root
```

**Iterative (BFS):**
```python
from collections import deque

def invertTree(root):
    if not root: return root
    queue = deque([root])
    while queue:
        node = queue.popleft()
        node.left, node.right = node.right, node.left
        if node.left:  queue.append(node.left)
        if node.right: queue.append(node.right)
    return root
```

## Complexity

Time: O(n) | Space: O(h) recursive / O(w) iterative where w = max width

## Key insight

The swap `root.left, root.right = invertTree(root.right), invertTree(root.left)` works because Python evaluates the right-hand side fully before assignment. The recursive calls return the already-inverted subtrees, which are then cross-assigned.

## Variants / follow-ups

- **Check if two trees are mirrors of each other (Symmetric Tree):** → [[dsa/trees/problems/symmetric-tree]]
- **Iterative, without BFS?** Use a stack (DFS iteratively); same logic — push node, swap children, push new children

## Sources

- [[dsa/trees/Tree overview]]
