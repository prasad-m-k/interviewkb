# Number of Islands

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/flood-fill-dfs]]
**Topic:** [[Grid overview]]
**Companies:** Amazon, Google, Microsoft, Facebook

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a 2D grid of '1's (land) and '0's (water), count the number of islands. An island is surrounded by water and is formed by connecting adjacent land cells horizontally or vertically.

## Approach
Classic flood fill: iterate every cell. When you find an unvisited '1', increment count and DFS to mark the entire island as visited (change '1' → '0').

## Solution (Python)
```python
def numIslands(grid):
    if not grid:
        return 0

    ROWS, COLS = len(grid), len(grid[0])
    count = 0

    def dfs(r, c):
        if not (0 <= r < ROWS and 0 <= c < COLS) or grid[r][c] != '1':
            return
        grid[r][c] = '0'   # mark visited in-place
        dfs(r+1, c); dfs(r-1, c)
        dfs(r, c+1); dfs(r, c-1)

    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1

    return count
```

## Complexity
Time: O(R×C) | Space: O(R×C) recursion stack

## Key insight
Flood fill from every unvisited '1'. Marking cells '0' prevents re-visiting. Each call to `dfs` from the outer loop is a separate island.

## Variants / follow-ups
- **Max area of island**: return size from DFS; track max — [[dsa/grid/problems/max-area-of-island]]
- **Number of distinct islands**: hash the DFS traversal path (relative directions) to detect distinct shapes
- **Count islands in a stream of additions**: use Union-Find; efficiently merge newly added cells with neighbors

## Sources
- [[dsa/grid/patterns/flood-fill-dfs]]
