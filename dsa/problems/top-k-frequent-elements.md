# Top K Frequent Elements

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** Heap / Bucket Sort
**Companies:** Amazon, Google, Meta, Facebook

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an integer array `nums` and an integer `k`, return the `k` most frequent elements. You may return the answer in any order.

The answer is guaranteed to be unique. You must solve it in better than O(n log n).

## Approach

**Approach 1 — Min-Heap O(n log k):**
1. Count frequencies with `Counter`.
2. Use a min-heap of size `k`: push `(freq, num)` pairs; when size exceeds `k`, pop the minimum frequency.
3. Return remaining elements in the heap.

**Approach 2 — Bucket Sort O(n):**
1. Count frequencies.
2. Create `buckets` where `buckets[freq]` = list of numbers with that frequency. Array size is `len(nums) + 1` (max possible frequency).
3. Iterate buckets from highest to lowest frequency, collecting elements until k are gathered.

Bucket sort is optimal and often impresses interviewers. Lead with it if you can explain it clearly.

## Solution (Python)
```python
from collections import Counter
from typing import List

# O(n log k) — heap approach
def topKFrequent_heap(nums: List[int], k: int) -> List[int]:
    count = Counter(nums)
    return [num for num, _ in count.most_common(k)]
    # Or manually: heapq.nlargest(k, count, key=count.get)

# O(n) — bucket sort approach
def topKFrequent(nums: List[int], k: int) -> List[int]:
    count = Counter(nums)
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in count.items():
        buckets[freq].append(num)

    result = []
    for freq in range(len(buckets) - 1, 0, -1):
        for num in buckets[freq]:
            result.append(num)
            if len(result) == k:
                return result
    return result
```

## Complexity
Heap: Time O(n log k), Space O(n) | Bucket Sort: Time O(n), Space O(n)

## Key insight
The bucket sort works because frequency is bounded by `len(nums)` — there are at most `n` distinct possible frequency values. This bounded domain is what allows sorting in O(n) rather than O(n log n).

## Variants / follow-ups
- **Top K Frequent Words**: same approach, but break ties lexicographically — use `(-freq, word)` as the heap key.
- **K Closest Points to Origin**: heap keyed on distance.
- **Kth Largest in a Stream**: maintain a min-heap of size k; the root is always the Kth largest.
- **Interviewers ask**: "How would you handle an infinite stream efficiently?" → Approximate with Count-Min Sketch; exact Top-K requires keeping the full frequency map.

## MLOps connection
Feature importance ranking after computing SHAP values. Also applies to most frequent prediction classes, most common error types in model output logs, or most queried features in a feature store.

## Sources
- [[DSA overview]]
