# Product of Array Except Self

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** Prefix / Suffix Pass
**Companies:** Apple, Amazon, Facebook, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an integer array `nums`, return an array `answer` where `answer[i]` is the product of all elements except `nums[i]`.

You must solve it in O(n) time **without using division**. The follow-up asks for O(1) extra space (output array doesn't count).

Example: `nums = [1,2,3,4]` → `[24,12,8,6]`

## Approach
For each position `i`, the result is `prefix_product(0..i-1) × suffix_product(i+1..n-1)`.

**Two-pass O(n) time, O(1) extra space:**
1. **Left pass**: fill `result[i]` with the product of everything to the left of `i`. Initialize with `prefix = 1`; for each `i`, `result[i] = prefix`, then `prefix *= nums[i]`.
2. **Right pass**: multiply `result[i]` by the product of everything to the right. Initialize `suffix = 1`; traverse right-to-left; `result[i] *= suffix`, then `suffix *= nums[i]`.

No division needed. The output array counts as O(1) extra space per the problem's convention.

## Solution (Python)
```python
from typing import List

def productExceptSelf(nums: List[int]) -> List[int]:
    n = len(nums)
    result = [1] * n

    # Left pass: result[i] = product of all elements to the left of i
    prefix = 1
    for i in range(n):
        result[i] = prefix
        prefix *= nums[i]

    # Right pass: multiply by product of all elements to the right of i
    suffix = 1
    for i in range(n - 1, -1, -1):
        result[i] *= suffix
        suffix *= nums[i]

    return result
```

## Complexity
Time: O(n) | Space: O(1) extra (two scalar variables; output array not counted)

## Key insight
After the left pass, `result[i]` holds the left product. After the right pass (multiplying in-place), `result[i]` holds `left[i] × right[i]` — which is the desired product-except-self. The two single-pass scans are read once, write once — no temporary arrays needed.

## Variants / follow-ups
- **Handle zeros**: if `nums` contains zeros, division-based approaches break. This prefix/suffix approach handles zeros naturally — no special cases needed.
- **More than one zero**: if two or more zeros exist, every result is 0 except possibly the positions of the zeros themselves — same algorithm still gives the right answer.
- **Product of Array With Division**: `result[i] = total_product / nums[i]` — but fails if `nums[i] == 0`. The prefix/suffix approach avoids this entirely.
- **Interviewers ask**: "What if integer overflow is a concern?" → Use modular arithmetic or switch to float; discuss tradeoffs.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
