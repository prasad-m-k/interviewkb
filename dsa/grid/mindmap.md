# Grid Problems

## The Core Mental Model
- A grid is a graph — cells are nodes, edges connect adjacent cells
- Every grid problem maps to a known graph algorithm
- Ask: connectivity? → DFS. Shortest path? → BFS. Count/min/max paths? → DP. Simulate rules? → Simulation

## Direction Vectors

### 4-Directional (standard)
- `DIRS = [(0,1),(0,-1),(1,0),(-1,0)]`
- right / left / down / up
- Use for: islands, maze, BFS shortest path, most grid traversal

### 8-Directional (with diagonals)
- Add `(1,1),(1,-1),(-1,1),(-1,-1)`
- Use for: word search all directions, Game of Life, chess piece movement, binary matrix shortest path

### Special Moves
- Knight: `(±1,±2)` and `(±2,±1)` — 8 L-shaped moves
- Spiral direction cycle: right → down → left → up

## Boundary Conditions
- Always check `0 <= r < ROWS and 0 <= c < COLS` BEFORE `grid[r][c]`
- Handle empty grid: `if not grid or not grid[0]: return`
- Mark visited WHEN ENQUEUING (BFS), not when dequeuing

## Three Visited Strategies
- Separate array → grid unchanged; O(R×C) space
- In-place mark → modify `grid[r][c]`; O(1) extra; flood fill standard
- Backtrack (unmark after DFS) → word search; different paths need different state

## DFS Patterns

### Flood Fill / Connected Components
- Mark cells as visited in-place (grid[r][c] = '0')
- Count islands, max area, paint regions
- [[dsa/grid/patterns/flood-fill-dfs]]
- Problems: Number of Islands, Max Area of Island, Flood Fill, Surrounded Regions

### DFS + Backtracking
- Mark cell, recurse, UNMARK after
- Different paths can reuse cells
- [[dsa/grid/problems/word-search]]

## BFS Patterns

### Single-Source Shortest Path
- BFS from start; first arrival = shortest distance
- Queue: `(r, c, dist)` or separate dist array
- [[dsa/grid/patterns/bfs-shortest-path]]
- Problems: Maze Solver, Shortest Path in Binary Matrix

### Multi-Source BFS (key pattern)
- Add ALL sources to queue at step 0
- Each cell gets distance to nearest source
- [[dsa/grid/problems/01-matrix]]
- Problems: 01 Matrix, Rotting Oranges, Walls and Gates, Pacific Atlantic

### BFS with State
- State = (r, c, extra_info) e.g., keys held, walls removed
- Visited = set of full states

### Dijkstra (weighted grid)
- Replace queue with min-heap
- Cost = sum or max of edge weights
- [[dsa/grid/problems/swim-in-rising-water]]

## Matrix Traversal Patterns

### Spiral
- Pointers: top, bottom, left, right
- Four moves per layer; shrink inward
- Guard last row/col: `if top <= bottom` and `if left <= right`
- [[dsa/grid/problems/spiral-matrix]]

### Diagonal
- Anti-diagonal: all cells where `r + c == d`
- Main diagonal: all cells where `r - c == constant`

### Rotation
- 90° clockwise: transpose then reverse each row
- 90° CCW: transpose then reverse each column

## Dynamic Programming Patterns

### Counting Paths
- `dp[r][c] = dp[r-1][c] + dp[r][c-1]`
- Base: first row and column = 1
- [[dsa/grid/problems/unique-paths]]

### Min/Max Path Sum
- `dp[r][c] = min(dp[r-1][c], dp[r][c-1]) + cost`
- [[dsa/grid/patterns/dynamic-programming-grid]]

### Reverse DP (Dungeon Game)
- When forward state is too complex, go backward from target
- `dp[r][c]` = min HP needed to enter this cell and survive
- [[dsa/grid/problems/swim-in-rising-water]]

## Simulation
- Game of Life: bit trick (encode old + new state in one value)
- Rotting Oranges: multi-source BFS; track time per BFS level
- [[dsa/grid/patterns/simulation]]

## Problem Difficulty Ladder

### Easy Entry Points
- Flood Fill → simplest DFS on grid
- Number of Islands → flood fill + count
- Spiral Matrix → pointer traversal

### Medium — Master These
- 01 Matrix → multi-source BFS
- Rotting Oranges → multi-source BFS + time
- Word Search → DFS + backtrack
- Unique Paths → grid DP
- Pacific Atlantic → reversed multi-source BFS

### Hard — Advanced Patterns
- Word Search II → Trie + DFS
- Swim in Rising Water → Dijkstra on grid
- Dungeon Game → reverse DP
- Shortest Path with k obstacle removals → BFS with state

## Decision Framework
- Count regions / connected components → DFS (flood fill)
- Shortest path (unweighted) → BFS
- Shortest path (weighted) → Dijkstra
- Distance to all sources simultaneously → Multi-source BFS
- Count or sum all paths → DP
- Simulate rules step by step → Simulation
- Search for pattern → DFS + backtracking
- Traverse in specific order → Pointer-based (spiral, diagonal)
