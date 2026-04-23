# Binary Search

**Topic:** [[dsa/topics/algorithms]]
**Related:** [[dsa/patterns/bfs-dfs]], [[dsa/concepts/dynamic-programming]]

## What it is

Binary search is an O(log n) algorithm for finding a target value in a **sorted** collection by repeatedly halving the search space. Each comparison eliminates half the remaining candidates.

Binary search appears in two distinct forms in interviews:
1. **Classic:** Find an element in a sorted array.
2. **Search-on-answer:** Binary search over the *result space* to find the minimum/maximum value satisfying a condition.

## How it works

The invariant: at every step, the answer (if it exists) lies within `[left, right]`.

```python
def binary_search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2  # avoids integer overflow (critical in C++)
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1  # not found
```

**Why `left + (right - left) // 2` instead of `(left + right) // 2`?**
In Python, integers don't overflow, so both work. In C++/Java, `left + right` can overflow `int` if both are near `INT_MAX`. The safe form is always correct.

## The Three Templates

### Template 1: Exact Match
Use when: searching for a specific value that exists in the array.
- Loop condition: `left <= right`
- Terminates with `left > right`

```python
while left <= right:
    mid = left + (right - left) // 2
    if nums[mid] == target: return mid
    elif nums[mid] < target: left = mid + 1
    else: right = mid - 1
return -1
```

### Template 2: Left Boundary (first occurrence / minimum valid)
Use when: finding the *leftmost* position where a condition becomes true.
- Returns the first index where `nums[i] >= target`
- "Find insertion position" problems

```python
def left_bound(nums, target):
    left, right = 0, len(nums)  # note: right = len(nums), not len-1
    while left < right:         # note: strict <, not <=
        mid = left + (right - left) // 2
        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid         # shrink from right
    return left                 # left == right; the answer

# After loop: left is the index of first element >= target
# If left == len(nums), target is greater than all elements
```

### Template 3: Right Boundary (last occurrence / maximum valid)
Use when: finding the *rightmost* position where a condition is true.

```python
def right_bound(nums, target):
    left, right = 0, len(nums)
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] <= target:
            left = mid + 1      # shrink from left
        else:
            right = mid
    return left - 1             # last position where nums[i] <= target
```

## Search on Answer (Binary Search the Result Space)

When a problem asks for the *minimum/maximum X such that some condition holds*, binary search the answer space instead of the array.

**Pattern:**
```python
def can_satisfy(value, *args) -> bool:
    # Returns True if 'value' satisfies the condition
    ...

def solve():
    left, right = min_possible, max_possible
    while left < right:
        mid = left + (right - left) // 2
        if can_satisfy(mid, ...):
            right = mid       # try smaller (minimize)
            # or: left = mid + 1  (maximize)
        else:
            left = mid + 1
    return left
```

**Classic examples:**
- "Koko Eating Bananas" — minimize eating speed so all bananas eaten in H hours
- "Capacity to Ship Packages in D Days" — minimize ship weight
- "Median of Two Sorted Arrays" — binary search on partition point

## Complexity

| | Time | Space |
|---|---|---|
| Classic binary search | O(log n) | O(1) |
| With O(n) feasibility check | O(n log(range)) | O(1) |

## When to use

| Signal | Use this form |
|---|---|
| "sorted array", "find target" | Template 1 or 2 |
| "first/last occurrence", "leftmost/rightmost" | Template 2/3 |
| "minimum X such that..." | Search-on-answer, minimize: `right = mid` |
| "maximum X such that..." | Search-on-answer, maximize: `left = mid + 1` |
| "rotated sorted array" | Modified Template 1 with pivot detection |

## Common Interview Angles

### 1. Rotated Sorted Array
Pivot is the one place where `nums[mid] > nums[right]`. Use this to determine which half is sorted, then decide which half to search.

```python
def search_rotated(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        if nums[left] <= nums[mid]:          # left half is sorted
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:                                 # right half is sorted
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1
```

### 2. Find Peak Element
An element is a peak if `nums[i] > nums[i-1]` and `nums[i] > nums[i+1]`. Always move toward the larger neighbor.

```python
def find_peak(nums):
    left, right = 0, len(nums) - 1
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] < nums[mid + 1]:
            left = mid + 1   # ascending slope; peak is to the right
        else:
            right = mid      # descending slope; peak is at mid or left
    return left
```

### 3. Search in 2D Matrix
A matrix where each row is sorted and the first element of each row > last element of previous row is essentially a 1D sorted array. Map `mid → (mid // cols, mid % cols)`.

```python
def search_matrix(matrix, target):
    m, n = len(matrix), len(matrix[0])
    left, right = 0, m * n - 1
    while left <= right:
        mid = left + (right - left) // 2
        val = matrix[mid // n][mid % n]
        if val == target: return True
        elif val < target: left = mid + 1
        else: right = mid - 1
    return False
```

## Examples

**Example 1:** `nums = [1, 3, 5, 7, 9]`, target = 5
- left=0, right=4, mid=2 → `nums[2]=5` → return 2

**Example 2 (left boundary):** `nums = [1, 2, 2, 2, 3]`, target = 2
- Finds index 1 (first occurrence of 2)

**Example 3 (search on answer):** Ship packages `[1,2,3,4,5,6,7,8,9,10]` in 5 days.
- Search space: `[max(packages), sum(packages)]` = `[10, 55]`
- Binary search capacity; for each capacity, simulate days needed with greedy.

## Sources
- [[dsa/topics/algorithms]]
- [[dsa/companies/google]]
- [[dsa/companies/apple]]
