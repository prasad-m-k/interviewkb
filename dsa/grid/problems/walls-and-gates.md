# Walls and Gates

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/grid/patterns/bfs-shortest-path]]
**Companies:** [[dsa/companies/google]], [[dsa/companies/meta]], [[dsa/companies/apple]]

## Problem

You are given a 2D grid `rooms` initialized with:
- `-1` — a **wall** (obstacle, impassable)
- `0` — a **gate**
- `INF` (2³¹ - 1) — an **empty room**

Fill each empty room with the **distance to its nearest gate**. Walls remain `-1`. If a room cannot reach any gate, it stays `INF`.

```
Input:
INF  -1   0  INF
INF INF INF  -1
INF  -1 INF  -1
  0  -1 INF INF

Output:
  3  -1   0   1
  2   2   1  -1
  1  -1   2  -1
  0  -1   3   4
```

## Why this problem is important

Walls and Gates is the canonical example of **multi-source BFS** — starting BFS simultaneously from all sources (gates) rather than from a single source. This technique appears in many real-world problems:

- Nearest facility (hospital, fire station) in a map
- Nearest infected node in a graph (similar to [[dsa/grid/problems/rotting-oranges]])
- Distance-transform in image processing
- Nearest transit stop in routing problems

## Approach

**Naive approach (wrong):** Run BFS from each empty room → O(rooms × gates) — too slow, and complex.

**Correct: Multi-source BFS from all gates simultaneously.**

The key insight: BFS from all gates at once is equivalent to asking "what is the minimum distance from any gate to each room?" BFS guarantees shortest paths in unweighted graphs, and processing all sources together produces the correct minimums in one pass.

Algorithm:
1. Enqueue all gates (value 0) at distance 0.
2. BFS outward. When we reach an `INF` room, set its value to `current_distance + 1` and enqueue it.
3. Never re-visit a room (it already has the optimal distance from the nearest gate).

## Solution (Python)

```python
from collections import deque

def walls_and_gates(rooms: list[list[int]]) -> None:
    """Modifies rooms in-place."""
    if not rooms:
        return

    INF = 2**31 - 1
    rows, cols = len(rooms), len(rooms[0])
    queue = deque()

    # Step 1: seed the queue with all gates
    for r in range(rows):
        for c in range(cols):
            if rooms[r][c] == 0:
                queue.append((r, c))

    # Step 2: multi-source BFS
    directions = [(0,1),(0,-1),(1,0),(-1,0)]
    while queue:
        r, c = queue.popleft()
        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and rooms[nr][nc] == INF:
                rooms[nr][nc] = rooms[r][c] + 1  # distance from nearest gate
                queue.append((nr, nc))
```

## Complexity

| | Value |
|---|---|
| Time | O(rows × cols) — each cell visited at most once |
| Space | O(rows × cols) — queue at most all cells |

## Key Insight

**Multi-source BFS is more than a performance optimization — it gives the correct answer.** Single-source BFS from one gate would assign each room its distance from *that* gate; you'd need to run it from all gates and take minimums. Multi-source BFS does this in one pass because the BFS frontier expands from all sources simultaneously, and BFS always yields shortest paths first.

Compare with Dijkstra: BFS works here because all edge weights are 1. If movement had variable costs, you'd need Dijkstra.

## Variants / Follow-ups

- **Rotting Oranges:** Same multi-source BFS, but check if all fresh oranges were reached. [[dsa/grid/problems/rotting-oranges]]
- **Nearest 0 in Binary Matrix:** Same pattern. [[dsa/grid/problems/01-matrix]]
- "What if rooms is very large (won't fit in memory)?" → Streaming BFS — process in tiles. Or use a distributed graph algorithm (Pregel model).
- "What if we want the K nearest gates?" → Run BFS, stop after K unique distances from each room. More complex tracking.
- "What if gates can be added/removed dynamically?" → Precompute with BFS, then use a Voronoi-style partition that can be updated incrementally (much harder — interview context only up to static version).

## Sources
- [[dsa/companies/google]]
- [[dsa/companies/meta]]
