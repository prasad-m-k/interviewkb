---
uid: 25f38dd2-2cca-4367-9b02-9a9f292ee289
---

# The Agent Loop

**Topic:** [[agents/topics/agent-fundamentals]], [[agents/topics/agent-architectures]]
**Related:** [[agents/concepts/what-is-an-agent]], [[agents/concepts/tool-calling]], [[agents/concepts/reasoning-and-planning]], [[agents/concepts/agent-evaluation]]

---

## What it is

Every agent, regardless of framework, is built around the same core control loop: the model **observes** its current state, **decides** on an action, **acts** (usually by calling a tool), and **observes** the result — repeating until a goal or stopping condition is reached. This is sometimes called the **think-act-observe loop**, or in classical AI terms, the **perceive-reason-act cycle**.

Everything else in agent design — memory, planning, tool schemas, guardrails — plugs into this loop at a specific point.

---

## How it works

```
       Goal
         │
         ▼
┌────────────────┐
│ 1. THINK       │
│ (reason, plan  │
│ next step)     │
└────────────────┘
         │
         ▼
┌────────────────┐
│ 2. ACT         │
│ (tool call or  │
│ final answer)  │
└────────────────┘───► if final answer: DONE (return the answer)
         │  if tool call:
         ▼
┌────────────────┐
│ 3. EXECUTE     │
│ (orchestrator  │
│ runs the tool) │
└────────────────┘
         │
         ▼
┌────────────────┐
│ 4. OBSERVE     │
│ (result fed    │
│ back in)       │
└────────────────┘
         │
         └──── loops back to Step 1 (THINK) with the new observation
```

The loop terminates when the model emits a final answer, a stopping condition is hit (see below), or an external actor intervenes (human-in-the-loop, timeout).

---

## The four stages in detail

1. **Think.** The model reasons over the goal, the conversation/action history, and the most recent observation. Mechanically, this is a single forward pass through a transformer — see [[agents/foundations/transformers-and-attention]] — producing the next chunk of text; this is where chain-of-thought or explicit "Thought:" scratchpad text happens — see [[agents/concepts/reasoning-and-planning]].
2. **Act.** The model emits a structured action: a tool call with arguments, or a final response. See [[agents/concepts/tool-calling]] for the mechanics.
3. **Execute.** The *orchestrator* (not the model) actually runs the tool — calls the API, executes the code, queries the database — and captures the result or error. This is a critical separation: the model never directly touches the environment; a harness does, on the model's behalf.
4. **Observe.** The tool's result (or error) is serialized and appended to the model's context for the next iteration.

---

## Stopping conditions

An unbounded loop is a production incident waiting to happen. Every real agent harness enforces explicit termination logic:

| Stopping condition | Purpose |
|---|---|
| Model emits a final answer / no more tool calls | Normal, successful termination |
| Max iteration / max turn count | Hard ceiling against infinite loops |
| Max token / cost budget | Prevents runaway spend |
| Wall-clock timeout | Bounds latency for interactive use |
| Repeated identical action (loop detection) | Model stuck retrying the same failing call |
| Human approval required and denied | Guardrail-triggered halt — see [[agents/concepts/guardrails-and-safety]] |

**Training emphasis:** ask students to name at least three of these before they ship any agent. "It'll just stop when it's done" is the single most common rookie mistake — models get stuck retrying a malformed tool call or oscillating between two actions.

---

## Complexity and cost

Each iteration of the loop re-sends the accumulated history to the model (unless context is actively managed — see [[agents/concepts/context-engineering]]). Cost and latency scale roughly linearly with the number of iterations, and context-length costs can scale worse than linearly if history isn't pruned:

```
iteration 1: context = goal
iteration 2: context = goal + action1 + observation1
iteration 3: context = goal + action1 + observation1 + action2 + observation2
...
iteration n: context grows by ~1 action+observation pair every step
```

This is why long-horizon agents need active context management, not just a bigger context window — see [[agents/concepts/context-engineering]].

---

## Anticipated Questions

1. "What's the difference between the agent loop and just prompting the model repeatedly in a chat?" — In a chat, the *human* supplies the next input each turn. In an agent loop, the *tool result* supplies the next input, and the model itself decided what tool to call — no human in the loop per step.
2. "How do you prevent infinite loops?" — Layered defenses: max iteration count, loop/cycle detection (hash recent actions, flag repeats), cost budget, and a fallback "I'm stuck, here's what I tried" response path rather than silent failure.
3. "Does the model 'remember' previous iterations, or is it re-reading everything each time?" — With most current architectures, the *entire* history (or a managed summary of it) is re-sent as input context on every iteration. The model has no persistent state between calls — see [[agents/concepts/memory-architectures]].
4. "Where does planning fit into this loop?" — Planning can happen inline every "Think" step (as in ReAct — [[agents/patterns/react-pattern]]), or as a separate upfront phase before the loop starts (as in plan-and-execute — [[agents/patterns/plan-and-execute]]).

---

## Sources
- [[agents/concepts/what-is-an-agent]]
- [[agents/concepts/tool-calling]]
- [[agents/patterns/react-pattern]]
- [[agents/foundations/transformers-and-attention]]
