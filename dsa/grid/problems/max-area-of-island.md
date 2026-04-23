# Max Area of Island

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/grid/patterns/flood-fill-dfs]]
**Companies:** [[dsa/companies/apple]], [[dsa/companies/google]], [[dsa/companies/meta]]

## Problem

Given a 2D binary grid of `0`s (water) and `1`s (land), return the maximum area of an island. An island is a group of `1`s connected 4-directionally (not diagonally). If there is no island, return 0.

```
Input:
[[0,0,1,0,0,0,0,1,0,0,0,0,0],
 [0,0,0,0,0,0,0,1,1,1,0,0,0],
 [0,1,1,0,1,0,0,0,0,0,0,0,0],
 [0,1,0,0,1,1,0,0,1,0,1,0,0],
 [0,1,0,0,1,1,0,0,1,1,1,0,0],
 [0,0,0,0,0,0,0,0,0,0,1,0,0],
 [0,0,0,0,0,0,0,1,1,1,0,0,0],
 [0,0,0,0,0,0,0,1,1,0,0,0,0]]

Output: 6
```

## Relationship to Number of Islands

Max Area of Island is [[dsa/problems/number-of-islands]] with an extra tracking requirement:
- Number of Islands: count connected components (DFS/BFS, return count)
- Max Area: measure the size of each component, return the maximum

If you know Number of Islands cold, Max Area of Island is a 2-line modification.

## Approach

DFS flood fill. For each unvisited land cell:
1. DFS into the component.
2. Mark visited cells as 0 (or with a separate visited set).
3. Count cells during the DFS; return the count.
4. Update global maximum.

## Solution (Python — DFS, in-place mutation)

```python
def max_area_of_island(grid: list[list[int]]) -> int:
    rows, cols = len(grid), len(grid[0])

    def dfs(r: int, c: int) -> int:
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] == 0:
            return 0
        grid[r][c] = 0  # mark visited (mutate in-place)
        return 1 + dfs(r+1,c) + dfs(r-1,c) + dfs(r,c+1) + dfs(r,c-1)

    return max(
        dfs(r, c)
        for r in range(rows)
        for c in range(cols)
        if grid[r][c] == 1
    )
```

## Solution (BFS variant — for interviewers who ask "can you avoid recursion?")

```python
from collections import deque

def max_area_bfs(grid: list[list[int]]) -> int:
    rows, cols = len(grid), len(grid[0])
    max_area = 0

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] != 1:
                continue
            # BFS from this cell
            queue = deque([(r, c)])
            grid[r][c] = 0
            area = 0
            while queue:
                cr, cc = queue.popleft()
                area += 1
                for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
                    nr, nc = cr + dr, cc + dc
                    if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                        grid[nr][nc] = 0
                        queue.append((nr, nc))
            max_area = max(max_area, area)

    return max_area
```

## Solution (without mutating the grid — uses visited set)

Use when the interviewer says the input is read-only:
```python
def max_area_no_mutate(grid: list[list[int]]) -> int:
    rows, cols = len(grid), len(grid[0])
    visited = set()

    def dfs(r, c):
        if (r, c) in visited or r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] == 0:
            return 0
        visited.add((r, c))
        return 1 + dfs(r+1,c) + dfs(r-1,c) + dfs(r,c+1) + dfs(r,c-1)

    return max((dfs(r, c) for r in range(rows) for c in range(cols)), default=0)
```

## Complexity

| | Value |
|---|---|
| Time | O(rows × cols) — each cell visited at most once |
| Space | O(rows × cols) — recursion depth in worst case (all land) |

## Key Insight

The in-place mutation trick (`grid[r][c] = 0`) is the most elegant way to mark visited cells — no extra data structure needed. But always ask the interviewer: "Is it OK to modify the input?" If not, use the `visited` set variant.

The DFS formulation `return 1 + dfs(...)` is compact but can hit Python's recursion limit for large grids (~1000×1000). For large inputs, use the BFS variant or `sys.setrecursionlimit`.

## Variants / Follow-ups

- "Return the number of islands instead of max area" → return a count, not a max. → [[dsa/problems/number-of-islands]]
- "Return coordinates of the largest island" → track cells during DFS.
- "What if the grid wraps around (toroidal topology)?" → Adjust boundary conditions to use modular arithmetic.
- "What if the grid is too large to fit in memory?" → External algorithm: process in row chunks, track partial islands at chunk boundaries, merge in a second pass (similar to map-reduce connected components).
- "Perimeter of the largest island?" → Each land cell adds 4; subtract 2 for each shared edge. Count during DFS.

## Sources
- [[dsa/companies/apple]]
- [[dsa/companies/google]]
