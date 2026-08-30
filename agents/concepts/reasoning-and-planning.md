# Reasoning and Planning (CoT, ReAct, Tree of Thought)

**Topic:** [[agents/topics/agent-fundamentals]], [[agents/topics/agent-architectures]]
**Related:** [[agents/concepts/agent-loop]], [[agents/patterns/react-pattern]], [[agents/patterns/plan-and-execute]], [[agents/patterns/reflection-pattern]]

---

## What it is

Reasoning and planning techniques change *how* the model uses its own output as intermediate computation before committing to an action. Instead of jumping straight from goal to action, the model is prompted (or trained) to externalize intermediate reasoning steps — which measurably improves accuracy on multi-step tasks, and, for agents specifically, produces the "Thought" that decides the next tool call.

---

## Chain-of-Thought (CoT)

The model is prompted to "think step by step," producing intermediate reasoning tokens before the final answer.

```
Q: If a train travels 60 mph for 2.5 hours, how far does it go?
CoT: Speed × Time = Distance. 60 × 2.5 = 150.
A: 150 miles
```

- Improves performance on arithmetic, logic, and multi-hop reasoning tasks.
- Works because the extra tokens give the model more "compute" (forward passes) to arrive at a correct answer — it can't skip steps.
- **Zero-shot CoT**: just append "Let's think step by step" to the prompt.
- **Few-shot CoT**: show worked examples with reasoning in the prompt.

CoT alone does not touch the environment — it's pure internal reasoning, no tools involved.

---

## ReAct (Reason + Act)

ReAct interleaves reasoning traces with actions: **Thought → Action → Observation**, repeated. This is CoT applied *inside* the agent loop — see [[agents/concepts/agent-loop]] and [[agents/patterns/react-pattern]] for the full pattern.

```
Thought: I need the current weather to answer this.
Action: get_weather(city="Austin")
Observation: 94°F, sunny
Thought: I have what I need.
Action: Final Answer("It's 94°F and sunny in Austin.")
```

Compared to plain CoT, ReAct grounds reasoning in real observations from the environment, which reduces hallucination — the model can't just "imagine" the weather, it has to call the tool and read the result.

---

## Tree of Thought (ToT)

Instead of committing to a single reasoning path, the model explores **multiple branches** of reasoning, evaluates them, and backtracks from dead ends — analogous to search algorithms (BFS/DFS) over a tree of partial solutions.

```
                              [Problem]
              /                   |                   \
         branch1                 branch2                 branch3
      /       \                   |
  eval        eval              eval        ← self-evaluate each partial path
    ✗           ✓                 ✗
(pruned)   (surviving)        (pruned)
                │
               └─ expand further from the surviving path
```

- Useful for problems with a large solution space and no single obvious next step (puzzles, planning, creative generation).
- Much more expensive than CoT or ReAct (many more model calls to explore + evaluate branches).
- Rarely used in production agents due to cost; more common in research and offline planning.

---

## Explicit planning (plan-then-execute)

Rather than reasoning one step at a time, the model produces an upfront **multi-step plan**, which is then executed (by the same or a different model/process), with optional re-planning if a step fails.

```
Goal: "Migrate the users table to the new schema"
Plan:
  1. Back up the current table
  2. Add new columns with defaults
  3. Backfill new columns from old ones
  4. Validate row counts match
  5. Drop deprecated columns
```

See [[agents/patterns/plan-and-execute]] for the full pattern, and [[agents/topics/agent-architectures]] for how this compares to interleaved reasoning (ReAct).

---

## Reasoning vs. Planning — the distinction to teach

| | Reasoning (CoT / ReAct) | Planning (plan-and-execute) |
|---|---|---|
| Granularity | One step at a time, decided just-in-time | Whole sequence decided upfront |
| Adapts to new information | Immediately (interleaved with action) | Only on explicit re-plan |
| Cost pattern | Steady, spread across the loop | Front-loaded planning cost, then cheap execution |
| Best for | Tasks where the right next step depends on what was just discovered | Tasks where the overall shape is knowable in advance |
| Failure mode | Can wander/loop without a big-picture goal check | Plan can go stale if the world changes mid-execution |

In practice, strong agent designs combine both: plan at a coarse level, reason (ReAct-style) within each planned step.

---

## Anticipated Questions

1. "Does CoT actually help, or is it just the model 'talking to itself'?" — It measurably improves accuracy on multi-step tasks because it gives the model additional forward-pass compute to work with before it has to commit to an answer; skipping straight to the answer forces the model to do all reasoning "in its head" in one shot.
2. "Why not always use Tree of Thought if it's more thorough?" — Cost. ToT requires evaluating and often generating many candidate branches, multiplying token usage and latency. Reserve it for problems where a single reasoning path is genuinely likely to be wrong (puzzles, combinatorial search), not routine tasks.
3. "How does the model know when to stop reasoning and act?" — It doesn't inherently; this is controlled by prompting/training conventions (e.g., ReAct's explicit `Action:` tag) and by the harness parsing model output for an action vs. continued reasoning.
4. "What happens if the plan turns out to be wrong halfway through?" — This is the central weakness of pure plan-and-execute: it needs an explicit re-planning trigger (a failed step, a validation check) to revise the plan rather than blindly continuing — see [[agents/patterns/plan-and-execute]] and [[agents/patterns/reflection-pattern]].

---

## Sources
- [[agents/patterns/react-pattern]]
- [[agents/patterns/plan-and-execute]]
- [[agents/concepts/agent-loop]]
