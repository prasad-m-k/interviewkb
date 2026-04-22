# Number of Islands

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/bfs-dfs]]
**Companies:** Amazon, Google, Meta, Apple, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a 2D binary grid of `'1'` (land) and `'0'` (water), return the number of islands. An island is surrounded by water and is formed by connecting adjacent land cells horizontally or vertically.

## Approach
This is a connected components counting problem on an implicit graph. Each `'1'` cell is a node; edges connect horizontally and vertically adjacent `'1'` cells.

**DFS flood fill:**
1. Scan every cell in the grid.
2. When a `'1'` is found, increment the island count and run DFS/BFS from that cell, marking every reachable `'1'` as `'0'` (visited) to avoid double-counting.
3. When DFS returns, all cells of that island are marked; the next `'1'` found belongs to a new island.

**BFS variant:** replace the recursive DFS with a queue; useful if the grid is large (avoids Python recursion depth limit).

## Solution (Python)
```python
from collections import deque
from typing import List

# DFS (recursive)
def numIslands_dfs(grid: List[List[str]]) -> int:
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'   # mark visited in-place
        dfs(r + 1, c); dfs(r - 1, c)
        dfs(r, c + 1); dfs(r, c - 1)

    count = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                dfs(r, c)
    return count

# BFS (avoids recursion depth issues on large grids)
def numIslands(grid: List[List[str]]) -> int:
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                grid[r][c] = '0'
                queue = deque([(r, c)])
                while queue:
                    row, col = queue.popleft()
                    for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
                        nr, nc = row + dr, col + dc
                        if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == '1':
                            grid[nr][nc] = '0'
                            queue.append((nr, nc))
    return count
```

## Complexity
Time: O(m × n) | Space: O(min(m, n)) for BFS queue, O(m × n) for DFS call stack worst case

## Key insight
Marking cells in-place (`grid[r][c] = '0'`) avoids a separate `visited` set. This is acceptable when the interviewer allows input mutation. If not, use a `visited: set` of `(r, c)` tuples.

## Variants / follow-ups
- **Max Area of Island**: same flood fill; track count of cells per island.
- **Number of Distinct Islands**: same flood fill; serialize the DFS path shape as a string; use a set to count distinct shapes.
- **Pacific Atlantic Water Flow**: multi-source BFS from both ocean edges; find cells reachable from both.
- **Surrounded Regions**: flood fill from the border, then flip all unvisited `'O'` cells.
- **Interviewers ask**: "What if the grid is too large to fit in memory?" → Distributed graph traversal; split by rows, process in chunks, merge border components.

## MLOps connection
Cluster detection in distributed systems topology, or finding connected components in a data lineage graph. Also maps to anomaly clustering: given a binary matrix of (time, feature) anomaly flags, count distinct anomaly clusters.

## Sources
- [[DSA overview]]
