# Merge Intervals

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** Sort + Greedy Scan
**Companies:** Apple, Google, Facebook, Amazon, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an array of intervals `[start, end]`, merge all overlapping intervals and return the non-overlapping result.

Example: `[[1,3],[2,6],[8,10],[15,18]]` → `[[1,6],[8,10],[15,18]]`

Two intervals overlap if `b.start <= a.end` (their start/end interleave or touch).

## Approach
Sorting by start time means all potentially overlapping intervals are adjacent. A single left-to-right scan then either extends the last merged interval or starts a new one.

1. Sort intervals by start time.
2. Initialize `merged` with the first interval.
3. For each subsequent interval `[start, end]`:
   - If `start <= merged[-1][1]` (overlaps or touches): extend end to `max(merged[-1][1], end)`
   - Otherwise: append as a new interval.

The `max` in the extend step handles fully-contained intervals: `[1,10]` followed by `[2,5]` should stay `[1,10]`, not shrink to `[1,5]`.

## Solution (Python)
```python
from typing import List

def merge(intervals: List[List[int]]) -> List[List[int]]:
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]

    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])

    return merged
```

## Complexity
Time: O(n log n) for sorting | Space: O(n) for output

## Key insight
`max(merged[-1][1], end)` handles the containment case: if the current interval is fully inside the last merged one (e.g., `[1,10]` + `[2,5]`), you must not shrink the end. Without `max`, you'd incorrectly output `[1,5]`.

## Variants / follow-ups
- **Insert Interval**: insert a new interval into a sorted non-overlapping list and merge — binary search for position, then merge left and right neighbors.
- **Meeting Rooms I**: can one person attend all meetings? → Sort by start, check if any `intervals[i].start < intervals[i-1].end`.
- **Meeting Rooms II**: minimum rooms needed → use a min-heap of end times; count concurrent overlaps.
- **Employee Free Time**: merge all employee schedules, find gaps — same merge pattern, then invert.
- **Interviewers ask**: "What if the input is already sorted?" → skip the sort, O(n). "What if intervals arrive as a stream?" → maintain a sorted structure (SortedList / balanced BST) and merge on insert.

## MLOps connection
Merging time windows: if multiple monitoring alerts fire for overlapping time ranges, deduplicate into consolidated incident windows. Also: computing non-overlapping training/validation time splits from overlapping data collection windows.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
