# Plan-and-Execute Pattern

**Topic:** [[agents/topics/agent-architectures]]
**Related concepts:** [[agents/concepts/reasoning-and-planning]], [[agents/concepts/agent-loop]], [[agents/patterns/react-pattern]]

---

## What it solves

ReAct decides one step at a time, which means it has no big-picture view of the whole task — it can wander, take a locally-reasonable-but-globally-wrong step, or lose track of the overall goal across a long trace. Plan-and-execute solves this by producing an explicit multi-step plan *before* execution starts, so the agent (or a cheaper executor) works off a known shape, replanning only when a step actually fails.

---

## Template / Skeleton

```python
def plan_and_execute_agent(goal: str, tools: dict, max_replans: int = 3) -> str:
    plan = planner_llm.generate(
        prompt=f"Goal: {goal}\nProduce a numbered step-by-step plan."
    )
    steps = parse_plan(plan)
    results = []

    replans = 0
    i = 0
    while i < len(steps):
        step = steps[i]
        try:
            # Each step may itself be executed by a simple tool call or
            # by a smaller ReAct-style sub-loop for that one step.
            result = executor_llm.execute_step(step, tools, context=results)
            results.append((step, result))
            i += 1
        except StepFailedError as e:
            if replans >= max_replans:
                return f"Failed after {max_replans} replans: {e}"
            # Re-plan the REMAINING work given what's been learned so far
            remaining_goal = f"Original goal: {goal}\nCompleted so far: {results}\nFailure: {e}"
            new_plan = planner_llm.generate(
                prompt=f"{remaining_goal}\nProduce a revised plan for the remaining work."
            )
            steps = steps[:i] + parse_plan(new_plan)
            replans += 1

    return synthesizer_llm.generate(goal=goal, results=results)
```

---

## Worked Example

```
Goal: "Migrate the users table to the new schema"

Plan (upfront):
  1. Back up the current table
  2. Add new columns with defaults
  3. Backfill new columns from old ones
  4. Validate row counts match
  5. Drop deprecated columns

Execute step 1 → success
Execute step 2 → success
Execute step 3 → FAILS (backfill query times out on large table)

Re-plan (given the failure):
  3a. Backfill in batches of 10,000 rows
  3b. Validate batch-by-batch, not all at once
  4. Validate row counts match
  5. Drop deprecated columns

Execute step 3a → success
Execute step 3b → success
Execute step 4 → success
Execute step 5 → success
```

---

## Signal Phrases

- "Design an agent for a multi-step task with a known overall shape"
- "The agent should show its plan before executing" (auditability requirement)
- "We need to control cost by not re-reasoning from scratch every single step"
- "Break this complex task into sub-tasks"

---

## Plan-and-Execute vs. ReAct

| | ReAct | Plan-and-execute |
|---|---|---|
| When the plan is decided | Implicitly, one step at a time | Explicitly, upfront (with re-planning on failure) |
| Adapts to new info | Immediately, every step | Only when a re-plan is triggered |
| Auditability | Reasoning trace, but no single "plan" artifact | Explicit plan is itself a reviewable artifact — good for human sign-off before execution |
| Cost pattern | Even, spread across steps | Planning cost upfront, then (often cheaper) execution per step |
| Best for | Tasks where the right next step is only knowable after seeing the last result | Tasks with a knowable overall shape, even if individual steps have some uncertainty |

---

## Complexity

| Aspect | Notes |
|---|---|
| Planning cost | One (often larger) call to produce the full plan |
| Execution cost | Can use a cheaper/faster model per step if steps are well-specified |
| Re-planning cost | Only incurred on failure — amortized well if failures are rare |
| Predictability | Higher than ReAct for well-understood task classes; the plan is a checkpoint a human can review before execution |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Plan goes stale | The world changed after planning, before execution finished | Re-plan trigger on step failure or validation mismatch, not just blind continuation |
| Overly rigid plan | Planner assumed conditions that don't hold at execution time | Keep steps coarse enough to allow some execution-time judgment, or make replanning cheap |
| Plan looks right but is subtly wrong | Planner LLM lacks grounding the executor would have discovered | Validate plan steps against real state where possible before committing |

---

## Problems using this pattern
- [[agents/scenarios/coding-agent-design]]
- Multi-step operational tasks (migrations, deployments, data pipelines)

---

## Sources
- [[agents/concepts/reasoning-and-planning]]
- [[agents/patterns/react-pattern]]
