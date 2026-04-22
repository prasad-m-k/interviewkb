# Two Heaps

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/heap]]

## What it solves
Problems that require tracking the **median** of a dynamic stream, or partitioning data into two halves and querying the boundary. The two-heap pattern maintains a max-heap for the lower half and a min-heap for the upper half, keeping them balanced so the median is always available in O(1).

## Template / skeleton

```python
import heapq

class TwoHeaps:
    def __init__(self):
        self.lo = []   # max-heap (lower half) — negate values
        self.hi = []   # min-heap (upper half)

    def add(self, num):
        # Step 1: always push to lo first
        heapq.heappush(self.lo, -num)
        # Step 2: rebalance — move lo's max to hi
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        # Step 3: keep lo.size >= hi.size (lo can be 1 larger)
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))

    def median(self):
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2.0
```

**Invariant after every `add`:**
- `lo` contains the smaller half; `hi` contains the larger half
- `len(lo) - len(hi)` is 0 or 1
- Every element in `lo` ≤ every element in `hi`
- Median = `lo[0]` (odd total) or `(lo[0] + hi[0]) / 2` (even total)

## Signal phrases
- "find the median of a data stream"
- "sliding window median"
- "median as elements are added"
- "maintain running statistics on a stream"

## Complexity
| Operation | Time |
|---|---|
| Add number | O(log n) |
| Find median | O(1) |
| Space | O(n) |

## MLOps connection
Running median is a useful statistic for production monitoring (latency p50, feature value drift). The two-heap structure is the interview representation of a streaming quantile estimator.

## Problems using this pattern
- [[dsa/problems/find-median-from-data-stream]]

## Sources
- [[DSA overview]]
