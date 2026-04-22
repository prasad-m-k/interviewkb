# 01 Matrix (Distance to Nearest Zero)

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/bfs-shortest-path]] (multi-source BFS)
**Topic:** [[Grid overview]]
**Companies:** Facebook, Amazon, Google

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an m×n binary matrix of 0s and 1s, return a matrix of the same size where each cell contains the distance to the nearest 0.

## Approach
**Multi-source BFS**: initialize queue with ALL cells that contain 0 (at distance 0). BFS expands outward. Each 1-cell is assigned distance when first reached (guaranteed to be the shortest).

**Wrong approach:** BFS/DFS from each 1-cell separately — O((R×C)²) time.

## Solution (Python)
```python
from collections import deque

def updateMatrix(mat):
    ROWS, COLS = len(mat), len(mat[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    dist = [[float('inf')] * COLS for _ in range(ROWS)]
    queue = deque()

    # All zeros are sources at distance 0
    for r in range(ROWS):
        for c in range(COLS):
            if mat[r][c] == 0:
                dist[r][c] = 0
                queue.append((r, c))

    while queue:
        r, c = queue.popleft()
        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and dist[nr][nc] > dist[r][c] + 1):
                dist[nr][nc] = dist[r][c] + 1
                queue.append((nr, nc))

    return dist
```

## Complexity
Time: O(R×C) | Space: O(R×C)

## Key insight
Multi-source BFS is the key. All zeros start at distance 0 and propagate outward simultaneously. The first time BFS reaches a cell is via the shortest path. No cell is ever processed twice because its distance can only decrease (and we only add to queue when we find a shorter distance).

## Variants / follow-ups
- **Walls and Gates**: exact same pattern — sources are gates (value 0), walls block expansion (value -1) → [[dsa/grid/problems/walls-and-gates]]
- **Rotting oranges**: sources are rotten oranges; track time (BFS level) → [[dsa/grid/problems/rotting-oranges]]

## Sources
- [[dsa/grid/patterns/bfs-shortest-path]]
