# Course Schedule II

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/topological-sort]]
**Companies:** Google, Amazon, Meta, Apple

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
There are `numCourses` courses labeled `0` to `numCourses-1`. Given a list of `prerequisites` pairs `[a, b]` meaning "to take course `a`, you must first take course `b`", return a valid ordering to finish all courses. If impossible (cycle), return an empty list.

## Approach
This is a topological sort problem on a directed graph. Each course is a node; each prerequisite `[a, b]` is a directed edge b → a.

Use **Kahn's algorithm** (BFS with in-degree tracking):
1. Build adjacency list and compute in-degree for each node.
2. Initialize a queue with all nodes of in-degree 0 (no prerequisites).
3. Process each node: append to result, decrement in-degrees of its neighbors, enqueue neighbors with in-degree 0.
4. If result contains all `numCourses` nodes → valid ordering. Otherwise → cycle detected.

Cycle detection is automatic: if there's a cycle, some nodes never reach in-degree 0 and are never enqueued.

## Solution (Python)
```python
from collections import deque, defaultdict
from typing import List

def findOrder(numCourses: int, prerequisites: List[List[int]]) -> List[int]:
    graph = defaultdict(list)
    indegree = [0] * numCourses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        indegree[course] += 1

    queue = deque([i for i in range(numCourses) if indegree[i] == 0])
    order = []

    while queue:
        course = queue.popleft()
        order.append(course)
        for next_course in graph[course]:
            indegree[next_course] -= 1
            if indegree[next_course] == 0:
                queue.append(next_course)

    return order if len(order) == numCourses else []
```

## Complexity
Time: O(V + E) where V = numCourses, E = len(prerequisites) | Space: O(V + E)

## Key insight
`len(order) == numCourses` is the cycle check — if any node is part of a cycle, its in-degree never reaches 0, so it never enters the queue, so it never enters `order`. You don't need a separate cycle-detection pass.

## Variants / follow-ups
- **Course Schedule I**: just return True/False (cycle or not) — same algorithm, skip storing order.
- **Alien Dictionary**: build the graph from character ordering constraints; same Kahn's approach.
- **Build System**: given build targets with dependencies, find a valid build order.
- **Interviewers ask**: "What if there are multiple valid orderings?" → any BFS order works; for lexicographically smallest, use a min-heap instead of a queue.

## MLOps connection
ML pipeline orchestrators (Airflow, Kubeflow, Prefect) implement exactly this algorithm to schedule steps in a DAG. A cycle in the pipeline graph means two steps each depend on the other — the orchestrator must detect and reject this.

## Sources
- [[DSA overview]]
