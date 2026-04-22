---
tags:
  - index
  - dsa
  - algorithms
  - data-structures
  - interview-prep
---

# DSA Index
Last updated: 2026-04-21

## Overview
- [[DSA overview]] — DSA for senior MLOps engineers; top 10 problems, core competency map, study strategy

## Topics
- [[dsa/topics/data-structures]] — HashMap, Heap, Deque, DLL, Graph — with interview context
- [[dsa/topics/algorithms]] — BFS/DFS, Topological Sort, DP, Greedy, Sliding Window, Two Pointers

## Concepts
- [[dsa/concepts/hash-map]] — O(1) lookup; complement pattern; combined with DLL for LRU
- [[dsa/concepts/heap]] — Min/max heap; Top-K; merge sorted; priority queue patterns
- [[dsa/concepts/graph]] — Adjacency list; BFS/DFS templates; cycle detection (3-color DFS)
- [[dsa/concepts/dynamic-programming]] — Bottom-up tabulation; state/recurrence/base-case framework
- [[dsa/concepts/deque]] — O(1) both ends; monotonic deque for sliding window max/min
- [[dsa/concepts/binary-tree]] — Traversal orders, BST, height, LCA, serialization patterns
- [[dsa/concepts/stack]] — Four stack modes: monotonic, matching, simulation, expression eval

## Patterns
- [[dsa/patterns/monotonic-stack]] — Decreasing (next greater) / increasing (next smaller); O(n) boundary queries
- [[dsa/patterns/sliding-window]] — Fixed and variable window; monotonic deque variant
- [[dsa/patterns/bfs-dfs]] — BFS (shortest path, level order); DFS (connectivity, flood fill)
- [[dsa/patterns/topological-sort]] — Kahn's BFS; DFS post-order; cycle detection in DAGs
- [[dsa/patterns/two-heaps]] — Max-heap (lower half) + min-heap (upper half) for streaming median
- [[dsa/patterns/dynamic-programming]] — 1D and 2D DP templates; interview checklist
- [[dsa/patterns/tree-traversal]] — DFS (pre/in/post), BFS level-order, LCA, serialization templates
- [[dsa/patterns/backtracking]] — Choose/explore/unchoose; constraint sets; N-Queens family

## Problems
- [[dsa/problems/lru-cache]] — Medium; HashMap + DLL; O(1) get/put; Amazon, Google, Meta
- [[dsa/problems/course-schedule-ii]] — Medium; Topological Sort; cycle detection; Google, Amazon
- [[dsa/problems/sliding-window-maximum]] — Hard; Monotonic Deque; O(n); Google, Amazon, Meta
- [[dsa/problems/find-median-from-data-stream]] — Hard; Two Heaps; O(log n) add, O(1) query; Google, Meta
- [[dsa/problems/top-k-frequent-elements]] — Medium; Heap/Bucket Sort; O(n) optimal; Amazon, Google
- [[dsa/problems/number-of-islands]] — Medium; BFS/DFS flood fill; Amazon, Google, Meta, Apple
- [[dsa/problems/merge-k-sorted-lists]] — Hard; Min Heap; O(N log k); Amazon, Google, Meta
- [[dsa/problems/task-scheduler]] — Medium; Greedy + Heap; pipeline scheduling; Facebook, Google
- [[dsa/problems/word-break]] — Medium; Dynamic Programming; O(n²); Google, Amazon, Meta
- [[dsa/problems/design-hit-counter]] — Medium; Queue/Circular Buffer; rate limiting; Google, Dropbox

### Stack
- [[dsa/problems/daily-temperatures]] — Medium; decreasing monotonic stack; next warmer day; Amazon, Google
- [[dsa/problems/largest-rectangle-histogram]] — Hard; increasing monotonic stack; left+right boundary; Amazon, Google
- [[dsa/problems/valid-parentheses]] — Easy; matching stack; nesting order; Amazon, Google, Meta
- [[dsa/problems/asteroid-collision]] — Medium; simulation stack; right-meets-left resolution; Amazon, Google

### Backtracking
- [[dsa/problems/n-queens]] — Hard; backtracking + O(1) constraint sets; row/col/diagonal; Google, Amazon
- [[dsa/problems/sudoku-solver]] — Hard; backtracking + row/col/box sets; most-constrained heuristic; Google
- [[dsa/problems/combination-sum]] — Medium; sorted input + start index avoids duplicates; Amazon, Google
- [[dsa/problems/generate-parentheses]] — Medium; open/close counters; Catalan number; Google, Bloomberg

### Apple-specific
- [[dsa/problems/merge-intervals]] — Medium; Sort + Greedy; overlapping windows; Apple, Google, Amazon
- [[dsa/problems/serialize-deserialize-binary-tree]] — Hard; Preorder DFS + null markers; Apple, Google
- [[dsa/problems/binary-tree-level-order-traversal]] — Medium; BFS level snapshot; Apple, Amazon
- [[dsa/problems/trapping-rain-water]] — Hard; Two Pointers; O(1) space; Apple, Google, Meta
- [[dsa/problems/product-of-array-except-self]] — Medium; Prefix/suffix pass; no division; Apple, Amazon
- [[dsa/problems/lowest-common-ancestor]] — Medium; DFS postorder; tree split test; Apple, Google, Meta
- [[dsa/problems/maximum-subarray]] — Medium; Kadane's Algorithm; Apple, Amazon, Google

## Binary Trees (full section)
- [[dsa/trees/index]] — Full tree KB: overview, BST, views, path problems, FAANG tricky problems

### Trees — Easy
- [[dsa/trees/problems/invert-binary-tree]] — Swap children; LC 226; Google, Apple
- [[dsa/trees/problems/symmetric-tree]] — Paired recursion mirror check; LC 101
- [[dsa/trees/problems/diameter-of-binary-tree]] — lh+rh at every node; foundation of max path sum; LC 543

### Trees — Medium
- [[dsa/trees/problems/right-side-view]] — Last per level BFS; LC 199; Amazon, Apple, Facebook
- [[dsa/trees/problems/left-side-view]] — First per level BFS; variant of LC 199
- [[dsa/trees/problems/top-view]] — BFS + horizontal distance, first per HD
- [[dsa/trees/problems/bottom-view]] — BFS + horizontal distance, last per HD
- [[dsa/trees/problems/path-sum]] — Root-to-leaf path I & II; backtracking; LC 112/113
- [[dsa/trees/problems/validate-bst]] — Bounds propagation; LC 98; Amazon, Apple
- [[dsa/trees/problems/kth-smallest-bst]] — Iterative inorder; LC 230; Amazon, Apple
- [[dsa/trees/problems/construct-from-preorder-inorder]] — Divide & conquer + hashmap; LC 105; Google, Apple
- [[dsa/trees/problems/flatten-to-linked-list]] — In-place right-threading; O(1) space; LC 114
- [[dsa/trees/problems/count-good-nodes]] — Top-down carry max-so-far; LC 1448

### Trees — Hard
- [[dsa/trees/problems/binary-tree-maximum-path-sum]] — Gain up + accumulate globally; LC 124; Google, Apple

## Flashcards
- [[dsa/flashcards/dsa-mlops-top10]] — Obsidian callout flashcards for all 10 problems
- [[dsa/flashcards/dsa-mlops-top10-anki]] — Anki TSV import file (HTML formatted)

## Companies
- [[dsa/companies/apple]] — Heavy on trees, intervals, arrays; clean code emphasis; 4–5 on-site rounds

## Sources
*(none yet)*
