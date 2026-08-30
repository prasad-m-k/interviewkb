---
uid: 2e5e014e-dc7f-4b28-917c-2142b91a2e53
---

# Multi-Agent Orchestration

**Topic:** [[solution-arch/topics/agentic-ai-architecture]]
**Related concepts:** [[solution-arch/concepts/model-context-protocol-mcp]], [[solution-arch/concepts/llm-observability-and-evals]]
**Related patterns:** [[solution-arch/patterns/agentic-workflow-patterns]], [[solution-arch/patterns/human-in-the-loop]], [[solution-arch/patterns/microservices-decomposition]]

## What it solves

A single agent's context window and single system-prompt persona eventually become a bottleneck: too many tools degrade selection accuracy, too much conversation history dilutes attention, and a single persona can't specialize (a "do everything" agent is worse at each individual thing than a focused one). Multi-agent orchestration splits work across cooperating agents, each with its own context, tools, and system prompt — the same decomposition instinct as [[solution-arch/patterns/microservices-decomposition]], applied to agents instead of services, with a non-deterministic router instead of a fixed API contract.

## Topology 1: Supervisor (Hierarchical)

```
                    ┌─────────────────┐
                    │   Supervisor     │  Owns overall goal, decides
                    │   Agent          │  which sub-agent acts next,
                    └────────┬────────┘  synthesizes final answer
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   ┌─────────────┐   ┌─────────────┐    ┌─────────────┐
   │ Research     │   │ Drafting     │    │ Review /     │
   │ Agent        │   │ Agent        │    │ Compliance   │
   │ (tools:      │   │ (tools:      │    │ Agent (tools:│
   │  web search) │   │  none)       │    │  none —      │
   │              │   │              │    │  critiques   │
   │              │   │              │    │  only)       │
   └─────────────┘   └─────────────┘    └─────────────┘
```

Most common production topology. The supervisor is itself just an orchestrator-worker pattern (see [[solution-arch/patterns/agentic-workflow-patterns]]) where workers are full agents rather than single calls.

## Topology 2: Peer Handoff (Sequential Specialists)

```
   User ──▶ [Triage Agent] ──hands off──▶ [Billing Agent]
                                              │
                                    (if out of scope) hands off
                                              ▼
                                        [Technical Agent]
```

Each agent decides when to hand off to a different specialist, passing accumulated context forward — common in customer support systems where the right specialist isn't known until the conversation develops.

## Topology 3: Blackboard (Shared Workspace)

```
                     ┌─────────────────┐
                     │  Shared state /   │
              ┌─────▶│  "blackboard"     │◀─────┐
              │      │  (working memory) │       │
              │      └─────────────────┘       │
        ┌─────┴─────┐                    ┌─────┴─────┐
        │ Agent A    │                    │ Agent B    │
        │ (reads/    │                    │ (reads/    │
        │  writes)   │                    │  writes)   │
        └───────────┘                    └───────────┘
```

Agents don't call each other directly; they read and write to shared state, and act opportunistically when they can contribute. Rarely used in production LLM systems today (coordination complexity is high) but shows up in interviews as a contrast case to supervisor topology — know it exists and why it's less common: harder to reason about, harder to evaluate/trace than an explicit supervisor hierarchy.

## Why Multi-Agent, Specifically (Not Just "More Prompting")

```
Reason 1 — Context isolation
  Each sub-agent's context window only holds what IT needs. A
  single mega-agent handling research + drafting + review would
  need all three contexts combined, hitting window limits sooner
  and diluting attention across unrelated concerns.

Reason 2 — Tool/permission separation
  A review agent that only critiques never needs write access to
  anything — enforced structurally, not just by prompt instruction.
  This is a genuine safety win over one agent holding all tools.

Reason 3 — Specialization
  A narrowly-scoped system prompt + tool set performs more reliably
  on its specific task than one broad persona trying to do everything.

Reason 4 — Parallelism
  Independent sub-agents (e.g. researching different sources) can
  run concurrently, cutting wall-clock latency.
```

## Cost of Multi-Agent (Say This Unprompted in an Interview)

```
- Token cost multiplies: each agent has its own system prompt +
  context overhead; a 3-agent system can cost 3-5x a single well-
  designed agent for the same task
- Coordination failures are a NEW failure mode: a supervisor
  misrouting to the wrong specialist, or two agents acting on stale
  shared state, has no analogue in a single-agent system
- Evaluation gets harder: you need trajectory-level evals (see
  [[solution-arch/concepts/llm-observability-and-evals]]) across
  the WHOLE multi-agent trace, not just a final-answer check
- Debugging requires full distributed tracing across agent
  boundaries — same operational investment as debugging
  microservices (see [[solution-arch/topics/microservices]]),
  now with non-deterministic routing on top
```

## When to Use vs Avoid

```
Use multi-agent when:
  ✅ Sub-tasks require genuinely different tools/context that would
     bloat a single agent past useful context budget
  ✅ Safety requires structural separation (a reviewer that
     structurally cannot take write actions)
  ✅ Independent sub-tasks benefit meaningfully from parallel execution

Avoid multi-agent when:
  ❌ A single well-structured prompt with routing (see
     [[solution-arch/patterns/agentic-workflow-patterns]]) handles
     the task's variability already
  ❌ Sub-agents need to share so much state that coordination
     overhead exceeds the specialization benefit
  ❌ You haven't yet established reliable evals for a SINGLE agent —
     adding more agents multiplies an unmeasured failure rate rather
     than fixing it
```

## Sources
- [[solution-arch/sources/building-effective-agents-anthropic]]
- [[solution-arch/patterns/microservices-decomposition]]
