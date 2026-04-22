# Flatten Binary Tree to Linked List

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** DFS postorder, in-place right-threading
**Companies:** Amazon, Google, Apple, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given the root of a binary tree, flatten it **in-place** into a linked list using the `right` pointer. The list should be in **preorder** traversal order. After flattening, every `left` pointer should be `null`.

```
Input:              Output (as linked list, right pointers):
       1                1
      / \               \
     2   5               2
    / \   \               \
   3   4   6               3
                            \
                             4
                              \
                               5
                                \
                                 6
Preorder: 1→2→3→4→5→6  ✓
```

## Key Insight

After flattening, each node's `right` = the next node in preorder. For any node:
- Its next node = its left subtree root (if left exists), else its right subtree root
- After the left subtree is exhausted, the right subtree must continue

**Find the rightmost node of the left subtree → connect it to the current right subtree → move left subtree to right.**

## Approach 1: Iterative (in-place, O(1) extra space)

```python
def flatten(root) -> None:
    curr = root
    while curr:
        if curr.left:
            # Find rightmost node of left subtree
            rightmost = curr.left
            while rightmost.right:
                rightmost = rightmost.right
            # Connect it to curr's right subtree
            rightmost.right = curr.right
            # Move left subtree to right, clear left
            curr.right = curr.left
            curr.left  = None
        curr = curr.right
```

## Traced execution

```
Initial:
       1
      / \
     2   5
    / \   \
   3   4   6

Step 1: curr=1, left exists (2)
  rightmost of left subtree: 2→4 (rightmost right child of 2 is 4)
  4.right = 5 (curr's right)
  1.right = 2, 1.left = None
  Tree:
       1
        \
         2
        / \
       3   4
            \
             5
              \
               6
  curr = 1.right = 2

Step 2: curr=2, left exists (3)
  rightmost of left subtree: 3 (no right child)
  3.right = 4
  2.right = 3, 2.left = None
  Tree:
       1
        \
         2
          \
           3
            \
             4
              \
               5
                \
                 6
  curr = 3

Step 3: curr=3, no left → curr=4
Step 4: curr=4, no left → curr=5
Step 5: curr=5, no left → curr=6
Step 6: curr=6, no left → curr=None → done ✓
```

## Approach 2: Recursive (postorder reverse)

Process right subtree, then left subtree, then connect. Use a `prev` pointer tracking the last processed node.

```python
def flatten(root) -> None:
    prev = [None]
    def dfs(node):
        if not node: return
        dfs(node.right)
        dfs(node.left)
        node.right = prev[0]
        node.left  = None
        prev[0] = node
    dfs(root)
```

This processes nodes in reverse preorder (right → left → root), building the list from tail to head. Each node's `right` is set to the previously processed node.

## Complexity

Time: O(n) | Space: O(1) iterative / O(h) recursive

## Key insight (iterative)

The "rightmost of left subtree" step finds exactly where the left subtree ends in preorder — that's where the right subtree should be appended. This is the Morris traversal idea applied to flattening.

## Variants / follow-ups

- **Flatten to left-linked list (preorder left pointers):** same, swap left/right in the algorithm
- **Flatten in-order or postorder:** change the order of processing
- **Why preorder?** The problem specifies preorder as the natural way to traverse a tree-as-list (visit root first, then drill into subtrees)

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/trees/patterns/dfs-subtree-returns]]
