# RAG (Retrieval-Augmented Generation) Pattern

**Topic:** [[ml/topics/nlp]], [[ml/topics/ml-system-design]]
**Related concepts:** [[ml/concepts/rag]], [[ml/concepts/embeddings]], [[ml/concepts/llm-fundamentals]]

---

## What it solves

LLMs trained with a knowledge cutoff cannot answer questions about recent events or proprietary information. Fine-tuning is expensive and bakes knowledge into opaque weights. RAG grounds generation in retrieved documents, making knowledge updateable, citable, and verifiable.

---

## Template / Skeleton

### Indexing Pipeline (Offline)
```python
# 1. Document ingestion
chunks = chunk_documents(docs, max_tokens=512, overlap=50)

# 2. Embedding
embeddings = embedding_model.encode(chunks)  # shape: (N, D)

# 3. Index
vector_db.upsert(ids=chunk_ids, vectors=embeddings, metadata=chunk_texts)
```

### Serving Pipeline (Online)
```python
def rag_query(query: str, k: int = 5) -> str:
    # 1. Embed query
    q_emb = embedding_model.encode(query)

    # 2. Retrieve
    results = vector_db.query(q_emb, top_k=k)
    contexts = [r.text for r in results]

    # 3. Re-rank (optional but recommended)
    ranked = cross_encoder.rank(query, contexts)
    top_contexts = ranked[:3]

    # 4. Generate
    prompt = build_prompt(query, top_contexts)
    return llm.generate(prompt)

def build_prompt(query, contexts):
    ctx = "\n\n".join(f"[{i+1}] {c}" for i, c in enumerate(contexts))
    return f"""Answer the question using only the context below.
Context:
{ctx}

Question: {query}
Answer:"""
```

---

## Signal Phrases

- "Design a document question-answering system"
- "We need to query over internal knowledge base / company documents"
- "LLM is hallucinating facts — how do you ground it?"
- "How do you add new knowledge to an LLM without retraining?"
- "Build a chatbot that answers questions about our product docs"

---

## Full System Design (Google-Scale)

```
                         ┌──────────────────────────────────────────┐
                         │           INDEXING PIPELINE               │
                         │                                           │
  Source Documents ──────►  Chunker  ──► Embedder ──► Vector Store  │
  (GCS / Drive / DB)     │                                           │
                         └──────────────────────────────────────────┘

                         ┌──────────────────────────────────────────┐
                         │            SERVING PIPELINE               │
                         │                                           │
  User Query ────────────►  Query Embed  ──► ANN Search (Matching   │
                         │                    Engine / FAISS)        │
                         │              ──► Cross-Encoder Re-rank    │
                         │              ──► Prompt Assembly          │
                         │              ──► LLM Generation           │
                         │              ──► Faithfulness Check       │
                         └──────────────────────────────────────────┘
```

**Google-specific stack:**
- Vector store: Vertex AI Matching Engine (ScaNN-based ANN, handles billions of vectors)
- Embedding: text-embedding-gecko (Vertex AI)
- LLM: Gemini via Vertex AI
- Orchestration: LangChain, LlamaIndex, or custom Beam pipeline

---

## Complexity

| Component | Latency | Notes |
|---|---|---|
| Query embedding | 5-20 ms | GPU accelerated; cacheable if queries repeat |
| ANN search | 1-10 ms | Scales to billions with HNSW or ScaNN |
| Cross-encoder re-rank | 20-100 ms | Runs on top-50; most expensive step |
| LLM generation | 200ms - 5s | Depends on model size and response length |
| Total P50 | ~500ms | Acceptable for enterprise; tight for consumer |

---

## Problems using this pattern
- [[ml/scenarios/llm-service-design]]
- Enterprise search, internal knowledge management
- Any "chat with your documents" use case

---

## Sources
- [[ml/concepts/rag]]
- [[ml/concepts/embeddings]]
- [[ml/concepts/llm-fundamentals]]
