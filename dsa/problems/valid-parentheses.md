# Valid Parentheses

**Difficulty:** Easy
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/concepts/stack]] (matching stack)
**Companies:** Amazon, Google, Meta, Bloomberg, Adobe

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given a string `s` containing only `'('`, `')'`, `'{'`, `'}'`, `'['`, `']'`, determine if the input string is valid. A string is valid if every open bracket is closed by the same type of bracket in the correct order.

```
"()"       → true
"()[]{}"   → true
"(]"       → false
"([)]"     → false
"{[]}"     → true
```

## Approach

Matching stack: push every opening bracket. On a closing bracket, pop the top and check it's the matching opener. If the stack is empty at a closing bracket (nothing to match), or the top doesn't match, return false. At the end, the stack must be empty (no unmatched openers).

## Solution (Python)

```python
def isValid(s: str) -> bool:
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}

    for ch in s:
        if ch in '([{':
            stack.append(ch)
        else:                              # closing bracket
            if not stack or stack[-1] != pairs[ch]:
                return False
            stack.pop()

    return not stack                       # true iff all openers were matched
```

## Why a stack and not a counter?

A counter works for a single bracket type (`()`). With multiple types you need to know *which* opener is unmatched. Example: `([)]` has equal counts of each type but is invalid because the nesting order is wrong. Only a stack preserves the nesting order.

## Complexity

Time: O(n) | Space: O(n)

## Variants and the patterns they introduce

### Minimum Add to Make Valid (LC 921)
Count unmatched openers and closers:
```python
def minAddToMakeValid(s: str) -> int:
    open_count = 0   # unmatched '('
    close_needed = 0 # unmatched ')'
    for ch in s:
        if ch == '(':
            open_count += 1
        elif open_count > 0:
            open_count -= 1  # matched
        else:
            close_needed += 1  # unmatched ')'
    return open_count + close_needed
```

### Minimum Remove to Make Valid (LC 1249)
Track indices of unmatched openers; remove them and unmatched closers:
```python
def minRemoveToMakeValid(s: str) -> str:
    s = list(s)
    stack = []          # indices of unmatched '('
    for i, ch in enumerate(s):
        if ch == '(':
            stack.append(i)
        elif ch == ')':
            if stack:
                stack.pop()
            else:
                s[i] = ''    # unmatched closer → remove
    for i in stack:
        s[i] = ''            # unmatched openers → remove
    return ''.join(s)
```

### Score of Parentheses (LC 856)
Assign numeric scores to nested brackets:
```python
def scoreOfParentheses(s: str) -> int:
    stack = [0]      # stack of partial scores at each nesting level
    for ch in s:
        if ch == '(':
            stack.append(0)
        else:
            v = stack.pop()
            stack[-1] += max(2 * v, 1)   # empty pair=1, wrapped pair=2×inner
    return stack[0]
```

### Longest Valid Parentheses (LC 32) — Hard
Store indices. When a valid pair closes, the longest run extends back to the index before the new stack top:
```python
def longestValidParentheses(s: str) -> int:
    stack = [-1]     # sentinel
    best = 0
    for i, ch in enumerate(s):
        if ch == '(':
            stack.append(i)
        else:
            stack.pop()
            if not stack:
                stack.append(i)    # new base
            else:
                best = max(best, i - stack[-1])
    return best
```

## Key insight

The stack encodes **nesting depth and order simultaneously**. A flat counter encodes depth only. Any problem about correct nesting order needs a stack, not a counter.

## Sources

- [[dsa/concepts/stack]]
- [[dsa/patterns/monotonic-stack]]
