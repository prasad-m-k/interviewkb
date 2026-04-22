# Swim in Rising Water

**Difficulty:** Hard
**Pattern:** Dijkstra / Binary Search + BFS on grid
**Topic:** [[Grid overview]]
**Companies:** Google, Amazon

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
You are given an n×n grid where `grid[r][c]` represents the elevation. Starting at (0,0), you can travel to any adjacent cell (4-directional). At time t, you can swim from cell A to cell B only if `max(grid[A], grid[B]) <= t`. Find the minimum time to reach (n-1, n-1).

## Approach 1: Dijkstra (optimal, O(n² log n))
Treat `cost(path)` as the maximum elevation encountered. Use a min-heap. At each step, expand the cell with the lowest maximum elevation so far.

```python
import heapq

def swimInWater(grid):
    n = len(grid)
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    # (max_elevation_on_path_so_far, row, col)
    heap = [(grid[0][0], 0, 0)]
    visited = set()

    while heap:
        t, r, c = heapq.heappop(heap)

        if (r, c) in visited:
            continue
        visited.add((r, c))

        if r == n-1 and c == n-1:
            return t

        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if 0 <= nr < n and 0 <= nc < n and (nr, nc) not in visited:
                new_t = max(t, grid[nr][nc])
                heapq.heappush(heap, (new_t, nr, nc))

    return -1
```

## Approach 2: Binary Search + BFS (O(n² log n), conceptually cleaner)
Binary search on time t. For a given t, can you reach (n-1,n-1) only using cells with elevation ≤ t? Check with BFS.

```python
from collections import deque

def swimInWater_v2(grid):
    n = len(grid)
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def can_reach(t):
        if grid[0][0] > t:
            return False
        queue = deque([(0, 0)])
        visited = {(0, 0)}
        while queue:
            r, c = queue.popleft()
            if r == n-1 and c == n-1:
                return True
            for dr, dc in DIRS:
                nr, nc = r + dr, c + dc
                if (0 <= nr < n and 0 <= nc < n
                        and (nr, nc) not in visited
                        and grid[nr][nc] <= t):
                    visited.add((nr, nc))
                    queue.append((nr, nc))
        return False

    lo, hi = grid[0][0], n * n - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if can_reach(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

## Complexity
Dijkstra: O(n² log n) | Binary Search + BFS: O(n² log n²) = O(n² log n)

## Key insight
This is "minimum of maximum" — a class of problems solvable by either Dijkstra (greedy: always expand the cheapest frontier) or binary search + feasibility check. The max-heap stores the bottleneck elevation, not the sum.

**Difference from standard Dijkstra**: cost = `max(path costs)` instead of `sum(path costs)`. The heap still works — greedy still optimal.

## Variants / follow-ups
- **Path with minimum effort** (LeetCode 1631): same pattern; cost = max absolute difference in height between consecutive cells

## Sources
- [[dsa/grid/patterns/bfs-shortest-path]]
- [[Grid overview]]
