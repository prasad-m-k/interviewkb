# Context Engineering

**Topic:** [[agents/topics/memory-and-context]]
**Related:** [[agents/concepts/memory-architectures]], [[agents/concepts/agent-loop]], [[agents/concepts/multi-agent-orchestration]], [[agents/foundations/nlp-fundamentals]], [[agents/foundations/transformers-and-attention]]

---

## What it is

Context engineering is the discipline of deciding *exactly what goes into the model's context window at each step* — not just prompting, but actively curating, compacting, and structuring the finite context budget as an agent runs. As agent loops get longer (more tool calls, more turns), naively appending everything to the context eventually breaks the agent — not because the model gets "confused," but because of concrete, measurable failure modes.

This has become its own discipline as agents moved from single-shot Q&A to long-horizon, many-step tasks (coding agents working across a large codebase, research agents running dozens of searches).

---

## Why naive context accumulation fails

The context budget is measured in **tokens**, not words or characters — see [[agents/foundations/nlp-fundamentals]] if the unit itself isn't already familiar.

```
Turn 1:  [goal]                                    ~500 tokens
Turn 5:  [goal][a1][o1][a2][o2][a3][o3][a4][o4]    ~4,000 tokens
Turn 20: [goal][a1][o1]...[a19][o19][a20][o20]     ~30,000 tokens
Turn 50: [goal][a1][o1]...[a49][o49][a50][o50]     ~90,000+ tokens
```

| Failure mode | Cause |
|---|---|
| Context window overflow | History exceeds the model's max context length |
| Cost blowup | Every turn re-sends the entire growing history — cost scales worse than linearly across a session |
| Latency growth | Longer input = longer time-to-first-token every turn |
| "Lost in the middle" | Models attend less reliably to information buried in the middle of a very long context, even within the window limit — a direct consequence of how self-attention distributes weight across a long sequence, see [[agents/foundations/transformers-and-attention]] |
| Signal dilution | Verbose tool outputs (raw JSON dumps, full file contents) crowd out the information that actually matters for the next decision |

---

## Core techniques

```
┌──────────────────────────────────────────────────────────────┐
│                   CONTEXT BUDGET (finite)                    │
│                                                              │
│ [ system prompt ]  [ curated history ]   [ retrieved facts ] │
│  [ tool schemas ]    [ current task ]    [ scratch/working ] │
│                                                              │
│  ← fixed, small →  ← actively managed →     ← on-demand →    │
└──────────────────────────────────────────────────────────────┘
```

### Compaction / summarization
Periodically replace older turns with a condensed summary that preserves decisions and key facts, discarding raw intermediate noise.
```
Before: [a1][o1][a2][o2][a3][o3][a4][o4]  (4 full turns)
After:  [Summary: "Searched for user's order history (3 orders found,
         most recent #4521 shipped). Attempted refund on #4521,
         blocked — needs manager approval."]
```

### Selective retrieval instead of full history
Rather than keeping everything in context, store history externally and retrieve only what's relevant to the current step (memory-as-retrieval — see [[agents/concepts/memory-architectures]]).

### Tool-result trimming
Truncate or structure verbose tool output before it enters context: return the 5 relevant rows, not the full 10,000-row query result; return a diff, not the whole file.

### Sub-agent context isolation
Delegate a bounded sub-task to a separate agent instance with its *own* clean context window; only the sub-agent's final result (not its entire working trace) returns to the parent's context. This is the primary tool for keeping the orchestrator's context small in multi-agent systems — see [[agents/concepts/multi-agent-orchestration]].

```
Parent agent context:  [goal] [task delegated to sub-agent] [sub-agent's
                                                            final summary only]

Sub-agent's own context (isolated): [sub-task] [a1][o1][a2][o2]...[an][on]
                                    ← this trace never touches the parent
```

### Structured scratchpad / file-based memory
Let the agent write intermediate notes or state to a file or structured store instead of holding everything in the live prompt; read back only what's needed for the next step.

---

## Context engineering vs. prompt engineering

| | Prompt engineering | Context engineering |
|---|---|---|
| Scope | Wording and structure of a single prompt | What information exists in context *at all*, across an entire multi-step run |
| Time horizon | One call | The full lifetime of an agent session |
| Core question | "How do I phrase this to get the right output?" | "What does the model need to see *right now*, and what should be dropped, summarized, or fetched on demand?" |

Prompt engineering is a subset of context engineering — context engineering also covers memory, retrieval, and multi-agent context isolation.

---

## Anticipated Questions

1. "Why not just use a model with a huge context window and skip all this?" — Larger windows push the failure point further out but don't eliminate cost growth, latency growth, or lost-in-the-middle attention degradation. Context engineering is still needed for long-running or high-volume agents even with large windows — it's a cost/quality lever, not just a workaround for small windows.
2. "Isn't summarizing risky — what if the summary drops something important?" — Yes, this is the central tradeoff. Mitigations: summarize conservatively (keep decisions and open questions verbatim), keep the *raw* history in external storage even after summarizing the in-context version, and let the agent explicitly re-fetch detail on demand.
3. "How do sub-agents help with context if they're still LLM calls that cost tokens?" — They trade *total* tokens spent (which may be higher) for keeping any single context window small and focused, which improves the model's attention quality and keeps the parent orchestrator's context from ballooning as the number of sub-tasks grows.
4. "What's the single highest-leverage context engineering technique to teach first?" — Tool-result trimming. It's the cheapest to implement and the most common real-world cause of context bloat — verbose, unstructured tool outputs accumulating turn after turn.

---

## Sources
- [[agents/concepts/memory-architectures]]
- [[agents/concepts/agent-loop]]
- [[agents/concepts/multi-agent-orchestration]]
- [[agents/foundations/nlp-fundamentals]]
- [[agents/foundations/transformers-and-attention]]
