---
uid: cf85fb2c-0c3b-4f25-a177-ef4aad49816f
---

# Agent Architectures

**Related concepts:** [[agents/concepts/reasoning-and-planning]], [[agents/concepts/multi-agent-orchestration]], [[agents/concepts/guardrails-and-safety]], [[agents/concepts/agent-evaluation]]

## Overview

"Agent architecture" covers the design choices that sit on top of the base agent loop ([[agents/concepts/agent-loop]]): how reasoning and action are interleaved, whether one agent or several handle the task, and how safety and quality are enforced. This page is the map; each pattern has its own deep-dive page.

## Single-Agent Patterns

| Pattern | Core idea | Page |
|---|---|---|
| **ReAct** | Interleave Thought → Action → Observation, one step at a time | [[agents/patterns/react-pattern]] |
| **Plan-and-execute** | Produce a full plan upfront, then execute it (with re-planning on failure) | [[agents/patterns/plan-and-execute]] |
| **Reflection** | Generate → self-critique → revise, before or after acting | [[agents/patterns/reflection-pattern]] |
| **Agentic RAG** | Treat retrieval as a tool the agent chooses to call, iteratively | [[agents/patterns/agentic-rag-pattern]], [[agents/concepts/agentic-rag]] |

```
ReAct (interleaved):                Plan-and-execute (upfront):
  Think → Act → Observe             Plan (all steps) → Execute step 1
  Think → Act → Observe             → Execute step 2
  Think → Act → Observe             → Execute step 3
  ...                               → (re-plan if a
                                       step fails)
```

## Multi-Agent Patterns

| Topology | Core idea | Page |
|---|---|---|
| Sequential pipeline | Fixed chain of agents, each stage's output feeds the next | [[agents/concepts/multi-agent-orchestration]] |
| Supervisor / orchestrator-worker | A supervisor routes sub-tasks to specialized workers | [[agents/patterns/supervisor-worker-pattern]] |
| Parallel fan-out/fan-in | Independent sub-tasks run concurrently, results synthesized | [[agents/concepts/multi-agent-orchestration]] |
| Evaluator-optimizer | Generator + critic loop until output passes review | [[agents/patterns/reflection-pattern]] |

Full comparison and coordination challenges: [[agents/topics/multi-agent-systems]].

## Choosing an Architecture

```
                       Is the task well-defined,
                       repeatable, low-variance?
                                   │
                  ┌────────────────┴────────────────┐
                 yes                               no
                  │                                 │
                  ▼                                 ▼
              Workflow                    Does it need multiple
          (see "What Is an               distinct capabilities /
            Agent" above)                  can it parallelize?
                                                    │
                                    ┌───────────────┴───────────────┐
                                   no                              yes
                                    │                               │
                                    ▼                               ▼
                              Single agent                     Multi-agent
                         (ReAct / plan-execute)            (supervisor-worker,
                                                            parallel fan-out)
```

Every architectural upgrade — from workflow to single agent, from single agent to multi-agent — adds cost, latency, and failure surface. Justify each step up this ladder against the task's actual requirements; don't default to the most sophisticated option.

## Cross-Cutting Concerns (apply to every architecture)

- **Safety:** [[agents/concepts/guardrails-and-safety]] — every architecture needs an execution-boundary enforcement layer, regardless of how sophisticated the reasoning is.
- **Evaluation:** [[agents/concepts/agent-evaluation]] — trajectory quality, not just outcome, needs measurement in every architecture.
- **Context management:** [[agents/concepts/context-engineering]] — becomes more, not less, important as architectures get more sophisticated (longer loops, more agents).
- **Underlying mechanics:** every architecture on this page is still, underneath, a neural network doing next-token prediction — see [[agents/foundations/neural-networks]] and [[agents/foundations/transformers-and-attention]] if that layer isn't already solid.

## Anticipated Questions

- "Which architecture should I default to when teaching this?" — ReAct for single-agent tasks (it's the most general-purpose and the easiest to reason about), supervisor-worker for multi-agent tasks (most predictable and auditable of the multi-agent topologies).
- "Can patterns be combined?" — Routinely. A supervisor-worker system's individual workers are often themselves ReAct agents; a plan-and-execute agent's execution step for a given plan item might invoke reflection before moving on.
- "Is more architectural sophistication always better?" — No — see the decision ladder above. Sophistication should track task requirements, not be a default.

## Sources
- [[agents/concepts/agent-loop]]
- [[agents/concepts/multi-agent-orchestration]]
