# Count Good Nodes in Binary Tree

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** DFS top-down, carry max-so-far on path
**Companies:** Google, Amazon, Apple, Facebook

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

A node `X` is **good** if no node on the path from root to `X` has a value greater than `X.val`. Count the number of good nodes.

```
         3
        / \
       1   4
      /   / \
     3   1   5

Good nodes: 3 (root), 3 (left-left, path=[3,1,3] max=3), 4, 5
Node 1 (left child of 3): path=[3,1], max=3 > 1 → NOT good
Node 1 (left child of 4): path=[3,4,1], max=4 > 1 → NOT good

Answer: 4
```

## Approach

DFS top-down, carrying `max_so_far` (the maximum value seen from root to current node). A node is good if `node.val >= max_so_far`.

```python
def goodNodes(root) -> int:
    def dfs(node, max_so_far: int) -> int:
        if not node: return 0
        is_good = 1 if node.val >= max_so_far else 0
        new_max = max(max_so_far, node.val)
        return is_good + dfs(node.left, new_max) + dfs(node.right, new_max)

    return dfs(root, float('-inf'))
```

## Traced execution

```
dfs(3, -inf): 3 >= -inf ✓, is_good=1, new_max=3
  dfs(1, 3):  1 >= 3? No, is_good=0, new_max=3
    dfs(3, 3): 3 >= 3 ✓, is_good=1, new_max=3 → return 1
    → return 0+1 = 1
  dfs(4, 3):  4 >= 3 ✓, is_good=1, new_max=4
    dfs(1, 4): 1 >= 4? No → return 0
    dfs(5, 4): 5 >= 4 ✓, is_good=1 → return 1
    → return 1+0+1 = 2
  → return 1+1+2 = 4  ✓
```

## Complexity

Time: O(n) | Space: O(h)

## Key insight

This is a "carry information downward" pattern — the current node's goodness depends on what we've seen from the root. The running maximum is the state passed down, not returned up.

Compare to diameter / max path sum (which return information upward). The direction of information flow tells you which DFS template to use.

## Variants / follow-ups

- **Count nodes where node.val > all ancestors:** same; just change `>=` to `>`
- **Count nodes where node.val equals max on path:** same structure; change condition
- **Pseudo-palindromic Paths in a BST (LC 1457):** carry a bitmask of digit parities down — same top-down carry pattern

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/trees/patterns/dfs-subtree-returns]]
