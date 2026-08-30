# Research Agent — System Design

**Difficulty:** Hard
**Topic:** [[agents/topics/multi-agent-systems]]
**Pattern:** [[agents/patterns/supervisor-worker-pattern]], [[agents/patterns/agentic-rag-pattern]]
**Related:** [[agents/concepts/context-engineering]], [[agents/concepts/agent-evaluation]]

---

## Problem

Design an agent (or system of agents) that, given an open-ended research question, searches multiple sources, synthesizes findings, and produces a cited report — the "deep research" class of product.

---

## Clarifying Questions

- Single source type (web search only) or multiple (web, internal docs, databases)?
- Does the output need citations traceable to specific sources?
- Is latency (a few minutes) acceptable in exchange for thoroughness, or does this need to be near-instant?
- How is "research quality" actually judged — comprehensiveness, accuracy, or both?

---

## Requirements

| Type | Requirement |
|---|---|
| Functional | Decompose a broad question, search multiple angles, synthesize with citations |
| Non-functional | Bounded cost/time (unbounded search is a runaway-cost risk), parallelizable for latency |
| Quality | Claims must be traceable to retrieved sources — no unfounded synthesis |

---

## Architecture

```
Question: "How are the top 3 cloud providers pricing GPU inference in 2026?"

                    ┌────────────────────────────────────────────┐
                    │                 SUPERVISOR                 │
                    │ Decomposes into independent sub-questions: │
                    │ 1. AWS GPU inference pricing               │
                    │ 2. GCP GPU inference pricing               │
                    │ 3. Azure GPU inference pricing             │
                    └──────────────────────┬─────────────────────┘
                       ┌───────────────────┼───────────────────┐
                       ▼                   ▼                   ▼
                 ┌───────────┐       ┌───────────┐       ┌───────────┐   ← run concurrently,
                 │ Worker 1  │       │ Worker 2  │       │ Worker 3  │   each an agentic-RAG
                 │ (agentic  │       │ (agentic  │       │ (agentic  │   loop over web search
                 │ RAG loop  │       │ RAG loop  │       │ RAG loop  │   (agentic-rag-pattern)
                 │ over web  │       │ over web  │       │ over web  │
                 │ search)   │       │ search)   │       │ search)   │
                 └───────────┘       └───────────┘       └───────────┘
                       │ cited findings    │ cited findings    │ cited findings
                       └───────────────────┼───────────────────┘
                                           ▼
                                ┌─────────────────────┐
                                │   SYNTHESIS AGENT   │
                                │ merges findings,    │
                                │ resolves conflicts, │
                                │ preserves citations │
                                └─────────────────────┘
                                           │
                                           ▼
                                  Final cited report
```

This is the supervisor-worker topology ([[agents/patterns/supervisor-worker-pattern]]) applied with each worker itself running an agentic RAG loop ([[agents/patterns/agentic-rag-pattern]]) — patterns compose.

---

## Why Multi-Agent Here (and not a single agent)

Per the decision framework in [[agents/topics/multi-agent-systems]]: the three sub-questions are genuinely independent (parallelism win) and each requires enough search-and-read volume that bundling all three into one agent's context would bloat it past useful working size (context win). Both conditions favor multi-agent over single-agent.

---

## Citation and Groundedness

The highest-risk failure mode for a research agent is confident, uncited synthesis — the report reads well but a claim can't be traced back to a retrieved source.

```
Worker output (structured, not free text):
  {
    "claim": "AWS charges $X/hour for H100 inference instances",
    "source_url": "https://...",
    "source_excerpt": "..."
  }
```
Structuring worker output this way (rather than free-form prose) makes it possible for the synthesis agent — and an evaluator — to verify every claim traces to a real, retrieved source. This directly supports the faithfulness/groundedness evaluation discussed in [[ml/concepts/rag]] and [[agents/concepts/agent-evaluation]].

---

## Cost and Latency Controls

| Control | Purpose |
|---|---|
| Max searches per worker | Bounds runaway agentic-RAG loops (see [[agents/concepts/agentic-rag]]) |
| Concurrent worker execution | Total latency ≈ slowest worker, not sum of all workers |
| Synthesis-triggered re-delegation cap | If synthesis finds a gap and re-queries a worker, cap the number of re-delegation rounds |

---

## Evaluation

- **Comprehensiveness:** does the report cover the question's major angles, or miss an obvious one?
- **Groundedness:** does every claim trace to an actual retrieved source (checkable via the structured citation format above)?
- **Conflict handling:** when sources disagree, does the synthesis surface the disagreement rather than silently picking one?
- **Cost/latency:** total searches and wall-clock time per report, tracked against budget.

---

## Key Insight

A research agent's quality is bounded less by the model's writing ability and more by two engineering disciplines: **decomposition** (did the supervisor split the question into genuinely independent, well-scoped sub-questions?) and **groundedness enforcement** (is every claim traceable to a source, structurally, not just prompted for?).

---

## Sources
- [[agents/patterns/supervisor-worker-pattern]]
- [[agents/patterns/agentic-rag-pattern]]
- [[agents/concepts/agentic-rag]]
- [[ml/concepts/rag]]
