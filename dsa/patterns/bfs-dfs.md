# BFS / DFS

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/graph]], [[dsa/concepts/deque]]

## What it solves
Graph traversal, connected component counting, shortest path in unweighted graphs, flood fill, and cycle detection.

## Template / skeleton

**BFS (queue — shortest path, level order):**
```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# BFS with distance tracking
def bfs_distance(graph, start, target):
    visited = {start}
    queue = deque([(start, 0)])
    while queue:
        node, dist = queue.popleft()
        if node == target:
            return dist
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, dist + 1))
    return -1
```

**DFS (recursive — connectivity, topological sort):**
```python
def dfs(graph, node, visited):
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)

# DFS flood fill on a grid
def dfs_grid(grid, r, c):
    rows, cols = len(grid), len(grid[0])
    if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
        return
    grid[r][c] = '0'   # mark visited in-place
    for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
        dfs_grid(grid, r + dr, c + dc)
```

## Signal phrases
- "number of connected components / islands"
- "shortest path between X and Y"
- "can you reach from A to B"
- "level-order traversal"
- "all paths from source to target"
- "detect cycle in a graph"

## Complexity
O(V + E) time, O(V) space for the visited set and call stack/queue.

## Problems using this pattern
- [[dsa/problems/number-of-islands]]
- [[dsa/problems/course-schedule-ii]]

## Sources
- [[DSA overview]]
