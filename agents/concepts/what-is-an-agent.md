# What Is an Agent

**Topic:** [[agents/topics/agent-fundamentals]], [[agents/topics/agent-architectures]]
**Related:** [[agents/concepts/agent-loop]], [[agents/concepts/tool-calling]], [[agents/concepts/reasoning-and-planning]], [[agents/foundations/neural-networks]]

---

## What it is

An **agent** is a system in which an LLM (itself just a neural network trained to predict the next token — see [[agents/foundations/neural-networks]] if that's unfamiliar) dynamically directs its own actions — deciding what steps to take, what tools to call, and when to stop — to accomplish a goal, rather than following a path a human hard-coded in advance.

The load-bearing word is *dynamically*. A single LLM call is not an agent. A fixed sequence of LLM calls wired together in code (a **workflow**) is not an agent either, even if it looks sophisticated. The dividing line is **who decides the control flow**: if a developer's code decides what happens next, it's a workflow; if the model's own output decides what happens next, it's an agent.

> **Anthropic's working definition** (from "Building Effective Agents"): workflows are systems where LLMs and tools are orchestrated through predefined code paths; agents are systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks.

---

## The spectrum: LLM call → workflow → agent

```
Simple                                                 Complex
│                                                            │
▼                                                            ▼
┌───────────┐      ┌───────────────┐      ┌──────────────────┐
│ Single    │      │ Workflow      │      │ Agent            │
│ LLM call  │      │ (fixed DAG of │      │ (LLM decides     │
│           │      │ LLM + tool    │      │ the next step,   │
│ "Classify │   →  │ calls, wired  │   →  │ in a loop, until │
│ this      │      │ by code)      │      │ the goal is met  │
│ ticket"   │      │               │      │ or it gives up)  │
└───────────┘      └───────────────┘      └──────────────────┘
                      predictable,        flexible, open-ended,
                     low variance,          higher variance,
                      easy to test        harder to test/predict
```

Most production systems sit somewhere on this spectrum, and the right point is a design decision, not a status symbol — see [[agents/concepts/agent-loop]] and the "when to use an agent" section below.

---

## Defining properties

An agent (in the modern LLM sense) typically has some combination of:

| Property | Description |
|---|---|
| **Autonomy** | Decides its own next action rather than following a script |
| **Tool use** | Can act on the world (search, code execution, API calls) — see [[agents/concepts/tool-calling]] |
| **Environment feedback loop** | Observes the result of its actions and incorporates it into the next decision — see [[agents/concepts/agent-loop]] |
| **Persistence toward a goal** | Keeps working across multiple steps until a termination condition, not just one turn |
| **(Optional) Memory** | Retains state across steps or sessions — see [[agents/concepts/memory-architectures]] |
| **(Optional) Planning** | Decomposes a goal into sub-steps before or during execution — see [[agents/concepts/reasoning-and-planning]] |

None of these alone makes something an agent. A calculator app "uses a tool." A FAQ chatbot has "memory" of the conversation. What makes a system agentic is the LLM being the thing that decides, step by step, what happens next.

---

## Agent vs. adjacent terms

| Term | What it actually is | Key difference from an agent |
|---|---|---|
| **Chatbot** | Single-turn or multi-turn conversational responder | No tool use or environment action by default; responds, doesn't *do* |
| **Copilot / Assistant** | Suggests actions, human executes or approves each one | Human is in the loop for every action, not just for oversight |
| **Workflow / Pipeline** | Fixed sequence of steps, LLM calls are one stage among many | Control flow is defined by code, not decided by the model at runtime |
| **RPA (Robotic Process Automation)** | Scripted UI/API automation, no reasoning | No model in the loop deciding what to do next; pure replay of rules |
| **Agent** | LLM decides next action, executes it, observes result, repeats | Control flow *emerges* from model output at runtime |

---

## Why this distinction matters (training angle)

Students consistently conflate "uses an LLM + a tool" with "is an agent." The practical test to teach:

**"If I swap out the LLM for a slightly dumber one, does the system still do the right thing?"**
- If yes (because the code enforces the sequence) → it's a workflow.
- If no (because the *model itself* was responsible for choosing the right sequence) → it's an agent, and its quality is bounded by the model's judgment.

This matters operationally: workflows are easier to test, cheaper to run, and more predictable — prefer them when the task is well-defined and doesn't vary much. Reach for an agent when the task is open-ended, the number of steps can't be predicted in advance, or the right tool/step depends on information only discovered mid-task.

---

## When to use an agent (and when not to)

| Use a workflow when | Use an agent when |
|---|---|
| The steps are known in advance | The steps depend on what's discovered along the way |
| Task is high-volume, latency-sensitive | Task is open-ended, exploratory, or long-horizon |
| Errors must be tightly bounded (compliance, finance) | Some trial-and-error is acceptable |
| You need deterministic, auditable behavior | You need flexibility more than predictability |

Anthropic's own guidance: *find the simplest solution possible, and only increase complexity when needed.* Agents cost more (tokens, latency, failure surface) than workflows — don't reach for one by default.

---

## Anticipated Questions

1. "Is a system that calls one tool one time an agent?" — No. That's a single tool-augmented LLM call. Agent-ness requires the *loop*: the model observes the tool result and decides the next action based on it.
2. "Isn't 'agent' just marketing for 'LLM app with a for-loop'?" — Partly fair. The useful technical content is *who controls the branching logic* — code (workflow) or model output (agent) — not the presence of a loop by itself.
3. "Can a system be part workflow, part agent?" — Yes, and most production systems are. A fixed intake pipeline might hand off to an agentic sub-step for the ambiguous part. See [[agents/topics/agent-architectures]].
4. "Does 'agentic' require multiple agents?" — No. A single LLM in a loop with tools is already an agent. Multiple cooperating agents is a *further* design choice — see [[agents/concepts/multi-agent-orchestration]].

---

## Sources
- [[agents/concepts/agent-loop]]
- [[agents/concepts/tool-calling]]
- [[agents/topics/agent-fundamentals]]
- [[agents/foundations/neural-networks]]
