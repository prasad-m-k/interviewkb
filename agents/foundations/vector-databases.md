# Vector Databases — A Primer for Agent Builders

**Topic:** [[agents/topics/memory-and-context]], [[agents/topics/tool-use]]
**Related:** [[agents/foundations/embeddings-and-similarity]], [[agents/concepts/agentic-rag]], [[agents/concepts/memory-architectures]], [[agents/concepts/multi-agent-orchestration]], [[ml/concepts/rag]]

---

## Why this page exists

Every RAG design and every agent-memory design in this KB ([[agents/concepts/agentic-rag]], [[agents/concepts/memory-architectures]]) eventually says "store it in a vector database" without unpacking what that actually means or why a regular database won't do. This page fills that gap: what the problem is, how vector databases solve it, and how to reason about the tradeoffs when picking or explaining one.

---

## The problem a vector database solves

Once text is turned into embeddings ([[agents/foundations/embeddings-and-similarity]]), finding the most relevant ones means finding the vectors closest to a query vector — a **nearest-neighbor search**. The naive approach — compare the query against every stored vector, one at a time — works fine at a few thousand vectors and falls over completely at millions or billions:

```
Brute-force search cost:  O(N × D)   per query
  N = number of stored vectors
  D = number of dimensions per vector

At N = 10,000,000 vectors and D = 1,536 dimensions:
  ~15 billion floating-point operations — PER QUERY
```

A vector database's whole reason for existing is to avoid this: it builds a specialized **index** over the vectors so that a query can find the (approximately) closest matches by touching a tiny fraction of the total data, trading a small amount of accuracy for orders-of-magnitude speed — this is called **Approximate Nearest Neighbor (ANN)** search.

---

## Core operations

```
INGEST (write path):
  Document → Chunk → Embed → Store {vector, metadata, id} in the index

QUERY (read path):
  Query → Embed → ANN search the index → top-K nearest vectors → return with metadata
```

Every vector database exposes some version of these three operations:
- **Upsert** — insert or update a vector, its metadata (source, timestamp, tags), and an ID
- **Query (top-K search)** — given a query vector, return the K most similar stored vectors
- **Filter** — restrict the search to vectors matching metadata conditions (e.g., "only documents tagged `region: EU`"), combining exact filtering with approximate similarity search

---

## How the index actually works: ANN algorithms

### HNSW (Hierarchical Navigable Small World) — the most common in production
Builds a multi-layer graph: a sparse top layer for taking long jumps across the whole dataset, and progressively denser layers below for fine-grained local search.

```
Layer 2 (sparse):             o─────────────────────────────o
                               │                             │
Layer 1 (medium):             o──────────────o──────────────o───────────o
                               │              │              │           │
Layer 0 (all vectors):        o──────o───────o──────o───────o─────o─────o─────o
```

A query starts at the top (sparse) layer, greedily jumps toward the closest node, then drops down a layer and repeats with finer precision — like using a highway system to get roughly to the right city, then local roads to find the exact address. This is why HNSW gives strong recall with low latency, at the cost of higher memory usage (the graph itself takes space) and slower index builds.

### IVF (Inverted File Index) — cluster-then-search
Pre-clusters all vectors into groups (via k-means or similar); a query only compares against the vectors in the nearest few clusters, not the whole dataset. Faster to build and lighter on memory than HNSW, but generally lower recall unless tuned carefully (checking more clusters trades speed back for accuracy).

### LSH (Locality-Sensitive Hashing)
Hashes vectors such that similar vectors are likely (not guaranteed) to land in the same hash bucket; a query only checks vectors sharing its bucket. Simple and memory-light, but generally lower recall than HNSW at comparable speed — less common in modern production stacks.

### ScaNN (Google)
A quantization-based approach (compresses vectors before comparison) tuned for very high throughput at very large scale; the ANN index behind Vertex AI Matching Engine.

### The tradeoff every index makes

| Factor | What "better" costs you |
|---|---|
| **Recall** (did we actually find the true nearest neighbors?) | Higher recall generally costs more compute per query |
| **Query latency** | Lower latency generally costs recall or memory |
| **Memory footprint** | Graph-based indexes (HNSW) use more memory than cluster-based (IVF) |
| **Index build time / update cost** | Indexes optimized for fast queries are often slower/costlier to build or update incrementally |

There is no universally "best" index — the right choice depends on dataset size, how often data changes, and the latency budget, the same kind of tradeoff-driven decision taught throughout [[agents/concepts/agent-evaluation]] and [[agents/topics/agent-architectures]].

---

## Popular vector databases / libraries

| System | Type | Notes |
|---|---|---|
| **FAISS** | Library (not a hosted DB) | Meta's ANN library; extremely fast, in-process; you build the surrounding infrastructure yourself |
| **Pinecone** | Managed, hosted | Fully managed; popular default for production RAG; no infra to run |
| **Weaviate** | Open-source, self-hosted or managed | Built-in hybrid (keyword + vector) search, GraphQL API |
| **Milvus** | Open-source, self-hosted | Designed for very large scale, Kubernetes-native |
| **Qdrant** | Open-source, self-hosted or managed | Strong metadata filtering, Rust-based, lightweight to run |
| **Chroma** | Open-source, embedded | Popular for prototyping/local development; simple to start with |
| **pgvector** | Postgres extension | Adds vector search to an existing Postgres database — no new system to operate if you already run Postgres |
| **Vertex AI Matching Engine** | Managed (GCP) | ScaNN-based; built for billions of vectors at Google-scale |

**Practical guidance to teach:** start with the simplest option that fits the data volume (often `pgvector` if you already run Postgres, or `Chroma` for a prototype); move to a dedicated system (Pinecone, Weaviate, Milvus, Qdrant) once scale, latency, or filtering requirements outgrow it.

---

## How agents actually use vector databases

| Use case | What's stored | What's queried |
|---|---|---|
| **RAG corpus** | Embedded document chunks + source metadata | Query embedding → top-K relevant chunks — see [[agents/concepts/agentic-rag]] |
| **Long-term agent memory** | Embedded past episodes, facts, or user preferences | Current situation's embedding → most relevant past memories — see [[agents/concepts/memory-architectures]] |
| **Semantic tool selection** | Embedded tool descriptions | Current task's embedding → most relevant tools to offer the model, when the full tool library is too large to list every time |
| **Shared memory in multi-agent systems** | A common vector store multiple agents read from and write to | Any agent can retrieve context another agent produced earlier, without the coordinator having to manually relay it — an alternative to passing everything through direct handoffs in [[agents/patterns/supervisor-worker-pattern]] |

That last row matters for [[agents/concepts/multi-agent-orchestration]] specifically: a shared vector store is one way multiple agents stay loosely coordinated (each can search what others have found) without needing every piece of information to flow explicitly through the supervisor — sometimes called a "blackboard" pattern.

---

## Anticipated Questions

1. "Why not just use a regular (relational or document) database with a vector column?" — Increasingly you can — `pgvector` does exactly this. The distinction that matters is whether the database has an actual ANN *index* for similarity search, not just storage. Without one, "search by similarity" degrades to the brute-force O(N) scan shown above once your data grows past a small scale.
2. "Is ANN search guaranteed to return the true nearest neighbors?" — No — that's what the "approximate" means. A well-tuned index typically achieves high recall (e.g., 95–99% of true nearest neighbors found), trading the remaining accuracy for large speed gains. For most agent use cases (retrieving relevant context, not exact lookup) this tradeoff is the right one.
3. "How do I choose between HNSW and IVF for an agent's memory store?" — If the dataset is small-to-medium and mostly static, HNSW's higher recall and query speed usually win despite the memory cost. If the dataset is huge and frequently updated, IVF's cheaper builds and lower memory footprint are often the better trade — see the tradeoff table above.
4. "Does every agent need a vector database?" — No. If the agent's tool set is small, its memory needs are trivial, and it doesn't do RAG, a vector database adds operational complexity for no benefit. Reach for one when you actually have a similarity-search problem (retrieval, memory, semantic tool search) — the same "don't add agency you don't need" judgment from [[agents/concepts/what-is-an-agent]] applies to infrastructure too.

---

## Sources
- [[agents/foundations/embeddings-and-similarity]]
- [[ml/concepts/rag]]
- [[agents/concepts/agentic-rag]]
- [[agents/concepts/memory-architectures]]
- [[agents/concepts/multi-agent-orchestration]]
