# Serialize and Deserialize Binary Tree

**Difficulty:** Hard
**Topic:** [[dsa/topics/data-structures]]
**Pattern:** [[dsa/patterns/tree-traversal]] (Preorder DFS)
**Companies:** Apple, Google, Amazon, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Design an algorithm to serialize a binary tree to a string and deserialize that string back to the original tree structure. There is no constraint on the format — you design it.

## Approach
**Key insight:** Preorder traversal (root → left → right) with explicit null markers is self-describing. When deserializing, you can reconstruct the tree by reading the same preorder sequence — each value creates a node; each null marker terminates a subtree.

**Serialize:** DFS preorder. At each node, output `val`. At each null, output `"N"`. Join with commas.

**Deserialize:** Split on commas into a value iterator. Recurse: read next value; if `"N"`, return None; otherwise create a node, then recursively assign left and right.

The iterator (`iter(...)`) is the key — it maintains state across recursive calls, advancing one element at a time regardless of recursion depth. No index tracking needed.

## Solution (Python)
```python
from collections import deque

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val; self.left = left; self.right = right

class Codec:
    def serialize(self, root: TreeNode) -> str:
        parts = []
        def dfs(node):
            if not node:
                parts.append('N')
                return
            parts.append(str(node.val))
            dfs(node.left)
            dfs(node.right)
        dfs(root)
        return ','.join(parts)

    def deserialize(self, data: str) -> TreeNode:
        vals = iter(data.split(','))
        def dfs():
            val = next(vals)
            if val == 'N':
                return None
            node = TreeNode(int(val))
            node.left = dfs()
            node.right = dfs()
            return node
        return dfs()
```

## Complexity
Time: O(n) serialize and deserialize | Space: O(n) for the string and recursion stack

## Key insight
Using `iter()` and `next()` eliminates the need for an index variable. The iterator is a shared cursor across all recursive calls — each `next(vals)` consumes one token from the global sequence. Without this pattern, you'd need to pass and return an index at every recursion level, which is messier.

## Variants / follow-ups
- **BFS serialization**: use level-order with null markers; produces a different string but is equally valid. Slightly easier to reason about, slightly more space (nulls at leaf level).
- **Serialize BST**: for a BST, preorder without null markers is sufficient to uniquely reconstruct — because BST properties determine where each value goes. Saves space but only works for BSTs.
- **Serialize N-ary tree**: same preorder approach; add a child count before each node's children.
- **Interviewers ask**: "Why preorder and not inorder?" → Inorder without null markers can't uniquely reconstruct a general binary tree. Preorder with nulls can. "What delimiter would you use in production?" → Avoid characters that appear in the data; use length-prefixed encoding for arbitrary node values.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
