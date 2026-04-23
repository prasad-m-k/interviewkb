# DSA Knowledge Base — Overview

**Audience:** Senior MLOps engineers preparing for FAANG technical interviews
**Last updated:** 2026-04-21

---

## Why DSA Matters for MLOps Engineers

Most MLOps engineers assume their system-design depth will carry them through coding rounds. It won't. FAANG coding interviews are language-agnostic and test the same DS&A fundamentals regardless of role. The difference for senior ML engineers is *which* problems appear most often and how interviewers frame them.

| Interview Problem | MLOps Analogy |
|---|---|
| LRU Cache | Model/feature cache eviction |
| Course Schedule (Topological Sort) | ML pipeline DAG scheduling |
| Sliding Window Maximum | Rolling-window feature computation |
| Find Median from Data Stream | Online statistics for drift monitoring |
| Merge k Sorted Lists | Merging sorted feature/log streams |
| Task Scheduler | Pipeline job scheduling with cooldowns |
| Top K Frequent Elements | Feature importance ranking |
| Number of Islands | Cluster/component detection |
| Word Break | Sequence tokenization / DP on strings |
| Design Hit Counter | Inference API rate-limiting |

---

## Why DSA Matters for SRE & Production Engineers

For SRE and Production Engineering roles, DSA questions shift from abstract puzzles to **operational coding**. Interviewers look for systems maturity: handling malformed inputs, managing memory for large files, and concurrency safety.

| Interview Problem | SRE / PE Analogy |
|---|---|
| [[dsa/problems/design-hit-counter]] | Monitoring metrics (past 5 min RPS) |
| [[dsa/problems/course-schedule-ii]] | Service dependency resolution / Dag execution |
| [[dsa/problems/lru-cache]] | In-memory cache for high-traffic endpoints |
| [[dsa/problems/merge-intervals]] | Combining maintenance windows or alert ranges |
| [[dsa/problems/task-scheduler]] | Job runner with execution constraints |
| [[dsa/problems/top-k-frequent-elements]] | Identifying the top talkers in a network log |
| [[dsa/problems/daily-temperatures]] | Finding the next spike in a metrics stream |

---

## Role-Specific Deep Dives
- [[dsa/companies/google]] — SRE: streaming, NALSD, concurrency, systems scripting
- [[dsa/companies/meta]] — PE: speed coding, systems scripting, Linux troubleshooting
- [[dsa/companies/apple]] — Heavy on trees, intervals, arrays; clean code emphasis

---

## Core Competencies Required

### Data Structures
- **HashMap / HashSet** — O(1) lookup; the most-used structure in interviews
- **Heap (Priority Queue)** — O(log n) insert/extract-min/max; essential for Top-K and streaming problems
- **Deque (Monotonic Queue)** — O(1) amortized sliding window problems
- **Graph (Adjacency List)** — Dependency graphs, connected components
- **Doubly Linked List** — Combined with HashMap for O(1) cache operations

### Algorithm Patterns
- **BFS / DFS** — Graph traversal, shortest paths, connected components
- **Topological Sort** — Directed acyclic graphs, dependency resolution
- **Two Heaps** — Partition a stream into two sorted halves for median queries
- **Sliding Window** — Subarray/substring problems with a moving range
- **Dynamic Programming** — Optimal substructure; bottom-up tabulation preferred in interviews

---

## Difficulty Distribution (Top 10)

| # | Problem | Difficulty | Pattern |
|---|---|---|---|
| 1 | LRU Cache | Medium | HashMap + DLL |
| 2 | Course Schedule II | Medium | Topological Sort |
| 3 | Sliding Window Maximum | Hard | Monotonic Deque |
| 4 | Find Median from Data Stream | Hard | Two Heaps |
| 5 | Top K Frequent Elements | Medium | Heap / Bucket Sort |
| 6 | Number of Islands | Medium | BFS / DFS |
| 7 | Merge k Sorted Lists | Hard | Min Heap |
| 8 | Task Scheduler | Medium | Greedy + Heap |
| 9 | Word Break | Medium | Dynamic Programming |
| 10 | Design Hit Counter | Medium | Queue / Sliding Window |

---

## Study Strategy for Senior Engineers

1. **Don't just memorize solutions.** Interviewers at senior level probe *why* you chose a data structure. Practice explaining trade-offs aloud.
2. **Connect to your domain.** If an interviewer asks about LRU Cache, mention you've seen similar eviction logic in feature stores. It signals depth.
3. **Time yourself.** Target 15–20 min for Medium, 25–30 min for Hard. Senior engineers are expected to be faster, not just more correct.
4. **Write clean Python.** Use `collections.deque`, `heapq`, `collections.Counter`. Familiarity with the standard library signals experience.
5. **State complexity upfront.** Senior engineers are expected to lead with complexity analysis, not arrive at it after probing.
