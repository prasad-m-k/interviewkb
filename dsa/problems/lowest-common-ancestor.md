# Lowest Common Ancestor of a Binary Tree

**Difficulty:** Medium
**Topic:** [[dsa/topics/data-structures]]
**Pattern:** [[dsa/patterns/tree-traversal]] (DFS postorder)
**Companies:** Apple, Amazon, Google, Meta, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a binary tree and two nodes `p` and `q`, find their lowest common ancestor (LCA). The LCA is the deepest node that is an ancestor of both `p` and `q`. A node is considered an ancestor of itself.

## Approach
**Recursive DFS (postorder):** for each node, ask: "does p or q exist in my left subtree? In my right subtree?"

The LCA is the first node where `p` and `q` are on *different sides* (one in left, one in right), or the node *is* `p` or `q` and the other is in its subtree.

**Return value logic:**
- If `node` is null → return None
- If `node == p` or `node == q` → return `node` (this is the shallowest root of a subtree containing this target)
- Recurse left and right
- If both left and right return non-null → `node` is the LCA; return `node`
- Otherwise → return whichever side is non-null (pass the found target up to the parent)

## Solution (Python)
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val; self.left = left; self.right = right

def lowestCommonAncestor(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    if not root or root is p or root is q:
        return root

    left  = lowestCommonAncestor(root.left,  p, q)
    right = lowestCommonAncestor(root.right, p, q)

    if left and right:
        return root      # p and q are in different subtrees
    return left or right # both in same subtree; propagate the found node up
```

## Complexity
Time: O(n) — visits every node once in the worst case | Space: O(h) recursion stack where h = height

## Key insight
The early return `if not root or root is p or root is q: return root` does two things at once: handles the null base case AND short-circuits when we find a target (we don't need to search deeper — if `p` is a descendant of `q`, returning `q` here is correct because `q` is the LCA).

**The split test:** `if left and right: return root` — when both sides returned something non-null, the current node is the meeting point. This is the only node in the tree where `p` and `q` exist on opposite sides.

## Variants / follow-ups
- **LCA of BST**: exploit BST ordering — if both `p.val` and `q.val < node.val`, go left; if both `>`, go right; else current node is LCA. O(h) time, O(1) space with an iterative version.
- **LCA with parent pointers**: walk both nodes to the root; store visited nodes in a set; first repeated node is the LCA. O(h) time, O(h) space.
- **LCA for multiple nodes**: generalize by returning a node if *any* target is found; the LCA is where the total count of found targets reaches the total number of targets.
- **Interviewers ask**: "What if p or q might not be in the tree?" → Add a boolean return value alongside the node; only declare LCA if both targets were confirmed found.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
