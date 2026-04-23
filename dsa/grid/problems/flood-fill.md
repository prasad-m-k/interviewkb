# Flood Fill

**Difficulty:** Easy
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/grid/patterns/flood-fill-dfs]]
**Companies:** [[dsa/companies/apple]], [[dsa/companies/google]], [[dsa/companies/meta]]

## Problem

An image is represented as an `m x n` integer grid `image` where `image[i][j]` represents the pixel value. You are given three integers `sr`, `sc`, and `color`. Perform a **flood fill** starting from pixel `image[sr][sc]`.

To perform a flood fill, consider the starting pixel, plus any pixels connected **4-directionally** to the starting pixel of the same color as the starting pixel, plus any pixels connected 4-directionally to those pixels (and so on). Replace the color of all such pixels with `color`.

```
Input:  image = [[1,1,1],[1,1,0],[1,0,1]], sr=1, sc=1, color=2
Output: [[2,2,2],[2,2,0],[2,0,1]]
```

## Importance

Flood Fill is the **prototype algorithm** for all grid connectivity problems. Number of Islands, Max Area of Island, Word Search, and Rotten Oranges all use flood fill as their core. Mastering flood fill means you have the DFS/BFS grid template.

Real-world uses: paint bucket tool, terrain generation, connected-component labeling in image processing.

## Gotcha

**If the starting pixel already has the target color, return immediately.** Without this check, the DFS will infinitely recurse (each cell's neighbor has the new color, which equals the original color, so it never terminates).

## Solution (Python — DFS)

```python
def flood_fill(image: list[list[int]], sr: int, sc: int, color: int) -> list[list[int]]:
    original = image[sr][sc]
    if original == color:     # critical: avoid infinite loop
        return image

    rows, cols = len(image), len(image[0])

    def dfs(r: int, c: int) -> None:
        if r < 0 or r >= rows or c < 0 or c >= cols:
            return
        if image[r][c] != original:   # not same color as origin
            return
        image[r][c] = color           # paint
        dfs(r+1, c); dfs(r-1, c)
        dfs(r, c+1); dfs(r, c-1)

    dfs(sr, sc)
    return image
```

## Solution (BFS — iterative, avoids recursion limit)

```python
from collections import deque

def flood_fill_bfs(image, sr, sc, color):
    original = image[sr][sc]
    if original == color:
        return image

    rows, cols = len(image), len(image[0])
    queue = deque([(sr, sc)])
    image[sr][sc] = color

    while queue:
        r, c = queue.popleft()
        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and image[nr][nc] == original:
                image[nr][nc] = color
                queue.append((nr, nc))

    return image
```

## Complexity

| | Value |
|---|---|
| Time | O(rows × cols) — each cell visited at most once |
| Space | O(rows × cols) — recursion depth or queue size |

## Key Insight

This is the simplest form of flood fill because the stopping condition is just "different color." All more complex grid problems add additional stopping conditions (boundaries, walls, visited markers), but the skeleton is identical.

The **in-place color change** doubles as a "visited" marker — you never need a separate `visited` set because once a cell is painted, it no longer matches `original`.

## The DFS Grid Template (Generalized)

```python
def dfs_grid(grid, r, c, *context):
    # 1. Boundary check
    if r < 0 or r >= len(grid) or c < 0 or c >= len(grid[0]):
        return base_value
    # 2. Visited / stopping condition
    if <should_stop(grid[r][c], context)>:
        return base_value
    # 3. Mark visited (mutate or use visited set)
    grid[r][c] = VISITED_MARKER
    # 4. Recurse into 4 neighbors
    result = combine(
        dfs_grid(grid, r+1, c, *context),
        dfs_grid(grid, r-1, c, *context),
        dfs_grid(grid, r, c+1, *context),
        dfs_grid(grid, r, c-1, *context),
    )
    # 5. (Optional) Restore if needed
    return result
```

## Variants / Follow-ups

- "8-directional flood fill (including diagonals)?" → Add 4 diagonal directions to the direction list.
- "Flood fill without modifying the original image?" → Use a `visited` set instead of in-place mutation.
- "What if you want to count the filled cells?" → Track a counter during DFS.
- "Surrounded Regions (flip regions not touching border)?" → [[dsa/grid/problems/surrounded-regions]] — inverted flood fill starting from borders.

## Sources
- [[dsa/companies/apple]]
- [[dsa/companies/google]]
- [[dsa/grid/patterns/flood-fill-dfs]]
