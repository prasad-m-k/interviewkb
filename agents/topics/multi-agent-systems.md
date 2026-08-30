# Multi-Agent Systems

**Related concepts:** [[agents/concepts/multi-agent-orchestration]], [[agents/concepts/context-engineering]], [[agents/patterns/supervisor-worker-pattern]]

## Overview

A multi-agent system composes several agent instances — each running its own think-act-observe loop ([[agents/concepts/agent-loop]]) — to handle a task that benefits from parallelism, specialization, or independent verification. Full mechanics and topology diagrams: [[agents/concepts/multi-agent-orchestration]].

## Topology Comparison

| Topology | Coordination | Predictability | Best for |
|---|---|---|---|
| Sequential pipeline | Fixed order, code-driven | High | Well-defined multi-stage processes |
| Supervisor / orchestrator-worker | Central LLM router | Medium-high | Heterogeneous sub-tasks needing different specialists |
| Parallel fan-out/fan-in | Central coordinator, concurrent workers | Medium | Independent sub-tasks (research across many sources) |
| Evaluator-optimizer | Generator + critic loop | Medium | Tasks where output quality benefits from review before delivery |
| Peer-to-peer / swarm | No central coordinator | Low | Rare in production; research/exploratory settings |

Detailed pattern: [[agents/patterns/supervisor-worker-pattern]] (the most common production topology).

## Why Not Always Multi-Agent

| Single agent wins when | Multi-agent wins when |
|---|---|
| Task fits comfortably in one context window | Task would force excessive, unrelated context into one agent |
| Steps are inherently sequential and interdependent | Sub-tasks are genuinely independent and parallelizable |
| Cost and latency predictability matter most | Specialization (different tools/prompts per sub-task) improves quality enough to justify overhead |
| Debugging simplicity matters | The task is large enough that debugging one long single-agent trace is *already* hard |

## The Coordination Tax

Every additional agent adds:
- **Token cost** — N agents each running their own loop, generally more total tokens than one agent handling it serially.
- **Handoff design** — what exactly gets passed between agents has to be deliberately structured (see context isolation in [[agents/concepts/context-engineering]]), whether via direct handoffs or a shared vector store ([[agents/foundations/vector-databases]]) other agents can query.
- **New failure surface** — error compounding (agent B trusts agent A's possibly-wrong output), and harder debugging (a failure could originate in any agent's trajectory).

## Anticipated Questions

- "Do multi-agent systems get better results than a single strong agent?" — Not automatically. They win when the task structure genuinely benefits from parallelism or specialization; for tasks a single well-scoped agent can already handle within its context budget, added agents mostly add cost and new failure modes without a quality gain.
- "How do you debug a multi-agent failure?" — Trace every agent's individual trajectory, not just the final synthesized output — the root cause could be in any sub-agent's reasoning, in a bad handoff between agents, or in the coordinator's routing decision. See [[agents/scenarios/agent-debugging-playbook]].
- "What's the most production-proven topology to teach first?" — Supervisor/orchestrator-worker. It keeps a single point of coordination (auditable, easy to reason about) while still getting the benefits of specialization and parallel worker execution.

## Sources
- [[agents/concepts/multi-agent-orchestration]]
- [[agents/patterns/supervisor-worker-pattern]]
- [[agents/foundations/vector-databases]]
