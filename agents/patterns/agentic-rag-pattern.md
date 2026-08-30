# Agentic RAG Pattern

**Topic:** [[agents/topics/agent-architectures]], [[agents/topics/tool-use]]
**Related concepts:** [[agents/concepts/agentic-rag]], [[ml/patterns/rag-pattern]], [[agents/patterns/react-pattern]]

---

## What it solves

Fixed-pipeline RAG (retrieve once, then generate — see [[ml/patterns/rag-pattern]]) fails on questions needing multiple hops of reasoning, and can't recover when the first retrieval misses due to a vocabulary mismatch. This pattern gives the agent control over retrieval itself: it decides whether to retrieve, reformulates the query if results are weak, and chains retrievals for multi-hop questions — retrieval becomes just another tool inside a ReAct-style loop ([[agents/patterns/react-pattern]]).

---

## Template / Skeleton

```python
def agentic_rag(query: str, retriever, llm, max_hops: int = 4) -> str:
    history = [f"Question: {query}"]

    for hop in range(max_hops):
        thought = llm.generate(prompt=build_prompt(history, tools=["search", "answer"]))
        history.append(f"Thought: {thought}")

        action = parse_action(thought)

        if action.type == "answer":
            return action.content

        # action.type == "search"
        results = retriever.query(action.search_query, top_k=5)

        # Agent self-checks whether retrieval was useful before committing to it
        relevance_check = llm.generate(
            prompt=f"Question: {query}\nRetrieved: {results}\n"
                   f"Does this actually help answer the question? yes/no + why."
        )
        history.append(f"Action: search(\"{action.search_query}\")")
        history.append(f"Observation: {results}")
        history.append(f"Relevance check: {relevance_check}")

    return llm.generate(prompt=build_prompt(history + ["Answer with best available information."]))
```

---

## Multi-Hop Worked Example

```
Question: "What is the headquarters city of the company that acquired
the startup founded by Person X?"

Thought: I need to find what startup Person X founded.
Action: search("startup founded by Person X")
Observation: "Person X founded Startup Y in 2019."
Relevance check: yes — identifies the startup

Thought: Now I need who acquired Startup Y.
Action: search("Startup Y acquisition")
Observation: "Startup Y was acquired by Company Z in 2023."
Relevance check: yes — identifies the acquirer

Thought: Now I need Company Z's headquarters.
Action: search("Company Z headquarters")
Observation: "Company Z is headquartered in Austin, TX."
Relevance check: yes — directly answers the remaining question

Thought: I have the full chain of facts needed.
Action: Final Answer("Austin, TX")
```

---

## Query Reformulation Sub-Routine

```python
def search_with_reformulation(query: str, retriever, llm, max_attempts: int = 2):
    for attempt in range(max_attempts):
        results = retriever.query(query, top_k=5)
        if is_relevant(results, query, llm):
            return results
        query = llm.generate(
            prompt=f"This search query returned weak/irrelevant results: \"{query}\"\n"
                   f"Rewrite it to better match likely document phrasing."
        )
    return results  # return best-effort after max_attempts
```

---

## Signal Phrases

- "Build a Q&A system over documents that requires multi-step reasoning"
- "The retrieval isn't finding the right documents — how do we make it adaptive?"
- "Design a research assistant that chains facts across multiple sources"
- "Combine retrieval with other tools (calculator, database) in one agent"

---

## Complexity

| Aspect | Notes |
|---|---|
| Latency | Multiple retrieval + LLM round trips vs. classic RAG's single pass — noticeably slower |
| Cost | Scales with number of hops; bound with `max_hops` |
| Accuracy | Higher on multi-hop and vocabulary-mismatch cases; roughly equivalent to classic RAG on single-fact lookups (with unnecessary overhead) |
| When it's overkill | Simple factoid questions where a single retrieval pass reliably succeeds — see the comparison table in [[agents/concepts/agentic-rag]] |

---

## Problems using this pattern
- [[agents/scenarios/research-agent-design]]
- Enterprise Q&A over documents requiring cross-referencing
- Multi-hop factual question answering

---

## Sources
- [[agents/concepts/agentic-rag]]
- [[ml/patterns/rag-pattern]]
- [[agents/patterns/react-pattern]]
