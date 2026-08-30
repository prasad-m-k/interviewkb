# Supervisor-Worker Pattern (Orchestrator-Worker)

**Topic:** [[agents/topics/multi-agent-systems]]
**Related concepts:** [[agents/concepts/multi-agent-orchestration]], [[agents/concepts/context-engineering]], [[agents/concepts/agent-evaluation]]

---

## What it solves

A single agent handling a large, heterogeneous task accumulates a bloated, unfocused context (see [[agents/concepts/context-engineering]]) and can't easily parallelize independent sub-work. The supervisor-worker pattern splits the task: a **supervisor** agent decomposes the goal and routes sub-tasks to specialized **worker** agents, each with its own clean context, then synthesizes their results — the most common, most auditable multi-agent topology in production.

---

## Template / Skeleton

```python
def supervisor_agent(goal: str, workers: dict, max_rounds: int = 5) -> str:
    plan = supervisor_llm.generate(
        prompt=f"""Goal: {goal}
Available workers: {list(workers.keys())}
Decompose this goal into sub-tasks and assign each to the best-fit worker."""
    )
    assignments = parse_assignments(plan)  # [(worker_name, sub_task), ...]

    worker_results = {}
    for worker_name, sub_task in assignments:
        # Each worker runs its OWN isolated agent loop with a clean context —
        # only the final result crosses back to the supervisor.
        worker_results[worker_name] = workers[worker_name].run(sub_task)

    synthesis = supervisor_llm.generate(
        prompt=f"""Goal: {goal}
Worker results: {worker_results}
Synthesize a final answer. If results conflict or are insufficient,
identify what additional work is needed."""
    )

    if needs_more_work(synthesis):
        # Supervisor can loop: assign follow-up sub-tasks based on gaps found
        return supervisor_agent(goal, workers, max_rounds - 1)

    return synthesis
```

---

## Architecture Diagram

```
                           ┌─────────────────┐
                    Goal ─►│ SUPERVISOR      │
                           │ (decomposes,    │
                           │ routes, checks, │
                           │ synthesizes)    │
                           └────────┬────────┘
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
             ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
             │ Worker A    │  │ Worker B    │  │ Worker C    │
             │ (own tools, │  │ (own tools, │  │ (own tools, │
             │ own clean   │  │ own clean   │  │ own clean   │
             │ context)    │  │ context)    │  │ context)    │
             └─────────────┘  └─────────────┘  └─────────────┘
                    │                │                │
                    └────────────────┼────────────────┘
                                     ▼
                            only FINAL RESULTS
                           cross back — not each
                            worker's full trace
                                     │
                                     ▼
                           ┌─────────────────┐
                           │ SYNTHESIS /     │
                           │ FINAL ANSWER    │
                           └─────────────────┘
```

The key context-engineering discipline here: each worker's *entire* action/observation trace stays inside the worker — only the distilled result returns to the supervisor's context. This is what keeps the supervisor's context small regardless of how much work any individual worker did.

---

## Worked Example (research task)

```
Goal: "Compare our top 3 competitors' pricing models"

Supervisor decomposes:
  → Worker A: research Competitor 1's pricing
  → Worker B: research Competitor 2's pricing
  → Worker C: research Competitor 3's pricing
  (all three run concurrently, each with its own search tool + context)

Worker A returns: "Competitor 1: tiered SaaS, $29/$99/$299 per month"
Worker B returns: "Competitor 2: usage-based, $0.002/API call"
Worker C returns: "Competitor 3: enterprise-only, custom quotes"

Supervisor synthesizes: "The three competitors use fundamentally different
pricing models — tiered SaaS, usage-based, and enterprise custom-quote —
suggesting [analysis]..."
```

---

## Signal Phrases

- "Design a system that breaks a large task into specialized sub-tasks"
- "We need different tools/expertise for different parts of this problem"
- "Parallelize independent research/work across multiple agents"
- "Build an orchestrator that delegates to specialists"

---

## Complexity

| Aspect | Notes |
|---|---|
| Latency | Can be much lower than sequential single-agent execution if workers run concurrently |
| Cost | Higher total tokens than a single agent (N loops instead of one), traded for latency and/or quality |
| Predictability | Medium — supervisor's routing decisions vary by input, but the topology itself is fixed and auditable |
| Debuggability | Requires tracing supervisor routing + every worker's individual trajectory — see [[agents/scenarios/agent-debugging-playbook]] |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Supervisor misroutes a sub-task | Ambiguous worker specialization boundaries | Sharpen worker descriptions/scope, same discipline as tool design in [[agents/topics/tool-use]] |
| Worker error silently accepted by synthesis | Supervisor doesn't validate worker output before synthesizing | Add explicit validation/critique step before synthesis — see [[agents/patterns/reflection-pattern]] |
| Redundant work across workers | Poor task decomposition, overlapping scopes | Supervisor prompt should explicitly require non-overlapping sub-tasks |
| Synthesis loses important nuance from a worker | Worker results summarized too aggressively before reaching supervisor | Keep worker results structured (not free text) where the task allows it |

---

## Problems using this pattern
- [[agents/scenarios/research-agent-design]]
- Large codebase analysis (route different modules to different sub-agents)
- Multi-domain customer support triage

---

## Sources
- [[agents/concepts/multi-agent-orchestration]]
- [[agents/concepts/context-engineering]]
- [[agents/topics/multi-agent-systems]]
