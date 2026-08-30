# Multi-Agent Orchestration

**Topic:** [[agents/topics/multi-agent-systems]]
**Related:** [[agents/concepts/context-engineering]], [[agents/patterns/supervisor-worker-pattern]], [[agents/concepts/agent-evaluation]]

---

## What it is

Multi-agent orchestration is the design of how *multiple* agent instances coordinate to accomplish a goal that's too large, too parallel, or too heterogeneous for one agent with one context window to handle well. It's a further design choice layered on top of the single-agent loop ([[agents/concepts/agent-loop]]) — every "agent" in a multi-agent system is still just an instance of that same loop, now composed with others.

---

## Why split into multiple agents at all

| Problem with a single agent | How multi-agent addresses it |
|---|---|
| Context window fills up with unrelated sub-tasks | Each sub-agent gets its own clean, focused context — see [[agents/concepts/context-engineering]] |
| Sequential work that could run in parallel | Independent sub-tasks fan out to concurrent agents |
| One agent needs deep expertise in several different domains | Specialize: a research agent, a coding agent, a review agent, each with tailored tools/prompts |
| A single agent's mistakes go unchecked | Add a reviewer/critic agent that checks another agent's output before it's used |

The cost: more total tokens spent (more LLM calls overall), coordination overhead, and new failure modes (see below). Multi-agent is a scaling and specialization tool, not automatically a quality improvement — don't reach for it before a single well-designed agent has been tried, per the same judgment taught in [[agents/concepts/what-is-an-agent]].

---

## Topologies

```
1. SEQUENTIAL (pipeline)
   Agent A ──► Agent B ──► Agent C ──► result
   (each agent's output is the next agent's input; like a workflow of agents)

2. SUPERVISOR / ORCHESTRATOR-WORKER
                  ┌─────────────┐
                  │ Supervisor  │
                  └──────┬──────┘
            ┌────────────┼────────────┐
            ▼            ▼            ▼
        Worker A     Worker B     Worker C
   (supervisor decides which worker handles what, and synthesizes results)

3. PARALLEL FAN-OUT / FAN-IN
                  ┌─────────────┐
                  │ Coordinator │
                  └──────┬──────┘
            ┌────────────┼────────────┐
            ▼            ▼            ▼
         Agent A      Agent B      Agent C                         (run concurrently,
            │            │            │                             independent sub-tasks)
            └────────────┼────────────┘
                         ▼
                  ┌─────────────┐
                  │ Synthesis   │
                  └─────────────┘

4. PEER-TO-PEER / SWARM
      Agent A ◄──────► Agent B
         ▲                 ▲
         │                 │
         ▼                 ▼
      Agent D ◄──────► Agent C
   (agents communicate directly, no central coordinator — rare in
    production; hardest to keep predictable)

5. EVALUATOR-OPTIMIZER (generator + critic)
      Generator ──► draft ──► Evaluator ──► feedback ──► Generator
                                   │
                          (loops until the evaluator accepts,
                           or a max-round limit is hit)
```

The **supervisor/orchestrator-worker** topology is the most common in production: it keeps coordination centralized and auditable, and maps naturally onto delegation (a supervisor breaks a goal into sub-tasks a specialized worker each handles) — see [[agents/patterns/supervisor-worker-pattern]] for the implementation pattern.

---

## Coordination challenges

| Challenge | Description |
|---|---|
| **Context isolation vs. information loss** | Sub-agents with isolated context avoid clutter, but can also lose context the parent had that would have helped — the summary handed back has to be genuinely sufficient |
| **Error compounding** | If worker A makes a subtle mistake, and worker B builds on A's output without re-verifying it, the error propagates and can be harder to catch than in a single agent's own self-correcting loop |
| **Communication overhead** | Passing state between agents (structured handoffs, shared memory, or message passing) has to be designed deliberately — implicit "the other agent will just know" does not happen. A shared vector store (see [[agents/foundations/vector-databases]]) is one common mechanism: agents write findings to it and others retrieve what's relevant, without every fact flowing through explicit handoffs |
| **Cost multiplication** | N agents each running their own loop can cost meaningfully more in tokens than one agent handling the task serially — parallelism buys latency, not necessarily lower total cost |
| **Debugging difficulty** | A failure could originate in any agent's trajectory; tracing requires visibility into every agent's full loop, not just the final synthesized output |

---

## Anticipated Questions

1. "When is multi-agent actually worth the added cost and complexity?" — When sub-tasks are genuinely independent and benefit from parallel execution (latency win), or when sub-tasks need different tools/expertise/context that would otherwise bloat a single agent's context past the point of good performance (quality win). If neither applies, a single agent is simpler and usually just as good.
2. "How do agents 'talk' to each other?" — Almost always through structured handoffs, not free-form chat: a supervisor passes a specific sub-task and receives back a specific, often schema-constrained result — not the sub-agent's entire raw working trace (see context isolation in [[agents/concepts/context-engineering]]).
3. "What happens if two agents disagree, or one agent's output is wrong?" — This is exactly what the evaluator-optimizer topology and reviewer/critic agents exist to catch — a dedicated verification step rather than assuming downstream agents will self-correct. See also [[agents/patterns/reflection-pattern]].
4. "Is a multi-agent system just several independent workflows glued together?" — Only if the coordinator's routing is fixed by code (a workflow of agents). If the *coordinator itself* is an LLM deciding at runtime which agent handles what and whether to loop back, it's agentic at the orchestration layer too — same workflow-vs-agent test from [[agents/concepts/what-is-an-agent]], just applied one level up.

---

## Sources
- [[agents/concepts/what-is-an-agent]]
- [[agents/concepts/context-engineering]]
- [[agents/patterns/supervisor-worker-pattern]]
- [[agents/patterns/reflection-pattern]]
- [[agents/foundations/vector-databases]]
