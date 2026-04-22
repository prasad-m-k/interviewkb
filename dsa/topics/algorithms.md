# Algorithms

**Topic:** [[dsa/topics/algorithms]]
**Related:** [[dsa/topics/data-structures]]

Survey of algorithm patterns that appear in FAANG coding interviews for senior MLOps engineers.

---

## BFS / DFS

**BFS** (Breadth-First Search): explores level by level using a queue. Finds shortest paths in unweighted graphs.

**DFS** (Depth-First Search): explores as deep as possible using a stack (or recursion). Good for connectivity, cycle detection, topological sort.

- Use BFS when you need shortest path or level-order traversal
- Use DFS when you need connectivity, components, or topological ordering
- Both are O(V + E)

**See:** [[dsa/patterns/bfs-dfs]]
**Problems:** [[dsa/problems/number-of-islands]], [[dsa/problems/course-schedule-ii]]

---

## Topological Sort

Orders nodes in a DAG such that every directed edge u → v has u before v. Two implementations:

- **Kahn's (BFS)**: track in-degrees; process nodes with in-degree 0; detect cycles by checking if all nodes were visited
- **DFS post-order**: push to stack after visiting all neighbors; reverse the stack

Kahn's is preferred in interviews — easier to explain, naturally detects cycles.

**See:** [[dsa/patterns/topological-sort]]
**Problems:** [[dsa/problems/course-schedule-ii]]

---

## Dynamic Programming

Break the problem into overlapping subproblems with optimal substructure.

- **Bottom-up (tabulation)**: fill a DP table iteratively — preferred in interviews for clarity
- **Top-down (memoization)**: recursion + cache — easier to reason about, but watch stack depth

Standard interview checklist:
1. Define `dp[i]` clearly in English
2. Write the recurrence relation
3. Identify base cases
4. Determine traversal order

**See:** [[dsa/patterns/dynamic-programming]]
**Problems:** [[dsa/problems/word-break]]

---

## Stack

Four distinct usage modes — all O(n) amortized because each element is pushed and popped at most once:

| Mode | Stack order | What you get at pop/push | Example |
|---|---|---|---|
| **Monotonic decreasing** | top = smallest | next greater element to the right | Daily Temperatures |
| **Monotonic increasing** | top = largest | next smaller / left+right boundaries | Largest Rectangle |
| **Matching** | unmatched openers | balanced nesting check | Valid Parentheses |
| **Simulation** | surviving entities | collision / cancel resolution | Asteroid Collision |

**Monotonic stack key:** the nested `while` loop looks O(n²) but is actually O(n) — each element enters and exits the stack exactly once.

**See:** [[dsa/patterns/monotonic-stack]], [[dsa/concepts/stack]]  
**Problems:** [[dsa/problems/daily-temperatures]], [[dsa/problems/largest-rectangle-histogram]], [[dsa/problems/valid-parentheses]], [[dsa/problems/asteroid-collision]]

---

## Greedy

Make the locally optimal choice at each step. Works when the greedy choice property holds.

- Harder to prove correct than DP — in interviews, state your greedy invariant explicitly
- Often combined with a heap to efficiently find the local optimum

**Problems:** [[dsa/problems/task-scheduler]]

---

## Sliding Window

Maintain a window [left, right] over an array or string. Expand right; shrink left when invariant is violated.

Two variants:
- **Fixed-size window**: slide right one step, remove left element
- **Variable-size window**: shrink until the window is valid again

**See:** [[dsa/patterns/sliding-window]]
**Problems:** [[dsa/problems/sliding-window-maximum]], [[dsa/problems/design-hit-counter]]

---

## Binary Search

O(log n) search on sorted data. The classic mistake is getting the boundary conditions wrong.

Template: `left, right = 0, len(nums) - 1` → `while left <= right` → `mid = left + (right - left) // 2`

Also applies to "search on answer" problems: binary search on the *result space*, not the array.

---

## Two Pointers

Two indices moving through the same array or two arrays. Common for sorted array problems, palindrome checks, pair sums.

**Variants:**
- Same direction (fast/slow) — cycle detection, sliding window
- Opposite direction (converge) — sorted pair sum, container with most water

---

## Backtracking

Explore all valid configurations by building candidates incrementally, abandoning ("backtracking") as soon as a partial candidate can't lead to a valid solution.

**Three-step skeleton:** choose → explore → unchoose. The undo step is the defining feature.

**N-Queens design:** when placing items on a grid/board, track constraints in sets (column, diagonal, box) for O(1) conflict checks instead of O(n) board scans.

| Problem family | Constraint tracked | Key |
|---|---|---|
| N-Queens | cols, `row-col`, `row+col` | place one queen per row |
| Sudoku | rows, cols, boxes `(r//3, c//3)` | most-constrained cell first |
| Word Search | visited path cells | unmark on backtrack |
| Combination Sum | sorted input + start index | avoid duplicate combos |
| Generate Parentheses | open/close counts | `close < open` invariant |

**See:** [[dsa/patterns/backtracking]]  
**Problems:** [[dsa/problems/n-queens]], [[dsa/problems/sudoku-solver]], [[dsa/problems/combination-sum]], [[dsa/problems/generate-parentheses]]
