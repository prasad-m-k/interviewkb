# Agentic RAG

**Topic:** [[agents/topics/agent-architectures]], [[agents/topics/tool-use]]
**Related:** [[ml/concepts/rag]], [[ml/patterns/rag-pattern]], [[agents/concepts/tool-calling]], [[agents/patterns/agentic-rag-pattern]], [[agents/foundations/embeddings-and-similarity]], [[agents/foundations/vector-databases]]

---

## What it is

Classic [[ml/concepts/rag|RAG]] is a fixed pipeline: retrieve once, then generate. **Agentic RAG** treats retrieval as just another tool the agent can choose to call, zero or many times, reformulating queries and chaining retrievals based on what it finds — the same shift from workflow to agent described in [[agents/concepts/what-is-an-agent]], applied specifically to retrieval.

```
Classic RAG (fixed pipeline):
  query ──► retrieve (always, exactly once) ──► generate ──► answer

Agentic RAG (model-directed):
  query ──► [Think: do I need to retrieve?]
            │
            ├── no  ──► answer directly
            │
            └── yes ──► retrieve ──► [Think: is this enough?]
                                     │
                                     ├── yes ──► generate ──► answer
                                     │
                                     └── no  ──► reformulate query
                                                 ──► retrieve again ──► (repeat)
```

---

## What agentic RAG adds over classic RAG

| Capability | Classic RAG | Agentic RAG |
|---|---|---|
| Decides *whether* to retrieve | No — always retrieves | Yes — skips retrieval for questions that don't need it |
| Decides *how many times* to retrieve | No — exactly once | Yes — can retrieve iteratively until satisfied |
| Reformulates the query | No | Yes — rewrites the query based on what the first retrieval revealed |
| Multi-hop reasoning | No (single retrieval can't chain facts) | Yes — retrieve fact A, use it to formulate the query for fact B |
| Combines retrieval with other tools | No | Yes — e.g., retrieve a document *and* call a calculator *and* check a database in the same task |
| Self-critiques retrieved context | No | Yes — can judge "this context doesn't actually answer the question" and retry |

---

## Multi-hop retrieval example

Question: *"What is the headquarters city of the company that acquired the startup founded by [Person X]?"*

A single retrieval pass over a document store cannot answer this — it requires chaining facts:

```
Thought: First I need to find what startup Person X founded.
Action: search("startup founded by Person X")
Observation: "Person X founded Startup Y in 2019."

Thought: Now I need to find who acquired Startup Y.
Action: search("Startup Y acquisition")
Observation: "Startup Y was acquired by Company Z in 2023."

Thought: Now I need Company Z's headquarters.
Action: search("Company Z headquarters")
Observation: "Company Z is headquartered in Austin, TX."

Thought: I have the full chain.
Action: Final Answer("Austin, TX")
```

This is exactly the ReAct loop ([[agents/patterns/react-pattern]]) with retrieval as one of the available tools.

---

## Query reformulation

If the first retrieval doesn't surface useful context, an agentic RAG system rewrites the query rather than giving up or hallucinating:

```
Original query: "yield"
Retrieved: (nothing relevant — the corpus uses "rate of return")
Reformulated: "rate of return" ──► retrieve again ──► relevant hits found
```

This addresses one of classic RAG's most common failure modes (see the failure-mode table in [[ml/concepts/rag]]) — a vocabulary mismatch between the query and the corpus — by giving the agent a chance to notice and correct it, instead of the pipeline just returning weak results.

---

## When agentic RAG is worth the added complexity

| Use classic RAG when | Use agentic RAG when |
|---|---|
| Questions are single-fact lookups | Questions require multi-hop reasoning across sources |
| Latency is critical (single retrieval pass is fast) | Some latency is acceptable in exchange for accuracy |
| The corpus vocabulary closely matches user queries | Query reformulation would meaningfully improve recall |
| The task never needs a mix of retrieval + other tools | The agent also needs to call other tools alongside retrieval |

Agentic RAG costs more (multiple LLM calls, multiple retrievals) and is harder to make predictable — apply the same "don't add agency you don't need" judgment from [[agents/concepts/what-is-an-agent]].

---

## Anticipated Questions

1. "Is agentic RAG just RAG with a for-loop?" — Functionally, yes — the loop *is* the point. The key addition is that the *model* decides whether to loop again and how to reformulate, not a fixed retry count in code.
2. "How does an agent know retrieved context is 'good enough'?" — Typically via an explicit self-check step in the reasoning trace ("does this passage answer the question?"), sometimes formalized with the same evaluation metrics used for classic RAG (faithfulness, context relevance — see [[ml/concepts/rag]]) applied at each hop.
3. "Doesn't giving the model control over retrieval risk it retrieving too much or too little?" — Yes, this is a real failure mode — an ungrounded agent can under-retrieve (answer from parametric memory when it shouldn't) or over-retrieve (loop unnecessarily, burning cost). Mitigate with explicit stopping conditions and evaluation (see [[agents/concepts/agent-evaluation]]).
4. "Is Self-RAG the same thing as agentic RAG?" — Self-RAG is one specific, trained implementation of the idea (special tokens like `[Retrieve]`/`[IsRel]`/`[IsSup]` learned during fine-tuning — see [[ml/concepts/rag]]). Agentic RAG is the broader pattern; it can be achieved via prompting a general-purpose agent rather than training a specialized model.

---

## Sources
- [[ml/concepts/rag]]
- [[ml/patterns/rag-pattern]]
- [[agents/patterns/react-pattern]]
- [[agents/patterns/agentic-rag-pattern]]
- [[agents/foundations/embeddings-and-similarity]]
- [[agents/foundations/vector-databases]]
