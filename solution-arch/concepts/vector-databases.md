# Vector Databases

**Topic:** [[solution-arch/topics/llm-application-architecture]]
**Related:** [[solution-arch/concepts/database-sharding]], [[ml/concepts/embeddings]], [[ml/concepts/rag]], [[solution-arch/patterns/rag-enterprise-integration]]

## What it is

A vector database (or vector index) stores high-dimensional numeric vectors (embeddings) and answers **nearest-neighbor** queries: "given this query vector, find the K most similar stored vectors." It's the retrieval backbone of RAG, semantic caching, and agent long-term memory. Architecturally it's a specialized index type — the same category of thing as a B-tree or inverted index, just optimized for similarity in continuous vector space instead of exact-match or range queries.

## How it works

```
Ingestion (offline / batch):
  Document ─▶ Chunk ─▶ Embed (embedding model) ─▶ Vector (e.g. 1536-dim)
                                                        │
                                                        ▼
                                              Store: [vector, metadata, id]
                                              in an Approximate Nearest
                                              Neighbor (ANN) index

Query (online / real-time):
  User query ─▶ Embed (SAME embedding model/version) ─▶ Query vector
                                                              │
                                                              ▼
                                          ANN search: top-K nearest
                                          stored vectors by
                                          cosine similarity / dot
                                          product / Euclidean distance
                                                              │
                                                              ▼
                                          Return matched chunks + metadata
                                          → feeds into LLM context
```

**Why "approximate" (ANN), not exact:** an exact nearest-neighbor search over millions of high-dimensional vectors is too slow for real-time queries. ANN algorithms trade a small accuracy loss for massive speed gains.

```
Common ANN index structures:

HNSW (Hierarchical Navigable Small World)
  → Graph-based; multi-layer graph where higher layers have fewer,
    longer-range links (like a skip list) enabling fast traversal
    to the query's neighborhood, then fine search at the base layer.
  → Best recall/speed trade-off for most workloads; higher memory
    footprint. The default choice for most production vector DBs.

IVF (Inverted File Index)
  → Partitions vector space into clusters (via k-means); at query
    time, only search the nearest few clusters instead of the whole
    space.
  → Lower memory than HNSW, faster to build, slightly lower recall
    unless tuned (nprobe = how many clusters to search).

Product Quantization (PQ)
  → Compresses vectors by splitting into subvectors, each quantized
    to a small codebook — trades some accuracy for large memory
    savings. Often combined with IVF (IVF-PQ) for billion-scale
    datasets.
```

## Complexity

```
HNSW:  Query ~O(log n), Build ~O(n log n), Memory: high (graph edges)
IVF:   Query ~O(n/nlist × nprobe), Build ~O(n), Memory: moderate
IVF-PQ: Query similar to IVF, Memory: low (compressed vectors)

Practical takeaway: HNSW for < ~10M vectors where recall matters most;
IVF/IVF-PQ when memory cost or dataset size (100M+) dominates the
decision.
```

## When to use

```
Use a dedicated vector database when:
  ✅ Semantic/similarity search is a core, high-QPS product feature
  ✅ Dataset is large enough that a brute-force scan is too slow
  ✅ You need hybrid search (vector + keyword/metadata filter) at scale

Use a Postgres extension (pgvector) instead when:
  ✅ Vector search is a secondary feature bolted onto an existing
     relational system, and dataset is small-to-medium (< a few
     million vectors)
  ✅ You want to avoid adding a new database technology to the stack
     purely for this feature — operational simplicity wins
  ✅ You need transactional consistency between vectors and
     relational data (e.g. update a record and its embedding
     atomically)

Use in-memory (e.g. a flat array + brute force) when:
  ✅ Dataset is small (thousands of vectors) — exact search is fast
     enough and avoids ANN's recall trade-off entirely
```

## Common interview angles

```
Q: "Cosine similarity vs dot product vs Euclidean distance — when
    does it matter?"
A: If embeddings are normalized to unit length, cosine similarity
   and dot product rank results identically (dot product is cheaper
   to compute). Euclidean distance behaves differently and is more
   common for non-normalized embeddings. Always match the distance
   metric to what the embedding model was TRAINED/optimized for —
   using the wrong metric silently degrades retrieval quality with
   no error thrown.

Q: "What happens when you upgrade your embedding model?"
A: Vectors from different model versions are NOT comparable — you
   cannot mix old and new embeddings in one index. Requires a full
   re-embed and re-index of the corpus, which is a real operational
   cost (compute + a cutover strategy — dual-write old/new index
   during migration, or accept a maintenance window).

Q: "How do you combine vector search with metadata filters (e.g.
    'only documents from the last 30 days')?"
A: Pre-filtering (filter candidates before ANN search — can hurt
   recall if the filtered set is small relative to index structure
   assumptions) vs post-filtering (ANN search first, then filter
   results — can return too few results if the top-K pre-filter
   discards most matches). Production vector DBs increasingly
   support filtered ANN search natively to avoid this trade-off.

Q: "Recall@K degraded after we scaled to 50M vectors — why, and
    what would you check?"
A: ANN index parameters (nprobe for IVF, ef_search for HNSW) often
   need retuning as index size grows — the same recall target
   requires searching more candidates at larger scale. Also check
   whether sharding the index (see [[solution-arch/concepts/database-sharding]])
   was done naively in a way that fragments similar vectors across
   shards, hurting per-shard recall.
```

## Examples

```
E-commerce product search: embed product descriptions, retrieve
similar products for "customers who searched this also liked..."

Enterprise RAG: embed document chunks, retrieve top-K relevant
chunks for a user's question before the LLM call.

Agent long-term memory: embed past conversation summaries, retrieve
relevant past context for a returning user (see
[[solution-arch/topics/agentic-ai-architecture]]).

Semantic caching: embed incoming queries, check for a similar past
query before making a new (expensive) LLM call.
```

## Sources
- [[ml/concepts/embeddings]]
- [[ml/concepts/rag]]
