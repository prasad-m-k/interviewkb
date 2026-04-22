# Grid Problems — Overview

## The One Mental Model You Need

**A grid is a graph.** Every cell is a node. Edges connect adjacent cells (4 or 8 directions depending on the problem). Once you see it as a graph, every grid problem becomes a graph problem you already know.

```
Grid:           Graph:
[1][2][3]       (0,0)-(0,1)-(0,2)
[4][5][6]         |     |     |
[7][8][9]       (1,0)-(1,1)-(1,2)
                  |     |     |
                (2,0)-(2,1)-(2,2)
```

## The Four Questions to Ask First

1. **How many directions?** 4 (up/down/left/right) or 8 (including diagonals)?
2. **Do I need shortest path?** Yes → BFS. No → DFS.
3. **Is there state to track per step?** Yes → weighted BFS/Dijkstra, or DP.
4. **Do I modify the grid?** Yes → in-place marking. No → separate visited array.

## Decision Framework

```
Grid problem?
  │
  ├─ Traversal / structure
  │   ├─ Spiral / diagonal / rotation → [[patterns/simulation]]
  │   └─ Layer-by-layer → [[patterns/matrix-traversal]]
  │
  ├─ Connectivity / counting regions
  │   ├─ Count islands / connected components → [[patterns/flood-fill-dfs]]
  │   └─ Mark regions → DFS with in-place marking
  │
  ├─ Shortest path (unweighted)
  │   ├─ Single source → BFS from start
  │   └─ Multiple sources → [[patterns/bfs-shortest-path]] (multi-source BFS)
  │
  ├─ Shortest path (weighted)
  │   └─ Dijkstra with min-heap
  │
  ├─ Infection / spread simulation with intervention (quarantine, block)
  │   └─ Greedy: quarantine max-threat component each day → [[problems/minimize-plant-infection-deaths]]
  │
  ├─ Path exists? → DFS or BFS
  │
  ├─ Count / sum paths → [[patterns/dynamic-programming-grid]]
  │
  ├─ Search for value in sorted matrix → [[patterns/binary-search-2d]]
  │   ├─ Rows stitch (last[i] < first[i+1]) → flatten O(log RC)
  │   └─ Row+col sorted only → staircase from top-right O(R+C)
  │
  └─ Search for pattern in grid → [[problems/word-search]] (DFS + backtrack)
```

## Master Template (copy-paste starter)

```python
def solve(grid):
    if not grid or not grid[0]:
        return

    ROWS, COLS = len(grid), len(grid[0])
    # 4-directional movement
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def in_bounds(r, c):
        return 0 <= r < ROWS and 0 <= c < COLS

    visited = [[False]*COLS for _ in range(ROWS)]

    def dfs(r, c):
        if not in_bounds(r, c) or visited[r][c]:
            return
        visited[r][c] = True
        for dr, dc in DIRS:
            dfs(r + dr, c + dc)

    for r in range(ROWS):
        for c in range(COLS):
            if not visited[r][c]:
                dfs(r, c)
```

## The Most Common Mistakes

1. **Not checking bounds before accessing grid[r][c]** — always `in_bounds(r, c)` first
2. **Forgetting to mark visited** — causes infinite loops in DFS, re-processing in BFS
3. **Using DFS when shortest path is needed** — DFS finds A path, not the SHORTEST path; use BFS
4. **Forgetting to unmark on backtrack** — necessary in word search; not in flood fill
5. **Off-by-one in spiral traversal** — track top/bottom/left/right pointers carefully
6. **Modifying grid during iteration** — safe for flood-fill (use -1 marker); dangerous for word search (unmark after)

## Time/Space Complexity Reference

| Pattern | Time | Space | Notes |
|---|---|---|---|
| DFS traversal | O(R×C) | O(R×C) | recursion stack up to R×C |
| BFS traversal | O(R×C) | O(R×C) | queue holds up to R×C cells |
| Multi-source BFS | O(R×C) | O(R×C) | all sources start simultaneously |
| DP on grid | O(R×C) | O(R×C) or O(C) | row-by-row reduces space |
| Spiral | O(R×C) | O(1) | pointer-based; no extra space |
| Dijkstra on grid | O(R×C log(R×C)) | O(R×C) | priority queue |
| Binary search (flatten) | O(log(R×C)) | O(1) | rows must stitch |
| Binary search (staircase) | O(R + C) | O(1) | row+col sorted |

## Sources
- [[dsa/patterns/bfs-dfs]]
- [[dsa/index]]
