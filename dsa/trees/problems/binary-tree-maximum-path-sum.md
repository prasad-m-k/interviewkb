# Binary Tree Maximum Path Sum

**Difficulty:** Hard
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** [[dsa/trees/patterns/dfs-subtree-returns]] (gain = return value; global max = side effect)
**Companies:** Amazon, Google, Apple, Meta, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

A **path** in a binary tree is a sequence of nodes where each pair of adjacent nodes has an edge, and **no node appears twice**. The path does not need to pass through the root.

Find the **maximum sum** of any path. Node values can be negative.

```
        -10
        /  \
       9    20
           /  \
          15    7

Max path: 15 → 20 → 7 = 42
(The path -10 → 20 → 15 = 25 is worse; -10 → 9 is even worse)
```

```
    2
   / \
  -1  3

Max path: 2 → 3 = 5  (ignore -1; single node 3 = 3, root alone = 2)
```

## The Two Quantities

Every node makes a choice between two things that are **NOT the same**:

| Quantity | Formula | Used for |
|---|---|---|
| **Return to parent** | `node.val + max(0, left_gain, right_gain)` | Extend a path upward (only one arm) |
| **Local max path** | `left_gain + node.val + right_gain` | Best path through this node (both arms) |

You can only go up one arm (can't fork), but locally you can span both.

## Solution (Python)

```python
def maxPathSum(root) -> int:
    max_sum = [float('-inf')]

    def gain(node) -> int:
        """Returns the max gain this subtree can contribute to a path going upward."""
        if not node: return 0

        # max(0, ...) — don't take a subtree if it's negative
        left_gain  = max(0, gain(node.left))
        right_gain = max(0, gain(node.right))

        # Best path THROUGH this node (using both arms) — update global answer
        max_sum[0] = max(max_sum[0], node.val + left_gain + right_gain)

        # Return the best SINGLE arm (parent can only extend one direction)
        return node.val + max(left_gain, right_gain)

    gain(root)
    return max_sum[0]
```

## Traced execution

```
Tree:
    -10
    /  \
   9    20
       /  \
      15    7

gain(9):  left=0, right=0 → max_sum=max(-inf, 9+0+0)=9  → return 9
gain(15): left=0, right=0 → max_sum=max(9, 15)=15       → return 15
gain(7):  left=0, right=0 → max_sum=max(15, 7)=15       → return 7
gain(20): left_gain=max(0,15)=15, right_gain=max(0,7)=7
          max_sum = max(15, 20+15+7) = max(15,42) = 42  → return 20+max(15,7)=35
gain(-10):left_gain=max(0,9)=9, right_gain=max(0,35)=35
          max_sum = max(42, -10+9+35) = max(42,34) = 42 → return -10+max(9,35)=25

Answer: 42  ✓
```

## Why `max(0, gain)`?

If a subtree has negative total gain, **don't take it** — it only hurts the path sum. Clamping to 0 means "I can choose to not extend into this subtree."

```
     1
    /
  -999

gain(-999) = -999
left_gain for node 1 = max(0, -999) = 0 → don't include the left subtree
Max path = just node 1 = 1
```

## Common wrong answer

Forgetting `max(0, ...)`: if you pass negative gains upward, you contaminate parent calculations. Always clamp at 0.

## Complexity

Time: O(n) | Space: O(h)

## Why this is harder than Diameter

Diameter counts edges (always positive). Here values can be negative — you might skip entire subtrees, and a single node can be the best path. The `max(0, ...)` guard handles all three cases:
1. All positive → take both arms
2. One negative → take only the positive arm
3. Both negative → take neither arm (current node alone is the path)

## Variants / follow-ups

- **Path Sum (root to leaf):** → [[dsa/trees/problems/path-sum]] — simpler; no fork, always root-to-leaf
- **Max sum from any leaf to any other leaf:** similar but must include at least one arm on each side
- **Max sum path — return the path itself (not just sum):** track which nodes were included; harder

## Sources

- [[dsa/trees/patterns/dfs-subtree-returns]]
- [[dsa/trees/Tree overview]]
