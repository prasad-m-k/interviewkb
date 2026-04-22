# Merge K Sorted Lists

**Difficulty:** Hard
**Topic:** [[dsa/topics/data-structures]]
**Pattern:** Min Heap
**Companies:** Amazon, Google, Meta, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an array of `k` linked lists, each sorted in ascending order, merge all the lists into one sorted linked list and return it.

## Approach
Merging two sorted lists is O(n). Merging k lists naively (one by one) is O(n*k). 

**Min-heap approach O(N log k):**
At each step, we only need the current minimum across all list heads. Push all k heads into a min-heap. Pop the minimum, append to result, and push that node's `next` (if it exists). The heap always holds at most k elements.

Use a tuple `(val, list_index, node)` in the heap. The `list_index` tiebreaker avoids comparing `ListNode` objects (which Python can't do without `__lt__`).

## Solution (Python)
```python
import heapq
from typing import List, Optional

class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def mergeKLists(lists: List[Optional[ListNode]]) -> Optional[ListNode]:
    heap = []
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))

    dummy = ListNode(0)
    curr = dummy

    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next
```

## Complexity
Time: O(N log k) where N = total nodes across all lists, k = number of lists | Space: O(k) for the heap

## Key insight
The heap never grows beyond size k. Each node enters and exits the heap exactly once. The `list_index` as a tiebreaker in the tuple is critical — Python will try to compare the `ListNode` objects if the first element (val) is equal, which raises a TypeError.

## Variants / follow-ups
- **Merge K Sorted Arrays**: same algorithm; use `(val, array_index, element_index)` tuples.
- **Smallest Range Covering Elements from K Lists**: extend this — track the current range `[min, max]` as you pop and push; update range when current range improves.
- **External Sort**: merging k sorted file chunks on disk is the same problem; the heap is loaded into memory, the lists are file streams.
- **Interviewers ask**: "Can you do this without extra space?" → Divide and conquer: recursively merge pairs of lists. O(N log k) time, O(log k) recursion space.

## MLOps connection
Merging sorted log or event streams from multiple distributed training workers or serving replicas — each produces a sorted stream by timestamp; the heap gives the globally ordered merge. Also: external sort in data pipelines when a large dataset is split across files.

## Sources
- [[DSA overview]]
