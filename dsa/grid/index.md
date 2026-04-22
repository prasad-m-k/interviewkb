---
tags:
  - index
  - dsa
  - grid
  - matrix
  - algorithms
  - interview-prep
---

# Grid Problems — Index
Last updated: 2026-04-21

> Grid problems are 2D matrix problems. They range from simple traversal to complex graph problems. Master the direction vectors and the three core traversal strategies first — everything else is a variation.

## Overview
- [[Grid overview]] — Mental models, direction vectors, and the decision framework

## Concepts
- [[dsa/grid/concepts/direction-vectors]] — The foundation: 4-directional, 8-directional, diagonal-only
- [[dsa/grid/concepts/grid-as-graph]] — When to see a grid as a graph; BFS vs. DFS trade-offs
- [[dsa/grid/concepts/boundary-conditions]] — Row/col bounds, visited arrays, in-place marking

## Patterns
- [[dsa/grid/patterns/flood-fill-dfs]] — Island counting, region marking, connected components
- [[dsa/grid/patterns/bfs-shortest-path]] — Shortest path in unweighted grid; multi-source BFS
- [[dsa/grid/patterns/matrix-traversal]] — Spiral, diagonal, layer-by-layer
- [[dsa/grid/patterns/dynamic-programming-grid]] — Path counting, min-cost path, unique paths
- [[dsa/grid/patterns/simulation]] — Spiral order, rotate matrix, game of life
- [[dsa/grid/patterns/binary-search-2d]] — Search sorted matrix; flatten O(log RC) vs staircase O(R+C)

## Problems (Difficulty Ladder)
### Easy
- [[dsa/grid/problems/flood-fill]] — Paint bucket; basic DFS/BFS on grid
- [[dsa/grid/problems/number-of-islands]] — Count connected components
- [[dsa/grid/problems/spiral-matrix]] — Traverse in spiral order
- [[dsa/grid/problems/search-2d-matrix]] — LC 74; flatten binary search O(log RC); rows stitch together
- [[dsa/grid/problems/search-2d-matrix-ii]] — LC 240; staircase O(R+C); row+col sorted, rows don't stitch

### Medium
- [[dsa/grid/problems/01-matrix]] — Distance to nearest 0; multi-source BFS
- [[dsa/grid/problems/rotting-oranges]] — Multi-source BFS with time simulation
- [[dsa/grid/problems/unique-paths]] — DP on grid
- [[dsa/grid/problems/max-area-of-island]] — DFS + track max
- [[dsa/grid/problems/pacific-atlantic-waterflow]] — Multi-source BFS from borders
- [[dsa/grid/problems/walls-and-gates]] — Multi-source BFS
- [[dsa/grid/problems/word-search]] — DFS + backtracking on grid
- [[dsa/grid/problems/set-matrix-zeroes]] — In-place marking
- [[dsa/grid/problems/surrounded-regions]] — DFS from border cells

### Hard
- [[dsa/grid/problems/minimize-plant-infection-deaths]] — Greedy quarantine + multi-source BFS; minimize infected plants
- [[dsa/grid/problems/maze-solver]] — BFS shortest path with walls
- [[dsa/grid/problems/word-search-ii]] — Trie + DFS; multiple words
- [[dsa/grid/problems/shortest-path-in-binary-matrix]] — BFS with obstacle avoidance
- [[dsa/grid/problems/swim-in-rising-water]] — BFS/Dijkstra on weighted grid
- [[dsa/grid/problems/dungeon-game]] — Reverse DP

## Flashcards
- [[dsa/grid/flashcards/grid-anki]] — Anki deck: grid patterns and problems
