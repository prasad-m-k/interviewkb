# Symmetric Tree

**Difficulty:** Easy
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** DFS with paired recursion
**Companies:** Amazon, Apple, Microsoft, Bloomberg

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a binary tree, check whether it is a mirror of itself (symmetric around its center).

```
Symmetric:              Not symmetric:
       1                       1
      / \                     / \
     2   2                   2   2
    / \ / \                   \   \
   3  4 4  3                   3   3
```

## Approach

The tree is symmetric if the left subtree is a mirror of the right subtree. Two trees are mirrors if:
1. Their roots have equal values
2. The left subtree of one mirrors the **right** subtree of the other (and vice versa)

Recurse with **two pointers** — one going left-left/right-right (same direction), the other going left-right/right-left (mirrored direction).

## Solution (Python)

```python
def isSymmetric(root):
    def isMirror(left, right):
        if not left and not right: return True    # both null → symmetric
        if not left or not right:  return False   # one null → not symmetric
        return (left.val == right.val and
                isMirror(left.left,  right.right) and   # outer pair
                isMirror(left.right, right.left))        # inner pair

    return isMirror(root.left, root.right)
```

**Iterative (queue of pairs):**
```python
from collections import deque

def isSymmetric(root):
    queue = deque([(root.left, root.right)])
    while queue:
        left, right = queue.popleft()
        if not left and not right: continue
        if not left or not right:  return False
        if left.val != right.val:  return False
        queue.append((left.left,  right.right))   # outer
        queue.append((left.right, right.left))    # inner
    return True
```

## Complexity

Time: O(n) | Space: O(h)

## Key insight

The pairing `(left.left, right.right)` and `(left.right, right.left)` captures the mirror relationship. Visualize folding the tree in half — outer nodes pair with outer, inner with inner.

```
Level 2:  [3, 4]  vs  [4, 3]   ← indices mirror each other
Pairs to check: (3,3) and (4,4) ✓
```

## Variants / follow-ups

- **Check if two separate trees are mirrors:** call `isMirror(root1, root2)` directly
- **Invert one tree, then check equality:** works but O(n) extra space; the paired recursion is cleaner

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/trees/problems/invert-binary-tree]]
