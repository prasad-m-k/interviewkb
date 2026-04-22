# Grid as a Graph

**Topic:** [[Grid overview]]
**Related:** [[dsa/patterns/bfs-dfs]], [[dsa/grid/concepts/direction-vectors]]

## What it is
The key conceptual shift: a grid is a graph where cells are nodes and edges connect adjacent cells. This maps every grid problem to a known graph algorithm.

## The Mapping

```
Grid cell (r, c)     →  Graph node
Adjacent cells        →  Edges (weighted or unweighted)
Cell value            →  Node property (passable/blocked, cost)
grid[r][c] != '#'     →  Edge exists
grid[r][c] == '#'     →  Edge blocked (wall)
```

## BFS vs. DFS on a Grid

| | BFS | DFS |
|---|---|---|
| **Guarantee** | **Shortest path** (unweighted) | Any valid path |
| **Data structure** | Queue (deque) | Stack (recursion or explicit) |
| **Space** | O(R×C) for queue | O(R×C) for call stack |
| **When to use** | Shortest path, level-by-level expansion | Connectivity, counting regions, backtracking |
| **Risk** | More memory (queue can grow) | Stack overflow for very large grids (use iterative DFS) |

### Rule of thumb
- **Needs shortest path?** → BFS, always.
- **Just need to explore connectivity?** → DFS (simpler to write, similar complexity).
- **Weighted edges?** → Dijkstra (priority queue BFS).

## BFS Template on Grid

```python
from collections import deque

def bfs(grid, start_r, start_c, target_r, target_c):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    queue = deque([(start_r, start_c, 0)])   # (row, col, distance)
    visited = set()
    visited.add((start_r, start_c))

    while queue:
        r, c, dist = queue.popleft()

        if r == target_r and c == target_c:
            return dist

        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and (nr, nc) not in visited
                    and grid[nr][nc] != '#'):
                visited.add((nr, nc))
                queue.append((nr, nc, dist + 1))

    return -1  # unreachable
```

## DFS Template on Grid

```python
def dfs(grid, r, c, visited):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    if not (0 <= r < ROWS and 0 <= c < COLS):
        return
    if visited[r][c] or grid[r][c] == '#':
        return

    visited[r][c] = True

    for dr, dc in DIRS:
        dfs(grid, r + dr, c + dc, visited)
```

## Multi-Source BFS (very important pattern)
When you need the distance from **any** of multiple starting cells, initialize the BFS queue with all sources simultaneously:

```python
# Example: distance to nearest 0 in a grid
queue = deque()
for r in range(ROWS):
    for c in range(COLS):
        if grid[r][c] == 0:
            queue.append((r, c, 0))
            visited.add((r, c))
# Now BFS expands from ALL zeros simultaneously
# Each cell gets the distance to its NEAREST zero
```
Used in: [[dsa/grid/problems/01-matrix]], [[dsa/grid/problems/rotting-oranges]], [[dsa/grid/problems/walls-and-gates]], [[dsa/grid/problems/pacific-atlantic-waterflow]].

## Dijkstra on Grid (weighted)

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

## Common interview angles
- "Why BFS and not DFS for shortest path?" (DFS finds A path, not necessarily the shortest one; BFS explores level by level, so the first time it reaches the target is via the shortest path)
- "What is the time complexity of BFS on a grid?" (O(R×C) — each cell is visited at most once)
- "Can you do multi-source BFS?" (yes — add all sources to the queue before starting; BFS naturally handles this)

## Sources
- [[Grid overview]]
- [[dsa/patterns/bfs-dfs]]
