# Shortest Path in Binary Matrix

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/bfs-shortest-path]] (8-directional BFS)
**Topic:** [[Grid overview]]
**Companies:** Google, Amazon, Facebook

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an n×n binary matrix, find the shortest clear path from the top-left (0,0) to the bottom-right (n-1,n-1). A clear path only goes through cells with value 0. You can move in 8 directions (including diagonals). Return the length of the shortest path, or -1 if none.

## Approach
BFS, but 8-directional instead of 4-directional. This is the main twist.

## Solution (Python)
```python
from collections import deque

def shortestPathBinaryMatrix(grid):
    n = len(grid)

    if grid[0][0] == 1 or grid[n-1][n-1] == 1:
        return -1

    if n == 1:
        return 1 if grid[0][0] == 0 else -1

    # 8 directions (including diagonals)
    DIRS = [(0,1),(0,-1),(1,0),(-1,0),(1,1),(1,-1),(-1,1),(-1,-1)]

    queue = deque([(0, 0, 1)])   # (row, col, path_length)
    grid[0][0] = 1               # mark visited in-place

    while queue:
        r, c, length = queue.popleft()

        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if not (0 <= nr < n and 0 <= nc < n):
                continue
            if grid[nr][nc] != 0:
                continue

            if nr == n-1 and nc == n-1:
                return length + 1

            grid[nr][nc] = 1  # mark visited
            queue.append((nr, nc, length + 1))

    return -1
```

## Complexity
Time: O(n²) | Space: O(n²)

## Key insight
The only difference from standard maze BFS is the 8-directional DIRS array. Also: check if the destination was reached immediately after dequeuing, or check immediately before enqueuing (the second is slightly more efficient — catches the answer one step earlier).

**Note:** modifying the grid to mark visited is fine here. If the problem says "don't modify input", use a separate visited set.

## Variants / follow-ups
- What if there are k obstacles you can remove? → add `remaining_removals` to BFS state

## Sources
- [[dsa/grid/patterns/bfs-shortest-path]]
