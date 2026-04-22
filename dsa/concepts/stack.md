# Stack

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/deque]], [[dsa/patterns/monotonic-stack]]

## What it is

A stack is a LIFO (Last In, First Out) data structure. Push adds to the top; pop removes from the top. Both are O(1). In Python, a plain list serves as a stack: `stack.append(x)` / `stack.pop()`.

## The Four Stack Modes in Interviews

| Mode | What the stack holds | Signal phrase | Canonical problem |
|---|---|---|---|
| **Monotonic** | indices/values in sorted order | "next greater", "previous smaller", "span", "histogram" | Daily Temperatures, Largest Rectangle |
| **Matching** | unmatched opening tokens | "valid", "balanced", "well-formed" | Valid Parentheses |
| **Simulation** | live entities that collide or cancel | "collision", "asteroid", "expression" | Asteroid Collision |
| **Expression eval** | operands (and sometimes operators) | "evaluate", "calculator", "RPN" | Evaluate RPN, Basic Calculator |

---

## Mode 1: Monotonic Stack

Maintain the stack in strictly increasing or decreasing order. Before pushing, pop all elements that violate the order. The moment you would pop element `j` to make room for `i`, you've found `j`'s answer (`i` is `j`'s next-greater, or `j` is `i`'s previous-smaller, etc.).

**Decreasing stack → answers "next greater element"**  
**Increasing stack → answers "next smaller element" / "span"**

Full template and problems: [[dsa/patterns/monotonic-stack]]

---

## Mode 2: Matching Stack

Push opening tokens. On a closing token, pop and check it matches. If it doesn't, or the stack is empty, invalid.

```python
def isValid(s: str) -> bool:
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    for ch in s:
        if ch in '([{':
            stack.append(ch)
        elif not stack or stack[-1] != pairs[ch]:
            return False
        else:
            stack.pop()
    return not stack
```

**Key:** the stack holds unmatched openers. At the end, if the stack is empty every opener was matched.

---

## Mode 3: Simulation Stack

Entities on a stack interact with the incoming element (collide, cancel, override). Process left-to-right; the stack always contains "surviving" entities so far.

```python
# Asteroid Collision skeleton
stack = []
for val in asteroids:
    while stack and val < 0 < stack[-1]:   # right-moving meets left-moving
        if stack[-1] < -val:
            stack.pop(); continue          # top explodes, keep going
        elif stack[-1] == -val:
            stack.pop()                    # both explode
        break                             # top survives
    else:
        stack.append(val)
```

---

## Mode 4: Expression Evaluation

Two-stack approach (operators + operands), or single operand stack for postfix (RPN):

```python
# RPN (postfix) — single stack
def evalRPN(tokens):
    stack = []
    ops = {'+': lambda a,b: a+b, '-': lambda a,b: a-b,
           '*': lambda a,b: a*b, '/': lambda a,b: int(a/b)}
    for t in tokens:
        if t in ops:
            b, a = stack.pop(), stack.pop()
            stack.append(ops[t](a, b))
        else:
            stack.append(int(t))
    return stack[0]
```

---

## Complexity

All stack operations are O(1) amortized. For monotonic stack problems: each element is pushed and popped at most once → O(n) total even though there's a nested loop.

## When to use

- You need to efficiently find the **nearest larger/smaller** element to the left or right → monotonic stack
- You need to validate or reconstruct **nested/paired structure** → matching stack
- Elements **cancel or consume** each other based on sign or type → simulation stack
- You're evaluating **infix/postfix expressions** → eval stack

## Sources

- [[dsa/patterns/monotonic-stack]]
- [[dsa/topics/data-structures]]
