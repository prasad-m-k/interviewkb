# Topological Sort

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/graph]], [[dsa/concepts/deque]]

## What it solves
Ordering nodes in a directed acyclic graph (DAG) so that every edge u → v has u appearing before v. Used for dependency resolution, task scheduling, and detecting cycles in directed graphs.

## Template / skeleton

**Kahn's Algorithm (BFS — preferred in interviews):**
```python
from collections import deque, defaultdict

def topological_sort(n, edges):
    graph = defaultdict(list)
    indegree = [0] * n
    for u, v in edges:
        graph[u].append(v)
        indegree[v] += 1

    # Start with all nodes that have no dependencies
    queue = deque([i for i in range(n) if indegree[i] == 0])
    order = []

    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            indegree[neighbor] -= 1
            if indegree[neighbor] == 0:
                queue.append(neighbor)

    # If not all nodes are in order → cycle exists
    return order if len(order) == n else []
```

**DFS post-order (alternative):**
```python
def topo_dfs(n, graph):
    visited, in_stack, order = set(), set(), []
    def dfs(u):
        in_stack.add(u)
        for v in graph[u]:
            if v in in_stack: return False   # cycle
            if v not in visited:
                visited.add(v)
                if not dfs(v): return False
        in_stack.remove(u)
        order.append(u)
        return True
    for i in range(n):
        if i not in visited:
            visited.add(i)
            if not dfs(i): return []
    return order[::-1]
```

## Signal phrases
- "given a list of prerequisites / dependencies..."
- "can all tasks / courses be completed?"
- "find a valid ordering / schedule"
- "build order"
- "detect cycle in a directed graph"

## Complexity
O(V + E) time, O(V + E) space.

## MLOps connection
ML pipelines are DAGs. Kahn's algorithm is essentially what orchestrators (Airflow, Kubeflow, Prefect) use to determine task execution order and detect circular dependencies.

## Problems using this pattern
- [[dsa/problems/course-schedule-ii]]

## Sources
- [[DSA overview]]
