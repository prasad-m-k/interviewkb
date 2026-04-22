# Construct Binary Tree from Preorder and Inorder Traversal

**Difficulty:** Medium
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** Divide and conquer; preorder gives root, inorder gives split
**Companies:** Amazon, Google, Apple, Microsoft, Meta

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given `preorder` and `inorder` traversal arrays of a binary tree (all values unique), reconstruct the tree.

```
preorder = [3, 9, 20, 15, 7]
inorder  = [9, 3, 15, 20, 7]

Reconstructed tree:
       3
      / \
     9  20
        / \
       15   7
```

## The Core Insight

```
Preorder: [root | left subtree | right subtree]
Inorder:  [left subtree | root | right subtree]

Step 1: preorder[0] = 3 is the root
Step 2: find 3 in inorder → index 1
        inorder left  of index 1: [9]        → left subtree  (size = 1)
        inorder right of index 1: [15, 20, 7] → right subtree (size = 3)
Step 3: preorder[1 : 1+1] = [9]        → preorder of left subtree
        preorder[2 : ]    = [20, 15, 7] → preorder of right subtree
Step 4: recurse on each side
```

## Solution (Python)

```python
def buildTree(preorder: list[int], inorder: list[int]):
    if not preorder: return None

    # Build index map for O(1) root lookup in inorder
    inorder_idx = {val: i for i, val in enumerate(inorder)}

    def build(pre_start, pre_end, in_start, in_end):
        if pre_start > pre_end: return None

        root_val = preorder[pre_start]
        root = TreeNode(root_val)

        mid = inorder_idx[root_val]           # root position in inorder
        left_size = mid - in_start            # number of nodes in left subtree

        root.left  = build(pre_start + 1,
                           pre_start + left_size,
                           in_start,
                           mid - 1)
        root.right = build(pre_start + left_size + 1,
                           pre_end,
                           mid + 1,
                           in_end)
        return root

    return build(0, len(preorder) - 1, 0, len(inorder) - 1)
```

## Traced execution

```
preorder = [3, 9, 20, 15, 7]
inorder  = [9, 3, 15, 20, 7]
inorder_idx = {9:0, 3:1, 15:2, 20:3, 7:4}

build(0, 4, 0, 4):
  root_val=3, mid=1, left_size=1
  left  = build(1, 1, 0, 0):     root_val=9, mid=0, left_size=0
            left  = build(2,1,...) → None (pre_start>pre_end)
            right = build(2,1,...) → None
            return TreeNode(9)
  right = build(2, 4, 2, 4):     root_val=20, mid=3, left_size=1
            left  = build(3,3,2,2): root=15 → leaf
            right = build(4,4,4,4): root=7  → leaf
            return TreeNode(20)
  return TreeNode(3)
```

## Complexity

Time: O(n) with hashmap (O(n²) with linear search in inorder)  
Space: O(n) hashmap + O(h) recursion stack

## Key insight

`left_size = mid - in_start` maps the inorder split directly to preorder slices. The hashmap makes the `mid` lookup O(1) — without it, the algorithm is O(n²).

```
Left subtree preorder slice:  [pre_start+1 .. pre_start+left_size]
Right subtree preorder slice: [pre_start+left_size+1 .. pre_end]
```

## Sibling: Construct from Postorder + Inorder (LC 106)

```
postorder: [left | right | root]
Root = postorder[-1] instead of preorder[0]. Same logic.

def buildTree_post(inorder, postorder):
    inorder_idx = {val: i for i, val in enumerate(inorder)}
    def build(in_start, in_end, post_start, post_end):
        if in_start > in_end: return None
        root_val = postorder[post_end]
        root = TreeNode(root_val)
        mid = inorder_idx[root_val]
        left_size = mid - in_start
        root.left  = build(in_start, mid-1, post_start, post_start+left_size-1)
        root.right = build(mid+1, in_end, post_start+left_size, post_end-1)
        return root
    return build(0, len(inorder)-1, 0, len(postorder)-1)
```

## Variants / follow-ups

- **Construct from preorder only (BST):** possible for BST — each value determines left/right placement
- **Serialize/Deserialize (LC 297):** → [[dsa/problems/serialize-deserialize-binary-tree]] — preorder with null markers; doesn't need inorder
- **Can you reconstruct from inorder alone?** No — many trees share the same inorder sequence (e.g., all right-skewed trees with values [1,2,3])

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/problems/serialize-deserialize-binary-tree]]
