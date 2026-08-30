# Retrieval-Augmented Generation (RAG)

**Topic:** [[ml/topics/nlp]], [[ml/topics/ml-system-design]]
**Related:** [[ml/concepts/llm-fundamentals]], [[ml/concepts/embeddings]], [[ml/patterns/rag-pattern]]

---

## What it is

RAG combines a retrieval system with a generative model. Instead of relying on knowledge baked into model weights during pretraining, the model retrieves relevant documents at inference time and conditions its generation on them.

This solves two core LLM limitations: **knowledge staleness** (pretraining cutoff) and **hallucination** (generating plausible-sounding false statements).

---

## How it works

```
Query → Retriever → Top-K Documents → LLM → Grounded Response
         (dense vector search or BM25)
```

### Step 1: Indexing (offline)
1. Split documents into chunks (typically 256–512 tokens)
2. Embed each chunk using an embedding model (e.g., text-embedding-gecko, BERT, E5)
3. Store vectors in a vector database (Pinecone, Weaviate, pgvector, Vertex AI Matching Engine)

### Step 2: Retrieval (online)
1. Embed the query with the same embedding model
2. Run approximate nearest neighbor (ANN) search to retrieve top-K chunks
3. Optionally re-rank with a cross-encoder for higher precision

### Step 3: Generation
1. Concatenate retrieved chunks + query into a prompt
2. LLM generates a response grounded in the retrieved context

---

## Retrieval Methods

### Sparse Retrieval (BM25 / TF-IDF)
- Keyword-based; does not require embedding
- Fast; no GPU needed
- Fails for semantic queries ("What is the rate of return?" ≠ "yield")
- Still strong for exact keyword matches (product IDs, names)

### Dense Retrieval (Bi-Encoder)
- Query and document encoded independently into dense vectors
- Similarity = dot product or cosine distance
- ANN index (FAISS, ScaNN, HNSW) for sub-millisecond retrieval at scale
- Requires training on (query, positive_doc, negative_doc) triplets

### Cross-Encoder Re-Ranking
- Takes (query, document) pair as input; outputs a relevance score
- More accurate than bi-encoder but cannot be pre-indexed (must run at query time)
- Too slow to run on millions of documents → run only on top-50 from bi-encoder
- Implements the two-stage retrieval pattern (candidate generation → re-ranking)

### Hybrid Retrieval
Combine BM25 + dense retrieval via reciprocal rank fusion (RRF):
```
RRF_score(d) = sum(1 / (k + rank_i(d)))  for each retriever i
```
Often outperforms either alone.

---

## Chunking Strategies

The granularity of chunks affects recall and context quality:

| Strategy | Description | Trade-off |
|---|---|---|
| Fixed-size | Split every N tokens | Simple; may cut mid-sentence |
| Sentence | Split at sentence boundaries | Better coherence; variable size |
| Paragraph | Natural document structure | Good for structured docs |
| Semantic | Cluster sentences by embedding similarity | Best coherence; expensive |
| Parent-child | Index child chunks; retrieve parent for context | Best of both worlds; complex |

**Rule of thumb:** chunk size should be ≥ the context needed for the answer but ≤ what fits in the LLM prompt alongside multiple retrieved chunks.

---

## Evaluation

### Retrieval Quality
- **Recall@K:** fraction of relevant docs in top-K
- **MRR (Mean Reciprocal Rank):** rank of the first relevant document
- **NDCG@K:** position-discounted relevance

### Generation Quality (with Retrieval)
- **Faithfulness:** does the answer contradict the retrieved context? (RAGAS, FaithScore)
- **Answer Relevance:** does the answer address the question?
- **Context Precision:** fraction of retrieved chunks that are relevant
- **Context Recall:** fraction of relevant information that was retrieved

**RAGAS** is the standard open-source framework for RAG evaluation.

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Retrieved wrong documents | Embedding mismatch, poor chunking | Improve embedding model, add BM25 hybrid |
| Too much noise in context | K too large; irrelevant chunks fill the prompt | Re-ranking, reduce K, increase chunk quality |
| LLM ignores retrieved context | Sycophancy, prompt formatting | Stronger system prompt, citation training |
| Hallucination despite retrieval | LLM extrapolates beyond what is stated | Constrained decoding, faithfulness checks |
| Stale index | Documents updated but index not refreshed | Incremental indexing pipeline, change detection |
| Context window overflow | Many large chunks exceed token limit | Smaller chunks, summarize before passing |

---

## RAG vs. Fine-Tuning

| Scenario | RAG | Fine-Tuning |
|---|---|---|
| Knowledge is dynamic (changes weekly/daily) | RAG (easy to update index) | Fine-tuning (stale until next run) |
| Private proprietary knowledge | RAG (never enters weights) | Fine-tuning (weights contain the data) |
| Behavioral style / tone change | Fine-tuning (or RLHF) | RAG cannot change model behavior |
| Small labeled dataset (hundreds) | Fine-tuning with LoRA | RAG (no training needed) |
| Citation / attribution required | RAG (sources are explicit) | Fine-tuning (opaque) |
| Latency sensitive (<50ms) | Fine-tuning (no retrieval) | RAG adds retrieval latency |

In practice, both are combined: fine-tune for behavior; RAG for knowledge.

---

## Advanced Patterns

### Self-RAG
The model learns to decide *when* to retrieve, *what* to retrieve, and *how* to use retrieved context via special tokens (`[Retrieve]`, `[IsRel]`, `[IsSup]`). More accurate but more complex to train.

### Multi-Hop Retrieval
For questions requiring multiple reasoning steps ("Who founded the company that acquired X?"), a single retrieval is insufficient. Iteratively retrieve: answer intermediate question → use answer to formulate next query.

### Query Decomposition
Split a complex question into sub-questions, retrieve for each, merge results. Often implemented with chain-of-thought prompting.

---

## Common Interview Angles

1. "Explain how you would build a RAG system for answering questions over 1M internal company documents."
2. "What are the tradeoffs between BM25 and dense retrieval? When would you use each?"
3. "How would you evaluate a RAG system?"
4. "Your RAG system is returning correct documents but the LLM still gives wrong answers. What do you investigate?"
5. "When would you prefer fine-tuning over RAG? Give a concrete example."
6. "How do you handle documents that are updated daily in a RAG system?"

---

## Sources
- [[ml/concepts/embeddings]]
- [[ml/concepts/llm-fundamentals]]
- [[ml/patterns/rag-pattern]]
