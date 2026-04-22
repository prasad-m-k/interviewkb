# Graph

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/patterns/bfs-dfs]], [[dsa/patterns/topological-sort]]

## What it is
A graph is a set of vertices (nodes) and edges (connections between nodes). Edges can be directed or undirected, weighted or unweighted.

## How it works

**Representations:**
- **Adjacency list** (preferred): `graph[u]` = list of neighbors. Space O(V + E).
- **Adjacency matrix**: `matrix[u][v] = 1` if edge exists. Space O(V²); good for dense graphs.
- **Edge list**: list of (u, v) pairs. Used as input format; convert to adjacency list before processing.

**Build from edge list:**
```python
from collections import defaultdict
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)          # directed
    graph[v].append(u)          # add this line for undirected
```

## Complexity
| Algorithm | Time | Space |
|---|---|---|
| BFS / DFS | O(V + E) | O(V) |
| Topological Sort | O(V + E) | O(V + E) |
| Dijkstra (min-heap) | O((V+E) log V) | O(V) |

## When to use
- **Connectivity**: "how many connected components?" → DFS/BFS flood fill
- **Dependency ordering**: "can all tasks be completed?" → topological sort + cycle detection
- **Shortest path (unweighted)**: BFS gives shortest path in number of edges
- **Shortest path (weighted)**: Dijkstra with min-heap

## Common interview angles
- Build the graph from the input (often an edge list or matrix)
- Detect cycles in directed graphs (DFS with 3 colors: unvisited / in-stack / done)
- Topological sort → check if `len(order) == numNodes` to detect cycles
- Grid problems: treat each cell as a node, 4-directional edges to neighbors

## Examples
```python
# DFS (iterative)
def dfs(graph, start):
    visited = set()
    stack = [start]
    while stack:
        node = stack.pop()
        if node not in visited:
            visited.add(node)
            stack.extend(graph[node])

# BFS
from collections import deque
def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# Cycle detection in directed graph (DFS 3-color)
def has_cycle(graph, n):
    # 0=unvisited, 1=in-stack, 2=done
    state = [0] * n
    def dfs(u):
        state[u] = 1
        for v in graph[u]:
            if state[v] == 1: return True   # back edge = cycle
            if state[v] == 0 and dfs(v): return True
        state[u] = 2
        return False
    return any(dfs(u) for u in range(n) if state[u] == 0)
```

## Sources
- [[DSA overview]]
