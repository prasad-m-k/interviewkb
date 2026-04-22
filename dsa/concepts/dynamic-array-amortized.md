# Dynamic Array — Amortized O(1) Append

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/hash-map-internals]]

## What it is
A dynamic array (Python `list`, Java `ArrayList`, C++ `vector`) provides O(1) random access like a static array, but also grows automatically when full. The key claim: `append` is **O(1) amortized** — not O(1) worst case. Interviewers test whether you understand this distinction and *why* it holds.

## How growth works
The array starts with a small fixed capacity (e.g., 4). When it's full and you append:
1. Allocate a new array of 2× the current capacity
2. Copy all existing elements to the new array — **O(n) for this one operation**
3. Insert the new element
4. Discard the old array

```
capacity 4, size 4 → append → new capacity 8, copy 4 elements, insert → size 5
capacity 8, size 8 → append → new capacity 16, copy 8 elements, insert → size 9
```

## Why it's O(1) amortized — the proof

**Amortized analysis**: spread the cost of expensive operations over all operations. The "bankers" intuition: every cheap append "saves" credit; that credit pays for the eventual expensive copy.

With doubling, a copy of n elements only happens after n/2 cheap appends preceded it (the array was half-full when it last doubled). So:

```
Total work to insert n elements:
  n cheap inserts:       O(n)
  Copy costs at each resize: 1 + 2 + 4 + 8 + ... + n/2 + n = 2n − 1 = O(n)
  Total: O(n)
  Per operation: O(n) / n = O(1) amortized
```

The geometric series 1 + 2 + 4 + ... + n = 2n − 1 is the key. It converges because doubling means each resize is more expensive than all previous resizes *combined*.

**Why 2× and not 1.5×?** Both give O(1) amortized. Python actually uses ~1.125× growth for small arrays and up to ~1.5× for larger ones — a tradeoff between fewer copies (larger growth) and less wasted memory (smaller growth). Java `ArrayList` uses exactly 1.5×.

## Worst case vs amortized
| Operation | Amortized | Worst case (single op) |
|---|---|---|
| `append` | O(1) | O(n) — when resize triggers |
| `get(i)` | O(1) | O(1) |
| `insert(i, val)` | O(n) | O(n) — shift elements |
| `delete(i)` | O(n) | O(n) — shift elements |

The O(n) worst-case append **matters in latency-sensitive systems** (game loops, real-time trading). Fix: pre-allocate with known size (`list = [None] * n` in Python, `ArrayList(n)` in Java).

## Python list memory layout
CPython over-allocates to smooth out resizes. After an append that triggers growth, the allocated capacity is always larger than the size:

```python
import sys
a = []
for i in range(9):
    a.append(i)
    print(f"size={len(a)}, allocated≈{sys.getsizeof(a)}")
# Allocated jumps at 0, 4, 8, 16 — roughly doubling
```

## Common interview angles
- "What is the time complexity of appending to a Python list?" — O(1) amortized, O(n) worst case; explain the doubling argument
- "Why is inserting at the front of a list O(n)?" — all elements must shift right; no amortization applies
- "When would you avoid using a dynamic array?" — when you need guaranteed O(1) worst-case inserts (use a linked list or pre-allocated fixed array)
- "How do you pre-allocate a Python list?" — `[None] * n` or `list = []; list.extend([None] * n)` — prevents all resize copies

## Sources
*(none yet)*
