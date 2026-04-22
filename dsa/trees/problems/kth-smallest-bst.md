# Kth Smallest Element in a BST

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** Iterative inorder traversal; stop at kth visit
**Companies:** Amazon, Google, Apple, Microsoft, Facebook

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a BST and an integer `k`, return the `k`th smallest value (1-indexed).

```
BST:
      3
     / \
    1   4
     \
      2

k=1 → 1
k=2 → 2
k=3 → 3
k=4 → 4
```

## Approach

Inorder traversal of a BST yields values in sorted ascending order. The kth node visited in inorder = kth smallest.

**Iterative** is preferred for interviews (avoids Python's recursion limit for large trees, and Apple explicitly asks for it).

## Solution (Python)

```python
def kthSmallest(root, k: int) -> int:
    stack, curr, count = [], root, 0
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        count += 1
        if count == k:
            return curr.val
        curr = curr.right
    # k > number of nodes in tree — problem guarantees this won't happen
```

**Recursive (simpler, but watch recursion depth):**
```python
def kthSmallest(root, k: int) -> int:
    result = [None]
    count  = [0]
    def inorder(node):
        if not node or result[0] is not None: return
        inorder(node.left)
        count[0] += 1
        if count[0] == k:
            result[0] = node.val; return
        inorder(node.right)
    inorder(root)
    return result[0]
```

## Traced execution (k=2)

```
BST:  3
     / \
    1   4
     \
      2

Iterative inorder:
  push 3 → push 1 → stack=[3,1], curr=None
  pop 1: count=1 (k=2, skip), curr=1.right=2
  push 2: stack=[3,2], curr=None
  pop 2: count=2 == k → return 2  ✓
```

## Complexity

Time: O(h + k) — traverse to leftmost leaf O(h), then k steps  
Space: O(h) — stack depth

## Key insight

You don't need to visit all n nodes — stop immediately at the kth. The iterative approach makes early termination clean and natural.

## Follow-up: What if the BST is frequently modified and kth smallest is queried often?

Augment each node with `left_count` = number of nodes in its left subtree.

```
Augmented BST:
      3 (left_count=2)
     / \
    1   4
(lc=0) (lc=0)
     \
      2
   (lc=0)

kthSmallest with augmented tree — O(log n):
k=2:
  root=3: left_count=2
    k ≤ left_count (2 ≤ 2) → go left
  root=1: left_count=0
    k > left_count+1 (2 > 1) → go right; k=2-0-1=1
  root=2: left_count=0
    k == left_count+1 (1==1) → return 2  ✓
```

This reduces per-query time from O(h+k) to O(log n) at the cost of O(n) space and O(log n) update time per insert/delete.

## Variants / follow-ups

- **Kth largest:** inorder right-first (descending order), stop at kth
- **Find all elements in range [lo, hi]:** DFS with BST pruning — skip left subtree if `node.val ≤ lo`, skip right if `node.val ≥ hi`
- **Count nodes in range:** same as above but count instead of collect

## Sources

- [[dsa/trees/concepts/bst]]
- [[dsa/trees/Tree overview]]
