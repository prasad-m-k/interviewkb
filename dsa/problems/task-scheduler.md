# Task Scheduler

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** Greedy + Heap
**Companies:** Facebook, Google, Amazon

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a list of CPU tasks (each represented by a character) and a cooldown integer `n`, find the minimum number of intervals required to finish all tasks. The CPU can execute one task per interval or be idle. The same task must wait at least `n` intervals before being executed again.

Example: `tasks = ['A','A','A','B','B','B']`, `n = 2` → `8`
(A → B → idle → A → B → idle → A → B)

## Approach
**Greedy insight:** always execute the most frequent remaining task to minimize wasted idle time. After executing a task, it becomes unavailable for `n` intervals. Use a max-heap (by count) to always pick the most frequent task, and a cooldown queue to track when tasks become available again.

**Simulation with heap + queue:**
1. Count task frequencies. Push all `(-count, task)` to a max-heap.
2. Maintain a queue of `(count, available_at_time)` for cooling-down tasks.
3. At each time tick:
   - Re-enqueue any task whose cooldown has expired.
   - If the heap is non-empty, pop the most frequent task, execute it, decrement its count, and push it to the cooldown queue.
   - If the heap is empty, the CPU is idle for this interval.

## Solution (Python)
```python
import heapq
from collections import Counter, deque
from typing import List

def leastInterval(tasks: List[str], n: int) -> int:
    count = Counter(tasks)
    max_heap = [-c for c in count.values()]
    heapq.heapify(max_heap)

    time = 0
    cooldown = deque()   # (neg_count, available_at)

    while max_heap or cooldown:
        time += 1
        if max_heap:
            cnt = 1 + heapq.heappop(max_heap)   # execute: decrement count
            if cnt < 0:   # tasks remaining
                cooldown.append((cnt, time + n))
        if cooldown and cooldown[0][1] == time:
            heapq.heappush(max_heap, cooldown.popleft()[0])

    return time
```

## Complexity
Time: O(n_tasks × n) in the worst case | Space: O(1) since there are at most 26 unique task types

## Key insight
The greedy choice is: always execute the task with the highest remaining count. This minimizes idle time because the most frequent task is the bottleneck — delaying it wastes more future intervals. The heap gives O(log 26) = O(1) access to the most frequent task.

**Math shortcut (for follow-up):**
`result = max(len(tasks), (max_freq - 1) * (n + 1) + count_of_max_freq_tasks)`

This formula gives the answer directly without simulation. Know it as a follow-up when the interviewer asks for O(1) space and O(n) time.

## Variants / follow-ups
- **Rearrange String k Distance Apart**: same problem on a string.
- **Reorganize String**: special case where `n = 1`.
- **Interviewers ask**: "Can you solve it in O(n) without simulation?" → yes, use the formula above.

## MLOps connection
ML pipeline job scheduling: training jobs for different model versions need cooldown periods (GPU memory flush, artifact cleanup, distributed lock release). Scheduling to minimize total wall time while respecting per-job cooldowns is the exact same problem.

## Sources
- [[DSA overview]]
