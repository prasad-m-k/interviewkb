# Flashcards — Top 10 DSA Problems for MLOps Engineers (FAANG)

**Topic:** [[dsa/topics/algorithms]], [[dsa/topics/data-structures]]
**Created:** 2026-04-21

Format: read the question, formulate your answer, then expand the callout to check.

---

## Card 1
**Q: LRU Cache — Design a data structure with O(1) `get(key)` and `put(key, value)` using LRU eviction. What data structures do you combine and why?**

> [!note]- Answer
> **Data structures:** HashMap + Doubly Linked List (DLL)
>
> **Why the combination?**
> - HashMap → O(1) key lookup to find the node
> - DLL → O(1) move-to-front and remove-from-back *given a pointer* to the node
> - Neither alone is sufficient: HashMap can't track order; DLL alone is O(n) for lookup
>
> **Design:**
> - DLL head = LRU end, tail = MRU end; two dummy sentinels eliminate all edge cases
> - `get(key)`: lookup in map → remove node → insert at tail → return val
> - `put(key, val)`: if key exists, remove old node; create new node, insert at tail, add to map; if over capacity, remove `head.next` (LRU) and delete from map
>
> **Complexity:** O(1) get and put | O(capacity) space
>
> **Key gotcha:** dummy sentinel nodes are not optional style — they make every pointer operation uniform regardless of list size.
>
> *See also:* [[dsa/problems/lru-cache]]

---

## Card 2
**Q: Course Schedule II — Given `numCourses` and a list of `[course, prereq]` pairs, return a valid course order or `[]` if impossible. What algorithm do you use and how does cycle detection work?**

> [!note]- Answer
> **Algorithm:** Kahn's topological sort (BFS with in-degree tracking)
>
> **Steps:**
> 1. Build adjacency list; compute `indegree[course]` for each course
> 2. Queue all courses with `indegree == 0` (no prerequisites)
> 3. Process queue: pop course, append to `order`, decrement neighbors' in-degrees, enqueue any neighbor that reaches `indegree == 0`
> 4. Return `order` if `len(order) == numCourses`, else `[]`
>
> **Cycle detection is free:** nodes in a cycle never reach `indegree == 0`, so they never enter the queue, so `len(order) < numCourses`.
>
> **Complexity:** O(V + E) time, O(V + E) space
>
> **MLOps connection:** ML pipeline orchestrators use this exact algorithm to schedule DAG steps and detect circular dependencies.
>
> *See also:* [[dsa/problems/course-schedule-ii]], [[dsa/patterns/topological-sort]]

---

## Card 3
**Q: Sliding Window Maximum — Given `nums` and window size `k`, return the max of each window in O(n). What is the data structure and what invariant does it maintain?**

> [!note]- Answer
> **Data structure:** Monotonic decreasing deque of *indices*
>
> **Invariant:** Elements in the deque are always in decreasing order of `nums[index]`. The front of the deque is always the index of the current window's maximum.
>
> **Per element `i`:**
> 1. Evict expired indices from front: `while dq and dq[0] < i - k + 1: popleft()`
> 2. Maintain decreasing order: `while dq and nums[dq[-1]] < nums[i]: pop()`
> 3. Append `i`
> 4. If `i >= k - 1`: `result.append(nums[dq[0]])`
>
> **Why store indices, not values?** You need to check whether `dq[0]` is still inside the current window, which requires the original position.
>
> **Complexity:** O(n) time — each index pushed and popped at most once | O(k) space
>
> *See also:* [[dsa/problems/sliding-window-maximum]], [[dsa/patterns/sliding-window]]

---

## Card 4
**Q: Find Median from Data Stream — Support `addNum` in O(log n) and `findMedian` in O(1) on an unbounded stream. What is the two-heap approach?**

> [!note]- Answer
> **Structure:** Two heaps
> - `lo`: max-heap holding the **lower half** (Python: store negated values in `heapq`)
> - `hi`: min-heap holding the **upper half**
>
> **Invariant:** `len(lo) - len(hi)` is 0 or 1; every element in `lo` ≤ every element in `hi`
>
> **`addNum(num)` — 3 steps:**
> 1. Push `num` to `lo` (max-heap)
> 2. Move `lo`'s max to `hi` (enforces ordering invariant)
> 3. If `len(hi) > len(lo)`: move `hi`'s min back to `lo` (enforces size invariant)
>
> **`findMedian`:**
> - Odd count: return `-lo[0]`
> - Even count: return `(-lo[0] + hi[0]) / 2.0`
>
> **Complexity:** O(log n) addNum, O(1) findMedian
>
> *See also:* [[dsa/problems/find-median-from-data-stream]], [[dsa/patterns/two-heaps]]

---

## Card 5
**Q: Top K Frequent Elements — Return the k most frequent elements in better than O(n log n). Give both the heap and bucket sort approaches.**

> [!note]- Answer
> **Heap approach — O(n log k):**
> 1. `count = Counter(nums)`
> 2. `heapq.nlargest(k, count, key=count.get)` — or maintain a min-heap of size k manually
>
> **Bucket sort approach — O(n):**
> 1. Count frequencies
> 2. Create `buckets[i]` = list of nums with frequency `i`; array length = `len(nums) + 1`
> 3. Scan from high to low frequency, collect until k elements gathered
>
> **Why bucket sort is O(n):** frequency is bounded by `len(nums)`, so we can use counting sort on the frequency values.
>
> **Complexity:** Bucket sort: O(n) time and space. Heap: O(n log k) time, O(n) space.
>
> **Interview move:** mention bucket sort as the optimal approach, implement heap if the bucket sort explanation is unclear.
>
> *See also:* [[dsa/problems/top-k-frequent-elements]]

---

## Card 6
**Q: Number of Islands — Count connected components in a 2D binary grid. Walk through the BFS flood fill approach and its complexity.**

> [!note]- Answer
> **Approach:** BFS/DFS flood fill (connected components on an implicit graph)
>
> **Algorithm:**
> 1. Scan every cell
> 2. When `grid[r][c] == '1'`: increment count, start BFS/DFS from `(r, c)`
> 3. BFS marks every reachable `'1'` cell as `'0'` to prevent revisits
>
> **BFS skeleton:**
> ```
> queue = deque([(r, c)]); grid[r][c] = '0'
> while queue:
>     row, col = queue.popleft()
>     for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
>         if in_bounds and grid[nr][nc] == '1':
>             grid[nr][nc] = '0'; queue.append((nr, nc))
> ```
>
> **In-place marking** avoids a `visited` set — acceptable if input mutation is allowed.
>
> **Complexity:** O(m × n) time, O(min(m, n)) BFS space
>
> **Why prefer BFS over DFS?** DFS can hit Python's recursion limit on large grids (default 1000).
>
> *See also:* [[dsa/problems/number-of-islands]], [[dsa/patterns/bfs-dfs]]

---

## Card 7
**Q: Merge K Sorted Lists — Merge k linked lists each sorted in ascending order. What is the heap approach and its complexity?**

> [!note]- Answer
> **Algorithm:** Min-heap of size k
>
> **Steps:**
> 1. Push `(node.val, list_index, node)` for each list's head into a min-heap
> 2. Pop the minimum triple; append `node` to the result list
> 3. If `node.next` exists, push `(node.next.val, list_index, node.next)` to the heap
> 4. Repeat until heap is empty
>
> **Why `list_index` in the tuple?** Python can't compare `ListNode` objects. When two values are equal, Python tries to compare the next tuple element — `list_index` is an int and resolves the tie without ever comparing nodes.
>
> **Complexity:** O(N log k) where N = total nodes, k = number of lists | O(k) heap space
>
> **Alternative:** Divide and conquer — recursively merge pairs. Same O(N log k) time, O(log k) stack space.
>
> *See also:* [[dsa/problems/merge-k-sorted-lists]]

---

## Card 8
**Q: Task Scheduler — Minimum intervals to finish all tasks with cooldown n (same task can't repeat within n intervals). What is the greedy insight?**

> [!note]- Answer
> **Greedy insight:** Always execute the task with the highest remaining count. This minimizes idle time because the most frequent task is the bottleneck — any delay forces more idle intervals.
>
> **Simulation with max-heap + cooldown queue:**
> - Max-heap: `(-count,)` for each task type
> - Cooldown queue: `(remaining_count, available_at_time)`
> - Each tick: re-enqueue tasks whose cooldown expired → pop max-heap → execute → push to cooldown if tasks remain
>
> **O(1) space math shortcut (follow-up):**
> `result = max(len(tasks), (max_freq - 1) * (n + 1) + num_tasks_with_max_freq)`
>
> Logic: the most frequent task defines the frame structure: `(max_freq - 1)` full frames of `(n + 1)` slots, plus one final slot per task tied at max frequency.
>
> **Complexity:** O(n_tasks × n) simulation | O(1) with formula
>
> *See also:* [[dsa/problems/task-scheduler]]

---

## Card 9
**Q: Word Break — Can string `s` be segmented into words from `wordDict`? Give the DP formulation: state, recurrence, base case.**

> [!note]- Answer
> **State:** `dp[i] = True` if `s[:i]` can be segmented into dictionary words
>
> **Recurrence:** `dp[i] = True` if ∃ `j < i` such that `dp[j] == True` and `s[j:i] ∈ word_set`
>
> **Base case:** `dp[0] = True` (empty string is trivially segmentable)
>
> **Answer:** `dp[len(s)]`
>
> **Implementation notes:**
> - Convert `wordDict` to a `set` for O(1) lookup
> - `break` after setting `dp[i] = True` to prune the inner loop
> - Outer loop: `i` from 1 to n; inner loop: `j` from 0 to i
>
> **Complexity:** O(n²) time, O(n) space
>
> **Follow-up — Word Break II (return all sentences):** use memoized DFS, not DP, because you need to reconstruct paths not just True/False.
>
> *See also:* [[dsa/problems/word-break]], [[dsa/patterns/dynamic-programming]]

---

## Card 10
**Q: Design Hit Counter — Count hits in the past 300 seconds. Give both the deque approach and the circular buffer (O(1) space) follow-up.**

> [!note]- Answer
> **Deque approach:**
> - Store `(timestamp, count)` pairs; merge hits at the same timestamp
> - `hit(t)`: if last entry has same timestamp, increment count; else append new entry
> - `getHits(t)`: evict entries where `timestamp <= t - 300`; return sum of counts
> - Complexity: O(1) amortized hit, O(n) getHits
>
> **Circular buffer (O(1) space follow-up):**
> - Two arrays of size 300: `times[300]`, `hits[300]`
> - `hit(t)`: `idx = t % 300`; if `times[idx] == t`: increment; else reset both
> - `getHits(t)`: sum `hits[i]` where `times[i] > t - 300`
> - Complexity: O(1) hit, O(300) = O(1) getHits, O(1) space
>
> **Key insight for circular buffer:** `times[idx]` tells you which 300-second cycle this slot belongs to. Overwriting an old slot is safe because any old entry more than 300 seconds ago is outside the window.
>
> *See also:* [[dsa/problems/design-hit-counter]], [[dsa/patterns/sliding-window]]
