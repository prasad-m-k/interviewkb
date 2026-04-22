# Binary Tree — Overview

## The One Mental Model You Need

Every binary tree problem is one of three things:

```
1. MEASURE the tree          — height, diameter, node count, balance check
2. FIND something in it      — path sum, LCA, view from a direction
3. TRANSFORM it              — invert, flatten, serialize, construct
```

Pick the right traversal, and the implementation follows.

---

## Anatomy Reference

```
             1          ← root (depth 0)
           /   \
          2     3       ← depth 1
         / \     \
        4   5     6     ← depth 2 (leaves: 4, 5, 6)

Height  = 3  (edges from root to deepest leaf, 1-indexed: 3 levels)
Depth of node 5 = 2
Diameter = longest path between any two nodes = 4→2→1→3→6 = 4 edges
```

---

## Traversal Orders — When to Use Each

```
         A
        / \
       B   C
      / \
     D   E
```

| Order | Sequence | Code shape | Use when |
|---|---|---|---|
| **Preorder** (root→L→R) | A B D E C | process node, then recurse | serialization; construct copy; root-to-leaf paths |
| **Inorder** (L→root→R) | D B E A C | recurse left, process, recurse right | BST sorted output; validate BST |
| **Postorder** (L→R→root) | D E B C A | recurse both, then process | needs children's answers first: height, diameter, balanced, max path sum |
| **Level-order** (BFS) | A B C D E | queue, level snapshot | views; level sums; zigzag; nearest node |

---

## The Two DFS Templates — Memorise Both

### Template 1: Top-down (pass info down)
Use when the parent's value affects child processing (path sums, BST bounds).

```python
def dfs(node, carry):
    if not node:
        return base_case
    # use carry + node.val to compute local answer
    left  = dfs(node.left,  new_carry)
    right = dfs(node.right, new_carry)
    return combine(left, right)
```

### Template 2: Bottom-up (return info up)  ← most common in hard problems
Use when you need children's results to compute the parent's answer.

```python
def dfs(node):
    if not node:
        return base_value     # define what None returns

    left  = dfs(node.left)
    right = dfs(node.right)
    # now use left + right + node.val
    return something_up
```

**Rule of thumb:** if you're computing height/diameter/balance/max-path, use bottom-up. If you're building a path or checking BST bounds, use top-down.

---

## Decision Framework

```
Binary tree problem?
  │
  ├─ BFS / level properties
  │   ├─ Level grouping             → [[patterns/bfs-level-order]] (level order)
  │   ├─ View from a direction      → [[patterns/tree-views]] (left/right/top/bottom)
  │   └─ Zigzag / right-side view   → BFS + direction flag or last-per-level
  │
  ├─ DFS — needs children's result first (postorder)
  │   ├─ Height / depth             → [[problems/diameter-of-binary-tree]]
  │   ├─ Balance check              → return (is_balanced, height)
  │   ├─ Max path sum (any path)    → [[problems/binary-tree-maximum-path-sum]]
  │   └─ LCA                        → [[dsa/problems/lowest-common-ancestor]]
  │
  ├─ DFS — path from root (preorder / top-down)
  │   ├─ Path sum exists?           → [[problems/path-sum]]
  │   ├─ All root-to-leaf paths     → [[problems/path-sum-ii]]
  │   └─ Count good nodes           → [[problems/count-good-nodes]]
  │
  ├─ BST-specific
  │   ├─ Validate                   → [[trees/concepts/bst]]
  │   ├─ Kth smallest               → [[problems/kth-smallest-bst]]
  │   └─ LCA in BST                 → use BST property (no full DFS needed)
  │
  └─ Structural transformation
      ├─ Invert tree                 → [[problems/invert-binary-tree]]
      ├─ Flatten to linked list      → [[problems/flatten-to-linked-list]]
      └─ Construct from traversals   → [[problems/construct-from-preorder-inorder]]
```

---

## Complexity Reference

| Operation | Time | Space (recursion) |
|---|---|---|
| Any DFS traversal | O(n) | O(h) — O(log n) balanced, O(n) skewed |
| BFS / level-order | O(n) | O(w) — O(n) worst case (widest level ≈ n/2) |
| BST search/insert | O(h) | O(h) |

`h` = height. Always state which case (balanced vs. skewed) in interviews.

---

## The Most Common Mistakes

1. **Forgetting the base case for `None`** — every recursive function must handle `not node`
2. **Confusing height and depth** — height is bottom-up (height of leaf = 1); depth is top-down (depth of root = 0)
3. **Using a global mutable variable carelessly** — prefer returning the answer up through the call stack; use `nonlocal` intentionally
4. **DFS when BFS is simpler** — anything "per level" is almost always cleaner with BFS
5. **Reconstructing a BST from inorder only** — inorder alone doesn't uniquely define a BST structure (you need preorder or postorder too)
6. **Off-by-one in diameter** — diameter = `left_height + right_height` (heights, not depths); don't add 1

---

## Node Definition (reuse everywhere)

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

## Sources
- [[dsa/trees/index]]
- [[dsa/concepts/binary-tree]]
