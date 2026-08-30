# Agentic Workflow Patterns

**Topic:** [[solution-arch/topics/agentic-ai-architecture]]
**Related concepts:** [[solution-arch/concepts/function-calling-and-tool-use]], [[solution-arch/concepts/prompt-engineering-and-context-design]]
**Related patterns:** [[solution-arch/patterns/multi-agent-orchestration]], [[solution-arch/patterns/human-in-the-loop]]

## What it solves

A single LLM call handles simple tasks well but breaks down on multi-step, variable, or high-stakes work. These patterns are the composable building blocks — from fully deterministic to fully autonomous — for structuring LLM calls into a system. The architectural decision is **which pattern matches the task's variability and risk**, not "how do I build the most agentic thing possible." Anthropic's guidance (widely referenced in interviews): find the simplest solution possible, and only increase complexity when it demonstrably improves outcomes.

## The Complexity Spectrum

```
Simplest ─────────────────────────────────────────────▶ Most complex

Single call → Prompt chaining → Routing → Parallelization →
Orchestrator-worker → Evaluator-optimizer → Autonomous agent

Cost, latency, and FAILURE-MODE COMPLEXITY increase left to right.
Start left. Move right only when task variability or scale demands it.
```

## Pattern 1: Prompt Chaining (deterministic workflow)

```
Task decomposed into a FIXED sequence of LLM calls, each step's
output feeding the next, with optional code-based gates between
steps to fail fast before wasting downstream calls.

  in ─▶ [LLM: draft] ─▶ (gate: length check) ─▶ [LLM: translate]
        ─▶ (gate: format check) ─▶ [LLM: polish] ─▶ out
```

**Use when:** the task decomposes cleanly into fixed subtasks known in advance, and each step is simpler/more reliable in isolation than one large prompt trying to do everything at once.

## Pattern 2: Routing

```
                    ┌──▶ [Refunds specialist prompt]
   in ─▶ [Classifier]──▶ [Technical support specialist prompt]
                    └──▶ [General Q&A specialist prompt]
```

**Use when:** inputs fall into distinct categories that are better handled by separate, specialized prompts/models than one prompt trying to cover every case — classic example: routing easy queries to a cheap/fast model and complex ones to a capable/expensive model (see [[solution-arch/topics/cost-architecture-finops]]).

## Pattern 3: Parallelization

```
Sectioning (split independent subtasks, run concurrently):
  in ─▶ ┌─▶ [Check policy compliance] ─┐
        ├─▶ [Check factual accuracy]  ─┼─▶ [Aggregate] ─▶ out
        └─▶ [Check tone]               ─┘

Voting (run the same task N times, take majority/consensus):
  in ─▶ [LLM call ×N, same prompt] ─▶ [Majority vote] ─▶ out
```

**Use when:** subtasks are genuinely independent (sectioning — cuts latency via concurrency) or when a single call's reliability on a high-stakes judgment isn't good enough alone (voting — trades cost for reliability on classification-style decisions).

## Pattern 4: Orchestrator-Worker

```
   in ─▶ [Orchestrator LLM] ──dynamically determines subtasks──▶
                              [Worker 1] [Worker 2] [Worker N]
                              (subtasks NOT known in advance —
                               this is what distinguishes it from
                               routing/parallelization)
                              │
                              ▼
                        [Synthesizer] ─▶ out
```

**Use when:** the number and nature of subtasks can't be predicted ahead of time and must be determined based on the specific input (e.g. "research this topic" — the orchestrator decides how many sub-questions to spin off based on what it finds).

## Pattern 5: Evaluator-Optimizer

```
   in ─▶ [Generator] ─▶ [Evaluator against explicit criteria]
                              │
                    ┌─────────┴─────────┐
                   FAIL                PASS
                    │                    │
          feedback to Generator          ▼
          (bounded retry loop)          out
```

**Use when:** there are clear, articulable evaluation criteria, and iterative refinement measurably improves output quality (code correctness, translation quality) — not useful when there's no reliable way to judge "better."

## Pattern 6: Autonomous Agent (full agentic loop)

The general ReAct-style loop described in [[solution-arch/topics/agentic-ai-architecture]] — used only when the task genuinely requires open-ended, unpredictable steps. Requires the most guardrails (bounded iterations, cost budget, human-in-the-loop checkpoints) because control flow is fully delegated to the model.

## Decision Checklist

```
1. Can this be a single call with good context? → Use that. Stop.
2. Is the task a fixed sequence of known steps? → Prompt chaining.
3. Do inputs fall into distinct known categories? → Routing.
4. Are subtasks independent and parallelizable? → Parallelization.
5. Are subtasks unknown until the input is examined? → Orchestrator-worker.
6. Is there a clear quality bar worth iterating against? → Evaluator-optimizer.
7. Does the task require open-ended, unpredictable multi-step
   decisions with tool use? → Autonomous agent (with full
   guardrail stack — see [[solution-arch/concepts/ai-guardrails-and-safety]]).
```

## Sources
- [[solution-arch/sources/building-effective-agents-anthropic]]
