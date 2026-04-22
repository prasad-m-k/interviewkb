# Pattern: BFS Shortest Path on Grid

**Topic:** [[Grid overview]]
**Related:** [[dsa/grid/concepts/grid-as-graph]], [[dsa/grid/concepts/boundary-conditions]]

## What it solves
- Shortest path in an **unweighted** grid (each step costs 1)
- Distance from one source to all cells
- Distance from multiple sources simultaneously (multi-source BFS)
- Minimum steps to reach a target

## Template / Skeleton

### Single-Source BFS
```python
from collections import deque

def bfs_shortest_path(grid, sr, sc, er, ec):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    if grid[sr][sc] == 1 or grid[er][ec] == 1:   # blocked cells
        return -1

    queue = deque([(sr, sc, 0)])       # (row, col, distance)
    visited = set([(sr, sc)])

    while queue:
        r, c, dist = queue.popleft()

        if r == er and c == ec:
            return dist

        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and (nr, nc) not in visited
                    and grid[nr][nc] == 0):         # 0 = passable
                visited.add((nr, nc))
                queue.append((nr, nc, dist + 1))

    return -1  # unreachable
```

### Multi-Source BFS (KEY INSIGHT: all sources start at step 0)
```python
from collections import deque

def multi_source_bfs(grid):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]
    dist = [[-1] * COLS for _ in range(ROWS)]

    queue = deque()
    # Initialize with ALL sources at distance 0
    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == 0:         # source cells
                queue.append((r, c))
                dist[r][c] = 0

    while queue:
        r, c = queue.popleft()
        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and dist[nr][nc] == -1):    # not yet visited
                dist[nr][nc] = dist[r][c] + 1
                queue.append((nr, nc))

    return dist
```

### BFS with State (when you need more than position)
```python
# Example: shortest path avoiding obstacles with k removals allowed
queue = deque([(sr, sc, 0, k)])   # (row, col, dist, remaining_removals)
visited = set([(sr, sc, k)])

while queue:
    r, c, dist, removals = queue.popleft()
    if r == er and c == ec: return dist
    for dr, dc in DIRS:
        nr, nc = r + dr, c + dc
        if not in_bounds(nr, nc): continue
        new_removals = removals - (1 if grid[nr][nc] == 1 else 0)
        if new_removals >= 0 and (nr, nc, new_removals) not in visited:
            visited.add((nr, nc, new_removals))
            queue.append((nr, nc, dist + 1, new_removals))
```

## Signal phrases
"shortest path" / "minimum steps" / "minimum distance" / "nearest X" / "how many steps to reach" / "minimum time"

## Complexity
- Time: O(R×C) — each cell visited at most once
- Space: O(R×C) — queue and visited set

## Problems using this pattern
- [[dsa/grid/problems/maze-solver]] — shortest path from start to end with walls
- [[dsa/grid/problems/01-matrix]] — distance to nearest 0; multi-source BFS
- [[dsa/grid/problems/rotting-oranges]] — multi-source BFS; simulate time
- [[dsa/grid/problems/walls-and-gates]] — multi-source BFS from all gates
- [[dsa/grid/problems/shortest-path-in-binary-matrix]] — 8-directional BFS
- [[dsa/grid/problems/minimize-plant-infection-deaths]] — greedy quarantine; double BFS per component

## Why NOT DFS for shortest path?
DFS finds A path to the target, but not necessarily the shortest. In a grid, DFS might go around the long way. BFS is guaranteed to find the shortest path in unweighted graphs because it expands level by level (all distance-1 cells before distance-2 cells, etc.).

```
Grid:               DFS path (one possibility):   BFS path:
S . . . . E         S→→→↓ . E  (length 7)          S→ . . . →E (length 5)
. # # # . .         . # # # ↓ .
. . . . ↑ .         . . . . ↑ .
```

## Dijkstra on Grid (weighted version)
```python
import heapq

def dijkstra(grid):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    dist = [[float('inf')] * COLS for _ in range(ROWS)]
    dist[0][0] = grid[0][0]
    heap = [(grid[0][0], 0, 0)]   # (cost, row, col)

    while heap:
        cost, r, c = heapq.heappop(heap)
        if cost > dist[r][c]:
            continue
        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if 0 <= nr < ROWS and 0 <= nc < COLS:
                new_cost = cost + grid[nr][nc]
                if new_cost < dist[nr][nc]:
                    dist[nr][nc] = new_cost
                    heapq.heappush(heap, (new_cost, nr, nc))

    return dist[ROWS-1][COLS-1]
```

## Sources
- [[Grid overview]]
- [[dsa/patterns/bfs-dfs]]
