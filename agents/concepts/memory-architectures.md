# Memory Architectures

**Topic:** [[agents/topics/memory-and-context]]
**Related:** [[agents/concepts/context-engineering]], [[agents/concepts/agentic-rag]], [[agents/concepts/agent-loop]], [[agents/foundations/vector-databases]], [[agents/foundations/embeddings-and-similarity]]

---

## What it is

LLMs are stateless between calls — every "memory" an agent appears to have is actually information explicitly re-inserted into the prompt on the next call. Memory architecture is the design of *what gets stored, where, and how it's retrieved and re-inserted* so the agent behaves as if it remembers.

---

## The memory hierarchy

```
┌──────────────────────────────────────────────────────────────────┐
│                         LONG-TERM MEMORY                         │
│     persists across sessions — database, vector store, files     │
│                                                                  │
│    ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐    │
│    │ Semantic      │  │ Episodic      │  │ Procedural       │    │
│    │ (facts,       │  │ (past events, │  │ (learned skills, │    │
│    │ knowledge)    │  │ sessions)     │  │ strategies)      │    │
│    └───────────────┘  └───────────────┘  └──────────────────┘    │
└─────────────────────────────────┬────────────────────────────────┘
                                  │ retrieve relevant slices
                                  ▼
┌──────────────────────────────────────────────────────────────────┐
│                        SHORT-TERM MEMORY                         │
│       the live context window for this session / this loop       │
│        conversation history + tool call/observation trace        │
└──────────────────────────────────────────────────────────────────┘
```

### Short-term (working) memory
The active context window: the current conversation, the running trace of actions and observations within one agent loop invocation (see [[agents/concepts/agent-loop]]). It vanishes when the session ends unless explicitly persisted. Managing what stays in it as it fills up is the subject of [[agents/concepts/context-engineering]].

### Long-term memory
Persisted outside the context window, retrieved on demand:
- **Semantic memory** — facts and general knowledge the agent has learned or been given (user preferences, domain facts, company policies).
- **Episodic memory** — records of specific past interactions or sessions ("last time this user asked about X, the resolution was Y").
- **Procedural memory** — learned strategies or successful action sequences ("this class of bug is usually fixed by checking the migration first") — the least common to implement explicitly, but powerful for agents that improve over repeated tasks.

---

## Storage and retrieval mechanisms

| Mechanism | What it stores | Retrieval method |
|---|---|---|
| Plain conversation buffer | Raw message history | Just re-send it (works until context limit) |
| Summarized buffer | Compressed history | Re-send the summary; regenerate periodically |
| Key-value / structured store | Explicit facts (`user.timezone = "CST"`) | Direct lookup by key |
| Vector store | Embedded chunks of past interactions or documents | Semantic similarity search (nearest neighbor) — see [[agents/foundations/embeddings-and-similarity]] and [[agents/foundations/vector-databases]] for how this actually works under the hood |
| File / scratchpad | Notes the agent writes for itself mid-task | Read back by the agent or a sub-agent |

Vector-store-backed long-term memory is essentially [[ml/concepts/rag]] applied to an agent's own history rather than to a document corpus — retrieve the most relevant past episodes instead of the most relevant documents.

---

## The write/read cycle

```
DURING the task:                AFTER the task:
agent acts → result             agent (or a summarizer)
  ▼                             condenses the session into
relevant facts flagged for      a memory record
persistence                       ▼
  ▼                             write to long-term store
written to scratch /            (with metadata: timestamp,
working memory                  user, outcome, tags)
```

```
NEXT session / NEXT step:
  new query arrives
       │
       ▼
  embed query → search long-term store → retrieve top-K relevant memories
       │
       ▼
  inject into short-term context for this turn
```

---

## Design tradeoffs

| Choice | Pro | Con |
|---|---|---|
| Keep full raw history in context | Nothing lost | Expensive, hits context limits fast |
| Summarize periodically | Scales to long sessions | Risk of losing important detail in compression |
| Vector-retrieve relevant memories only | Scales indefinitely, only relevant info surfaced | Retrieval can miss things that don't embed well (numbers, exact names) |
| No long-term memory at all | Simplest, no state to manage | Agent "forgets" the user every session — poor UX for repeat interactions |

---

## Anticipated Questions

1. "If the model is stateless, how does ChatGPT 'remember' things about me across sessions?" — It doesn't, in the model itself. A memory feature retrieves stored facts about you and silently re-injects them into the context at the start of the conversation — the model just sees them as part of the prompt.
2. "Isn't long-term memory just RAG?" — Mechanically, yes — usually a vector store + similarity search. The distinction is conceptual: RAG typically retrieves from a static document corpus; agent memory retrieves from the agent's own accumulated experience, and often gets written to as well as read from.
3. "What happens when the context window fills up mid-task?" — Something has to give: truncate oldest history, summarize and replace it, or offload older turns to long-term storage and retrieve only what's relevant. See [[agents/concepts/context-engineering]] for the strategies.
4. "How do you know what's *worth* writing to long-term memory?" — This is an open design problem. Common heuristics: explicit user statements ("remember that I prefer metric units"), task outcomes (success/failure + why), and anything that would change future behavior if known in advance. Storing everything indiscriminately just recreates the context-overflow problem one layer down.

---

## Sources
- [[agents/concepts/context-engineering]]
- [[agents/concepts/agentic-rag]]
- [[agents/foundations/embeddings-and-similarity]]
- [[agents/foundations/vector-databases]]
- [[ml/concepts/embeddings]]
- [[ml/concepts/rag]]
