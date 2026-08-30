---
uid: 12b26a62-1c6e-4134-b331-ebf65a2275d7
---

# Agent Evaluation

**Topic:** [[agents/topics/agent-architectures]]
**Related:** [[agents/concepts/agent-loop]], [[agents/concepts/guardrails-and-safety]], [[ml/topics/model-evaluation]]

---

## What it is

Evaluating an agent is harder than evaluating a single model call, because the thing under test is not one output but a **trajectory** — a sequence of decisions, tool calls, and intermediate states that may reach the right answer by the wrong path, or the wrong answer despite reasonable-looking steps along the way. Standard ML evaluation ([[ml/topics/model-evaluation]]) largely assumes one input → one output; agent evaluation has to assess the whole path.

---

## What to measure

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                TRAJECTORY                               │
└─────────────────────────────────────────────────────────────────────────┘
Goal ──► [Think][Act][Observe] → [Think][Act][Observe] → ... → Final Answer
              │                   │                    │
              ▼                   ▼                    ▼
              Was this the right  Was this tool call   Did the final answer
              tool to reach for?  correctly formed?    satisfy the goal?
```

| Level          | Question                                    | Example metric                                                              |
| -------------- | ------------------------------------------- | --------------------------------------------------------------------------- |
| **Outcome**    | Did the agent achieve the goal?             | Task success rate                                                           |
| **Trajectory** | Did it take a reasonable path to get there? | Tool-call precision/recall against a reference path, step count vs. optimal |
| **Efficiency** | How much did it cost to get there?          | Tokens used, tool calls made, wall-clock time                               |
| **Safety**     | Did it stay within bounds the whole way?    | Guardrail violation rate, unauthorized action attempts                      |

A common trap: measuring only outcome. An agent that gets the right answer by calling a destructive tool it shouldn't have, or by getting lucky on a wrong intermediate step, looks identical to a well-behaved agent on an outcome-only metric.

---

## Evaluation methods

### Task success rate (outcome-based)
Run the agent on a benchmark set of tasks with known correct outcomes; measure the percentage completed correctly. Simple, but says nothing about *how* it got there.

### Trajectory evaluation
Compare the agent's actual sequence of actions against a reference trajectory (or a set of acceptable trajectories) for that task. Useful for catching "right answer, dangerous or inefficient path."

### LLM-as-judge
Use a (typically stronger or differently-prompted) LLM to evaluate the agent's final output or full trajectory against a rubric, when a single correct answer can't be checked programmatically (e.g., "was this customer support response helpful and on-brand?").

```
Agent transcript ──► Judge LLM + rubric ──► score + rationale

Rubric example:
  - Did the agent use tools appropriate to the request? (0-2)
  - Did it avoid unnecessary or redundant tool calls? (0-2)
  - Is the final answer factually grounded in tool observations? (0-2)
  - Did it ask for clarification when the request was ambiguous? (0-2)
```

LLM-as-judge is useful but not free of bias — judges can be fooled by confident-sounding but wrong output, and need their own periodic validation against human judgment.

### Human evaluation
The ground truth for subjective quality (tone, judgment calls, edge-case handling) that automated methods struggle to capture. Expensive, so typically sampled rather than run on every case.

### Regression suites
A fixed, versioned set of representative tasks re-run on every change to the agent's prompt, tools, or underlying model, to catch quality regressions before shipping — directly analogous to a software test suite, but with output that must be scored (pass/fail thresholds or LLM-as-judge) rather than exact-matched.

### Production monitoring / tracing
Live agents need observability: log the full trajectory (thoughts, actions, observations) for every run, not just the final output, so failures can be diagnosed after the fact rather than only caught in offline eval. See [[agents/scenarios/agent-debugging-playbook]] for what to do with these traces when something goes wrong.

---

## Building an eval set

| Source | What it captures |
|---|---|
| Hand-written test cases | Known edge cases, adversarial inputs, safety boundaries |
| Sampled production traffic | Realistic distribution of what users actually ask |
| Synthetic generation (LLM-generated variations) | Scale, coverage of paraphrases and edge cases |
| Failure cases from past incidents | Regression protection — a failure should never recur silently |

**Rule of thumb to teach:** don't ship an agent to production without a regression suite that would have caught the last three failures you found in manual testing. If you can't point to that suite, you don't have an eval process, you have vibes.

---

## Anticipated Questions

1. "Why isn't 'did it get the right answer' enough?" — Because an agent that reaches the right answer via an unsafe, inefficient, or lucky path will keep doing that on inputs where luck runs out, or where the unsafe path has real consequences. Outcome-only evaluation is blind to *how* — see the trajectory row above.
2. "Isn't LLM-as-judge circular — using an LLM to grade an LLM?" — It's a real limitation, not a solved problem. It's most trustworthy for well-specified rubrics with grounded criteria (did it cite a source that exists, did it stay within N tool calls) and least trustworthy for open-ended quality judgments — pair it with periodic human spot-checks to calibrate.
3. "How many eval examples are 'enough'?" — Enough to cover the task's major branches and known edge cases, and enough that a regression shows up as a real signal, not noise — for most agent tasks this starts in the dozens to low hundreds of cases, growing as new failure modes are discovered in production.
4. "How is this different from standard ML model evaluation?" — Standard ML eval ([[ml/topics/model-evaluation]]) scores a fixed input/output mapping. Agent eval scores a variable-length *process* with branching decisions, so it needs trajectory-level and efficiency-level metrics in addition to outcome metrics.

---

## Sources
- [[ml/topics/model-evaluation]]
- [[agents/concepts/agent-loop]]
- [[agents/concepts/guardrails-and-safety]]
- [[agents/scenarios/agent-debugging-playbook]]
