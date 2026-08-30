# Building Effective Agents (Anthropic Engineering)

**Type:** article / engineering blog
**Author:** Anthropic (engineering team)
**Ingested:** 2026-07-27
**Topics covered:** [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/ai-solution-architecture]]

## Summary

A widely-cited engineering post (and the basis of Anthropic's public "building effective agents" cookbook) that draws a precise architectural distinction between **workflows** — systems where LLM calls and tools are orchestrated through predefined, developer-defined code paths — and **agents** — systems where the LLM dynamically directs its own process and tool usage. It catalogs a small set of composable building-block patterns (prompt chaining, routing, parallelization, orchestrator-worker, evaluator-optimizer, and the fully autonomous agent) and argues for finding the simplest solution possible, increasing complexity only when it demonstrably improves outcomes on the specific task.

The core recommendation most relevant to a Solution Architect: start with a single, well-optimized LLM call; add retrieval or few-shot examples before adding orchestration; add a fixed workflow pattern before reaching for a fully autonomous agent; and add multi-agent orchestration only when a single agent's context or tool set genuinely can't cover the task's specialization needs. This directly informs the "complexity spectrum" framing used throughout [[solution-arch/topics/agentic-ai-architecture]] and [[solution-arch/patterns/agentic-workflow-patterns]].

## Key Takeaways

- "Agentic" is a spectrum, not a binary — most production systems are workflows with agentic sub-steps, not fully autonomous agents end to end
- The simplest solution that meets requirements is the right starting point; added orchestration complexity has a real cost (latency, token spend, new failure modes) that must be justified by a measurable outcome improvement
- Six composable patterns cover nearly all production agentic system designs: prompt chaining, routing, parallelization (sectioning/voting), orchestrator-worker, evaluator-optimizer, autonomous agent
- Autonomous agents require explicit stopping conditions (max iterations, budget, checkpoints) — without them, cost and failure risk are unbounded
- Tool design (clear, unambiguous descriptions and schemas) affects agent reliability as much as prompt design does

## What it updated
- Informed: [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/patterns/agentic-workflow-patterns]], [[solution-arch/patterns/multi-agent-orchestration]], [[solution-arch/patterns/human-in-the-loop]], [[solution-arch/concepts/function-calling-and-tool-use]], [[solution-arch/concepts/prompt-engineering-and-context-design]]
