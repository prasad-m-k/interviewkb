# Validate Binary Search Tree

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** DFS top-down with bounds; or iterative inorder
**Companies:** Amazon, Google, Apple, Microsoft, Bloomberg

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a binary tree, determine if it is a valid BST. A valid BST has the property: for every node, **all** values in the left subtree are strictly less than the node's value, and **all** values in the right subtree are strictly greater.

```
Valid BST:             Invalid BST (subtly wrong):
      5                       5
     / \                     / \
    1   4                   4   6
       / \                 / \
      3   6               3   7  ← 7 > 5 but is in left subtree of 5 ✗
```

## Common Mistake

Only checking `node.left.val < node.val < node.right.val` at each node. This fails the example above — `7 < 4` is false locally, but `7 < 5` is the global constraint that's violated.

**The constraint is global, not local.**

## Approach 1: Bounds Propagation (Preferred)

Pass `(lo, hi)` bounds down the tree. Every node must satisfy `lo < node.val < hi`.

- Going left: upper bound tightens to `node.val`
- Going right: lower bound tightens to `node.val`

```python
def isValidBST(root, lo=float('-inf'), hi=float('inf')) -> bool:
    if not root: return True
    if not (lo < root.val < hi): return False
    return (isValidBST(root.left,  lo,       root.val) and
            isValidBST(root.right, root.val, hi))
```

### Traced on the invalid example

```
isValidBST(5, -inf, inf):  5 in (-inf,inf) ✓
  isValidBST(4, -inf, 5):  4 in (-inf,5) ✓
    isValidBST(3, -inf, 4):  3 in (-inf,4) ✓ → True
    isValidBST(7, 4, 5):     7 in (4,5)? NO → False ✗
  → returns False
```

## Approach 2: Iterative Inorder (catch non-increasing sequence)

Inorder traversal of a valid BST yields a strictly increasing sequence. If any value ≤ previous, it's invalid.

```python
def isValidBST(root) -> bool:
    stack, prev, curr = [], float('-inf'), root
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        if curr.val <= prev:    # must be strictly greater
            return False
        prev = curr.val
        curr = curr.right
    return True
```

## Complexity

Time: O(n) | Space: O(h)

## Key insight

The BST invariant is **not** just parent-child; it's ancestor-descendant. Every node must lie within the valid range set by all its ancestors. The bounds propagation approach makes this explicit.

## Edge cases

- Duplicate values: `left.val < node.val < right.val` must be **strict** (no equals)
- `INT_MIN` / `INT_MAX` nodes: use `float('-inf')` and `float('inf')` as initial bounds, not integer constants

## Variants / follow-ups

- **Recover BST (LC 99):** exactly two nodes are swapped; find and fix them using inorder (O(1) space with Morris traversal)
- **Convert sorted array to BST:** always pick mid as root → guaranteed balanced
- **BST to Doubly Linked List:** inorder traversal, thread pointers

## Sources

- [[dsa/trees/concepts/bst]]
- [[dsa/trees/Tree overview]]
