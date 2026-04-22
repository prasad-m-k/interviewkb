# Pattern: DFS with Subtree Returns (Postorder)

**Topic:** [[dsa/trees/Tree overview]]
**Related:** [[dsa/concepts/binary-tree]], [[dsa/patterns/tree-traversal]]

## What it solves

Problems where the answer at a node depends on **answers from its children first** — you need to bubble information up from leaves to root. This is postorder DFS (process children before the current node).

Signal: "height", "diameter", "balanced", "max path sum", "count nodes matching X"

---

## The Core Template

```python
def dfs(node):
    if not node:
        return BASE_VALUE       # what does a null node contribute?

    left  = dfs(node.left)
    right = dfs(node.right)

    # Combine left + right + node.val → compute this node's answer
    return SOMETHING_UP_TO_PARENT
```

The trick is deciding what to return. Common choices:

| Problem | Return | Accumulate globally |
|---|---|---|
| Height | `1 + max(left, right)` | — |
| Diameter | height: `1 + max(left, right)` | `max_diameter = max(left + right)` |
| Balanced | `(is_balanced, height)` | — |
| Max path sum | `node.val + max(0, left, right)` | `max_sum = max(left + right + node.val)` |
| Count nodes | `1 + left + right` | — |

---

## Problem 1: Height of Binary Tree

```
       3
      / \
     9  20
        / \
       15   7

Height = 3
```

```python
def maxDepth(root):
    if not root: return 0
    return 1 + max(maxDepth(root.left), maxDepth(root.right))
```

---

## Problem 2: Diameter of Binary Tree

Diameter = longest path between any two nodes (may not pass through root).

```
       1
      / \
     2   3
    / \
   4   5

Diameter = 3  (path: 4→2→5 or 4→2→1→3, both length 3 edges)
Actually: 4→2→5 = 2 edges, 4→2→1→3 = 3 edges → diameter = 3
```

**Key:** at every node, the longest path through it = `left_height + right_height`.

```python
def diameterOfBinaryTree(root):
    diameter = [0]

    def height(node):
        if not node: return 0
        lh = height(node.left)
        rh = height(node.right)
        diameter[0] = max(diameter[0], lh + rh)   # path through this node
        return 1 + max(lh, rh)                     # height returned up

    height(root)
    return diameter[0]
```

---

## Problem 3: Balanced Binary Tree

A tree is balanced if every node's left and right subtrees differ in height by at most 1.

**Key:** return `(is_balanced, height)` in one DFS — don't call height() separately (that's O(n²)).

```python
def isBalanced(root):
    def check(node):
        if not node: return True, 0
        l_ok, lh = check(node.left)
        r_ok, rh = check(node.right)
        ok = l_ok and r_ok and abs(lh - rh) <= 1
        return ok, 1 + max(lh, rh)

    balanced, _ = check(root)
    return balanced
```

**Alternative:** return `-1` to signal "unbalanced" instead of a tuple.

```python
def isBalanced(root):
    def check(node):
        if not node: return 0
        lh = check(node.left)
        rh = check(node.right)
        if lh == -1 or rh == -1 or abs(lh - rh) > 1:
            return -1                # propagate "unbalanced" upward
        return 1 + max(lh, rh)
    return check(root) != -1
```

---

## Problem 4: Maximum Path Sum (Hard)

Path = any sequence of nodes along edges; no node repeated. Value can be negative.

```
       -10
       /  \
      9    20
           / \
          15   7

Max path: 15 → 20 → 7 = 42
```

**Key insight:** the value a node can contribute upward to its parent is `node.val + max(0, left_gain, right_gain)` — you can only continue in one direction going up. But locally, a path can use both children: `left_gain + node.val + right_gain`.

```python
def maxPathSum(root):
    max_sum = [float('-inf')]

    def gain(node):
        if not node: return 0
        left_gain  = max(0, gain(node.left))   # ignore negative contributions
        right_gain = max(0, gain(node.right))
        # best path through this node (uses both sides)
        max_sum[0] = max(max_sum[0], node.val + left_gain + right_gain)
        # return the best single-arm contribution upward
        return node.val + max(left_gain, right_gain)

    gain(root)
    return max_sum[0]
```

---

## The Return-vs-Accumulate Pattern

For diameter and max path sum, the function returns one thing (height or single-arm gain) but accumulates the real answer in a separate variable. This is because:
- The **return value** = what the parent needs to extend its path
- The **global variable** = the best local answer (using both arms)

These are two different quantities. Always ask: "what does my parent need from me?" vs. "what's the best I can do locally?"

---

## Complexity

All variants: O(n) time, O(h) space (recursion stack)

---

## Problems using this pattern

- [[dsa/trees/problems/diameter-of-binary-tree]]
- [[dsa/trees/problems/binary-tree-maximum-path-sum]]
- [[dsa/trees/problems/path-sum]]
- [[dsa/trees/problems/count-good-nodes]]

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/patterns/tree-traversal]]
