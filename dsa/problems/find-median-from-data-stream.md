# Find Median from Data Stream

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/two-heaps]]
**Companies:** Google, Amazon, Meta, Apple

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Design a data structure that supports:
- `addNum(num)` — add an integer from a data stream
- `findMedian()` — return the median of all integers added so far

The median is the middle value if odd count, or the average of the two middle values if even count.

## Approach
Sorting all numbers on each query is O(n log n) — too slow for a stream. We need O(log n) add and O(1) find.

Use the **two-heap pattern**:
- `lo`: max-heap holding the **lower half** of numbers (negate values for Python's min-heap)
- `hi`: min-heap holding the **upper half** of numbers

**Invariant after every `addNum`:**
1. Every element in `lo` ≤ every element in `hi`
2. `len(lo) - len(hi)` is 0 or 1

**`addNum` procedure:**
1. Push `num` to `lo` (max-heap).
2. Pop max from `lo`, push it to `hi` (this enforces the ordering invariant).
3. If `hi` is larger than `lo`, pop min from `hi` and push back to `lo` (enforce size invariant).

**`findMedian`:** if sizes are equal → average of both tops; else → top of `lo`.

## Solution (Python)
```python
import heapq

class MedianFinder:
    def __init__(self):
        self.lo = []   # max-heap (lower half) — store negated values
        self.hi = []   # min-heap (upper half)

    def addNum(self, num: int) -> None:
        heapq.heappush(self.lo, -num)
        # Move lo's max to hi (maintains ordering invariant)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        # Re-balance: lo must be >= hi in size
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))

    def findMedian(self) -> float:
        if len(self.lo) > len(self.hi):
            return float(-self.lo[0])
        return (-self.lo[0] + self.hi[0]) / 2.0
```

## Complexity
Time: O(log n) addNum, O(1) findMedian | Space: O(n)

## Key insight
Always push to `lo` first, then immediately pop from `lo` to `hi`. This single step guarantees the ordering invariant in one move regardless of whether `num` belongs in the lower or upper half. Without this, you'd need a conditional check.

## Variants / follow-ups
- **Sliding Window Median**: maintain the two-heap structure with a lazy-deletion map to handle evictions as the window slides. Much harder.
- **Find Percentile (not just median)**: generalize the size ratio. For p90, keep 90% of elements in `lo` and 10% in `hi`.
- **Interviewers ask**: "What if the stream contains a lot of duplicates?" → Same approach; no change needed. "What if numbers are always in a bounded range, like [0, 100]?" → Use a frequency array for O(1) add and O(range) median.

## MLOps connection
P50 latency monitoring. Streaming percentile computation for model inference latency, feature value distributions, or prediction confidence scores — all need efficient running-quantile structures. The two-heap pattern is the interview representation of reservoir sampling or sketch-based approaches used in production.

## Sources
- [[DSA overview]]
