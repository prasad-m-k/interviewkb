# Design Hit Counter

**Difficulty:** Medium
**Topic:** [[dsa/topics/data-structures]]
**Pattern:** [[dsa/patterns/sliding-window]]
**Companies:** Google, Amazon, Dropbox

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Design a hit counter that counts the number of hits received in the past 5 minutes (300 seconds).

Implement:
- `hit(timestamp)` — record a hit at the given timestamp (in seconds)
- `getHits(timestamp)` — return the number of hits in the interval `[timestamp - 299, timestamp]`

Note: timestamps are called in non-decreasing order. Multiple hits can have the same timestamp.

## Approach
This is a sliding window over timestamps. We need to count events within a 300-second window.

**Approach 1 — Deque of (timestamp, count) pairs:**
- Merge hits at the same timestamp into one entry (saves space for bursty traffic).
- On `getHits`, evict entries older than `timestamp - 300` from the front.
- Return the sum of all counts remaining in the deque.

**Approach 2 — Circular buffer (array of size 300):**
- Store hits in two arrays: `times[300]` (timestamp mod 300) and `hits[300]` (count).
- On `hit(t)`: if `times[t % 300] == t`, increment `hits[t % 300]`; else reset both (new cycle).
- On `getHits(t)`: sum slots where `times[slot] > t - 300`.
- O(300) = O(1) space, O(1) hit, O(300) = O(1) getHits.

The circular buffer approach is the one interviewers love for the follow-up: "optimize for space."

## Solution (Python)
```python
from collections import deque

# Approach 1: deque (clear, easy to explain)
class HitCounter:
    def __init__(self):
        self.hits = deque()   # (timestamp, count)

    def hit(self, timestamp: int) -> None:
        if self.hits and self.hits[-1][0] == timestamp:
            t, c = self.hits.pop()
            self.hits.append((t, c + 1))
        else:
            self.hits.append((timestamp, 1))

    def getHits(self, timestamp: int) -> int:
        while self.hits and self.hits[0][0] <= timestamp - 300:
            self.hits.popleft()
        return sum(c for _, c in self.hits)

# Approach 2: circular buffer (space-optimal follow-up)
class HitCounterCircular:
    def __init__(self):
        self.times = [0] * 300
        self.hits = [0] * 300

    def hit(self, timestamp: int) -> None:
        idx = timestamp % 300
        if self.times[idx] == timestamp:
            self.hits[idx] += 1
        else:
            self.times[idx] = timestamp
            self.hits[idx] = 1

    def getHits(self, timestamp: int) -> int:
        total = 0
        for i in range(300):
            if self.times[i] > timestamp - 300:
                total += self.hits[i]
        return total
```

## Complexity
Deque: O(1) amortized hit, O(n) getHits | Circular: O(1) hit, O(300) = O(1) getHits, O(300) = O(1) space

## Key insight
The circular buffer maps timestamp → slot via `timestamp % 300`. When a new timestamp maps to the same slot as an old one, you overwrite — which is correct, because an old entry from 5+ minutes ago is outside the window anyway. This is the key invariant: `times[idx]` tells you which cycle this slot belongs to.

## Variants / follow-ups
- **Rate Limiter**: same structure — count hits per user per window; reject if count > threshold.
- **Log Rate Limiting**: "at most N log messages per second per source."
- **Interviewers ask**: "What if the window isn't exactly 300 seconds?" → Parameterize the array size. "What about concurrent hits?" → Lock per slot (fine-grained) or a single lock (simple).

## MLOps connection
Inference API rate limiting: count requests per user in the past 5 minutes; reject if over quota. Also: monitoring — count model prediction errors in the last 5 minutes to trigger an alert. The circular buffer is exactly how high-throughput rate limiters are implemented in practice (e.g., Redis' fixed-window counter).

## Sources
- [[DSA overview]]
