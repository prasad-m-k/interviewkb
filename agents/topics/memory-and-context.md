# Memory and Context

**Related concepts:** [[agents/concepts/memory-architectures]], [[agents/concepts/context-engineering]], [[agents/concepts/agentic-rag]]

## Overview

LLMs have no persistent state between calls — everything an agent appears to "remember" is information deliberately re-inserted into its context. This topic covers two tightly linked but distinct concerns: **what to remember and where to store it** ([[agents/concepts/memory-architectures]]), and **what to actually put in the context window on any given call** ([[agents/concepts/context-engineering]]).

## The Two Concerns, Side by Side

| | Memory architecture | Context engineering |
|---|---|---|
| Question it answers | What should persist, and where? | What goes into *this* prompt, right now? |
| Time horizon | Across sessions / long-term | Within a single session or loop |
| Failure if done wrong | Agent "forgets" things it should know | Context overflow, cost blowup, lost-in-the-middle |
| Core mechanism | Storage (buffer, KV store, vector DB) + write/read cycle | Compaction, retrieval, trimming, sub-agent isolation |

## Memory Hierarchy

```
LONG-TERM (persists across sessions)
  Semantic (facts) · Episodic (past events) · Procedural (learned strategies)
                             │
                    retrieve relevant slices
                             ▼
SHORT-TERM (this session's live context window)
  conversation history + action/observation trace
```
Full detail: [[agents/concepts/memory-architectures]].

## Managing a Filling Context Window

```
Context budget is finite. As a session grows:
  compact/summarize older turns  →  keep decisions, drop raw noise
  retrieve instead of accumulate →  fetch only what's relevant, on demand
  trim tool outputs               →  return signal, not raw dumps
  isolate sub-agent context       →  parent only sees the sub-agent's summary
```
Full detail: [[agents/concepts/context-engineering]].

## Where This Connects to RAG

Long-term memory retrieval and [[ml/concepts/rag|RAG]] use the same underlying mechanism (embed → vector search → retrieve relevant chunks) applied to different corpora: RAG retrieves from a document corpus, agent memory retrieves from the agent's own accumulated history. [[agents/concepts/agentic-rag]] extends this further by letting the agent decide *when* and *how many times* to retrieve, from either source. For the mechanics behind "embed" and "vector search" themselves, see [[agents/foundations/embeddings-and-similarity]] and [[agents/foundations/vector-databases]].

## Anticipated Questions

- "Why treat memory and context as separate topics if they're this related?" — Because they fail differently and are solved with different tools: memory is a storage/retrieval design problem (what to persist, in what store); context is a per-call budget problem (what fits, what gets cut). A system can have excellent long-term memory and still blow its context budget on a single long tool call, or vice versa.
- "Does a bigger context window make memory architecture unnecessary?" — No — see the "why naive context accumulation fails" table in [[agents/concepts/context-engineering]]: cost, latency, and lost-in-the-middle attention degradation persist even within a large window.
- "What's the single most common mistake students make here?" — Treating "remember" as free. Every piece of retained information has a cost (context tokens if kept live, retrieval latency and potential noise if stored externally) — memory design is a tradeoff, not a checkbox.

## Sources
- [[agents/concepts/memory-architectures]]
- [[agents/concepts/context-engineering]]
- [[ml/concepts/rag]]
- [[agents/foundations/embeddings-and-similarity]]
- [[agents/foundations/vector-databases]]
