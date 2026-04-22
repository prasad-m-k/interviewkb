# Asteroid Collision

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/concepts/stack]] (simulation stack)
**Companies:** Amazon, Google, Bloomberg

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

An array `asteroids` represents asteroids in a row. Each asteroid's absolute value is its size; the sign is its direction (`+` = right, `-` = left). All move at the same speed.

When two asteroids meet (a right-moving and a left-moving), the smaller one explodes. If equal size, both explode. Two asteroids moving in the same direction never collide.

Return the state of the asteroids after all collisions.

```
[5, 10, -5]     → [5, 10]      (−5 destroyed by 10)
[8, -8]         → []           (equal, both destroyed)
[10, 2, -5]     → [10]         (−5 destroyed by 2? no: 2 < 5, 2 explodes; then 10 > 5, −5 explodes)
[-2, -1, 1, 2]  → [-2,-1,1,2]  (no collision: left-movers pass right-movers going same direction)
```

## Approach

Simulate left-to-right. The stack holds "surviving" asteroids so far.

A collision only happens when the stack top is positive (moving right) **and** the new asteroid is negative (moving left). Keep resolving collisions with the stack top until one of:
1. Stack is empty or top is also negative → new asteroid survives, push it
2. Top equals `−new` in magnitude → both explode, don't push
3. Top is larger → new asteroid explodes, stop (top survives)

## Solution (Python)

```python
def asteroidCollision(asteroids: list[int]) -> list[int]:
    stack = []

    for asteroid in asteroids:
        alive = True

        while alive and asteroid < 0 and stack and stack[-1] > 0:
            if stack[-1] < -asteroid:
                stack.pop()          # top explodes, new one keeps going
            elif stack[-1] == -asteroid:
                stack.pop()          # both explode
                alive = False
            else:
                alive = False        # top survives, new one explodes

        if alive:
            stack.append(asteroid)

    return stack
```

## State diagram

```
New asteroid:  positive (+)    negative (−)
               ─────────────────────────────
Stack top +    no collision     COLLISION
Stack top −    no collision     no collision (both going left)
Stack empty    push             push
```

Collision resolution loop only triggers in the top-right cell.

## Traced execution

```
asteroids = [10, 2, -5]

asteroid=10: stack=[], push → stack=[10]
asteroid=2:  top=10 > 0, new=2 > 0 → no collision, push → stack=[10,2]
asteroid=-5: top=2 > 0, new=-5 < 0 → COLLISION
    2 < 5 → pop 2 (2 explodes), keep going
    top=10 > 0, new=-5 < 0 → COLLISION
    10 > 5 → alive=False (-5 explodes)
No push. stack=[10]

Result: [10]  ✓
```

## Complexity

Time: O(n) — each asteroid pushed and popped at most once  
Space: O(n)

## Key insight

The stack holds "right-moving survivors" at the front. A left-mover has to defeat every right-mover ahead of it before it can rest. This is O(n) not O(n²) because each asteroid is processed at most twice (once pushed, once popped).

The `alive` flag replaces a `break` that would need a second `push` check. It makes the "new asteroid also explodes" case clean.

## Variants / follow-ups

- **Count survivors:** `len(asteroidCollision(asteroids))`
- **Multiple collision directions / gravity:** model as a more complex simulation
- **Remove All Adjacent Duplicates (LC 1047):** same simulation skeleton — pop when top equals current character:
  ```python
  def removeDuplicates(s):
      stack = []
      for ch in s:
          if stack and stack[-1] == ch:
              stack.pop()
          else:
              stack.append(ch)
      return ''.join(stack)
  ```
- **Remove K Digits (LC 402):** keep an increasing stack, pop up to k times when a smaller digit arrives → greedy monotonic stack hybrid

## Sources

- [[dsa/concepts/stack]]
- [[dsa/patterns/monotonic-stack]]
