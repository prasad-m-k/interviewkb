---
tags:
  - index
  - dsa
  - trees
  - binary-tree
  - bst
  - algorithms
  - interview-prep
---

# Binary Trees — Index
Last updated: 2026-04-21

> Every binary tree problem is one of three things: **measure** the tree (height, diameter, balance), **find** something in it (path sum, LCA, views), or **transform** it (invert, flatten, construct). Pick the right traversal and the implementation follows.

## Overview
- [[dsa/trees/Tree overview]] — Decision framework, two DFS templates, traversal order guide, ASCII anatomy

## Concepts
- [[dsa/trees/concepts/bst]] — BST invariant, search/insert/delete, validate, LCA, kth smallest

## Patterns
- [[dsa/trees/patterns/dfs-subtree-returns]] — Bottom-up DFS: height, diameter, balanced, max path sum
- [[dsa/trees/patterns/tree-views]] — Left/right/top/bottom view using BFS + horizontal distance

## Problems (Difficulty Ladder)

### Easy
- [[dsa/trees/problems/invert-binary-tree]] — Swap children recursively or iteratively; LC 226
- [[dsa/trees/problems/symmetric-tree]] — Mirror check with paired recursion; LC 101
- [[dsa/trees/problems/diameter-of-binary-tree]] — lh+rh at every node; foundation of max path sum; LC 543

### Medium
- [[dsa/trees/problems/right-side-view]] — Last node per level in BFS; LC 199
- [[dsa/trees/problems/left-side-view]] — First node per level in BFS; LC (variant of 199)
- [[dsa/trees/problems/top-view]] — First node per horizontal distance; BFS + HD
- [[dsa/trees/problems/bottom-view]] — Last node per horizontal distance; BFS + HD
- [[dsa/trees/problems/path-sum]] — Root-to-leaf path sum I & II; DFS subtract-as-you-go; LC 112/113
- [[dsa/trees/problems/validate-bst]] — Bounds propagation or iterative inorder; LC 98
- [[dsa/trees/problems/kth-smallest-bst]] — Iterative inorder, stop at kth; LC 230
- [[dsa/trees/problems/construct-from-preorder-inorder]] — Divide & conquer; O(n) with hashmap; LC 105
- [[dsa/trees/problems/flatten-to-linked-list]] — In-place right-threading; LC 114
- [[dsa/trees/problems/count-good-nodes]] — Top-down DFS carrying max-so-far; LC 1448

### Hard
- [[dsa/trees/problems/binary-tree-maximum-path-sum]] — Return gain upward, accumulate globally; LC 124

## Cross-references (also in main dsa/problems/)
- [[dsa/problems/binary-tree-level-order-traversal]] — BFS with level snapshot; LC 102
- [[dsa/problems/serialize-deserialize-binary-tree]] — Preorder + null markers; LC 297
- [[dsa/problems/lowest-common-ancestor]] — DFS postorder; LC 236
