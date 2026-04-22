# Diameter of Binary Tree

**Difficulty:** Easy (but pattern is reused in Hard problems)
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** [[dsa/trees/patterns/dfs-subtree-returns]] (bottom-up: return height, accumulate diameter)
**Companies:** Google, Amazon, Apple, Facebook

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

The **diameter** of a binary tree is the length of the longest path between any two nodes (measured in number of edges). The path may or may not pass through the root.

```
         1
        / \
       2   3
      / \
     4   5

Diameter = 3  (path: 4 → 2 → 1 → 3, or 5 → 2 → 1 → 3 — both 3 edges)
Note: 4 → 2 → 5 is only 2 edges; 1→3 adds a third.
```

```
    1
   /
  2
 /
3

Diameter = 2  (path: 3 → 2 → 1, passes through root)
```

## Approach

At every node, the longest path **through** that node = `left_height + right_height` (combining both arms). Track the global maximum across all nodes.

The function returns height (for the parent's use) but accumulates diameter as a side effect.

```
At node 2:  left_height=1 (node 4), right_height=1 (node 5)
             path through 2 = 1+1 = 2 edges
At node 1:  left_height=2 (subtree rooted at 2), right_height=1 (node 3)
             path through 1 = 2+1 = 3 edges  ← max
```

## Solution (Python)

```python
def diameterOfBinaryTree(root) -> int:
    diameter = [0]

    def height(node) -> int:
        if not node: return 0
        lh = height(node.left)
        rh = height(node.right)
        diameter[0] = max(diameter[0], lh + rh)   # path through this node
        return 1 + max(lh, rh)                     # height returned to parent

    height(root)
    return diameter[0]
```

## Traced execution

```
height(4): lh=0, rh=0 → diameter=max(0,0+0)=0 → return 1
height(5): lh=0, rh=0 → diameter=max(0,0+0)=0 → return 1
height(2): lh=1, rh=1 → diameter=max(0,1+1)=2 → return 2
height(3): lh=0, rh=0 → diameter=max(2,0+0)=2 → return 1
height(1): lh=2, rh=1 → diameter=max(2,2+1)=3 → return 3

Answer: 3 ✓
```

## Complexity

Time: O(n) | Space: O(h)

## Key insight

The diameter is NOT always `left_height(root) + right_height(root)`. It might be entirely within a subtree, not passing through the root at all. That's why you check `lh + rh` at **every** node, not just the root.

```
Counter-example:
       1
      /
     2
    / \
   3   4
  /     \
 5       6

Diameter = 4 (5→3→2→4→6), entirely in the left subtree of 1.
Checking only at root gives: left_height=4, right_height=0 → 4. ✓ (happens to work here)
But in general: MUST check every node.
```

## Variants / follow-ups

- **Diameter of N-ary tree:** same logic, take max over all children heights; sum the top-2 for the path
- **Longest path with same values (LC 687):** only add to path if child value equals current — same pattern
- **This is the foundation of max path sum:** see [[dsa/trees/problems/binary-tree-maximum-path-sum]]

## Sources

- [[dsa/trees/patterns/dfs-subtree-returns]]
- [[dsa/trees/Tree overview]]
