# Embeddings and Similarity — A Primer for Agent Builders

**Topic:** [[agents/topics/memory-and-context]], [[agents/topics/tool-use]]
**Related:** [[agents/foundations/vector-databases]], [[agents/concepts/agentic-rag]], [[agents/concepts/memory-architectures]], [[ml/concepts/embeddings]]

---

## Why this page exists

"Embed the query," "cosine similarity," "semantic search" — these phrases appear throughout [[agents/concepts/memory-architectures]] and [[agents/concepts/agentic-rag]] as if the reader already knows what an embedding is. This page fills that gap at the level an agent builder actually needs. For the deeper ML treatment (Word2Vec, contextual embeddings, sentence embeddings, item/user embeddings), see [[ml/concepts/embeddings]].

---

## What it is

An embedding is a list of numbers (a vector) that represents the *meaning* of a piece of text, produced by a neural network trained specifically for this purpose (an embedding model). Text that means similar things gets mapped to vectors that are close together in that numerical space — even if the actual words used are completely different.

```
┌───────────────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│ "The invoice is overdue"  │───►│ Embedding Model  │───►│ [0.12, -0.48,       │
└───────────────────────────┘    └──────────────────┘    │  0.91, ..., 0.03]   │
                                                          │ (e.g. 1536 numbers) │
                                                          └─────────────────────┘
```

The number of dimensions (1536 in the example above) is fixed by the embedding model — common models range from a few hundred to a few thousand dimensions. Every piece of text run through the *same* embedding model lands in the *same* vector space, which is exactly what makes comparing them meaningful.

---

## Why proximity = meaning (a mental picture)

```
      king

         queen
    prince




                                     truck
                                car
                                  bicycle

(royalty terms cluster together; vehicle terms cluster together;
 the clusters sit far apart — this is what "semantic similarity
 corresponds to geometric proximity" means in practice)
```

Real embedding spaces have hundreds or thousands of dimensions, not two — this picture is a simplification for intuition. The training process that produces these models (see [[ml/concepts/embeddings]] for Word2Vec, contextual, and sentence-embedding approaches) is what causes semantically related concepts to end up near each other, without anyone hand-labeling "these words are related."

---

## Measuring similarity: cosine similarity

Once two pieces of text are both vectors, "how similar are they?" becomes a geometry question: how close together do they point? **Cosine similarity** measures the angle between two vectors, ignoring their length — it returns a score from -1 (opposite meaning) to 1 (identical meaning), with 0 meaning unrelated.

```
Query: "customer hasn't paid their bill"

Cosine similarity to candidate documents:

  "invoice payment overdue"    0.89  ███████████████████████████
  "late payment on invoice"    0.84  █████████████████████████
  "weather forecast tomorrow"  0.06  ██
```

Notice that none of the candidate documents share many exact words with the query — "hasn't paid their bill" vs. "invoice payment overdue" have almost no word overlap. This is precisely what embeddings solve that keyword search (BM25 / TF-IDF) cannot: matching by *meaning*, not by shared vocabulary. See the sparse-vs-dense retrieval comparison in [[ml/concepts/rag]].

---

## Where agents actually use this

| Use case | How it works |
|---|---|
| **RAG retrieval** | Embed the query and every document chunk; retrieve the chunks whose embeddings are most similar to the query's — see [[agents/concepts/agentic-rag]] and [[ml/concepts/rag]] |
| **Long-term memory retrieval** | Embed past episodes/facts when writing them to memory; embed the current situation when reading, and retrieve the most similar past memories — see [[agents/concepts/memory-architectures]] |
| **Semantic tool selection** | In agents with a very large tool library, embed tool descriptions and the current task, and retrieve only the most relevant tools to offer the model — rather than listing hundreds of tools in every prompt |
| **Deduplication / clustering** | Detect near-duplicate documents or group similar support tickets by comparing embedding similarity, without exact-text matching |

In every case above, the embeddings themselves are typically stored and searched using a **vector database** — see [[agents/foundations/vector-databases]] for how similarity search stays fast at millions or billions of vectors.

---

## Anticipated Questions

1. "Is an embedding the same thing as a hash?" — No, and this is a common point of confusion. A hash is designed so that *any* change to the input produces a completely different, essentially random output — useful for exact-match lookup. An embedding is designed so that *similar meaning* produces *similar* (nearby) vectors — useful for approximate, meaning-based matching. They solve opposite problems.
2. "Do I need the same embedding model for the query and the documents?" — Yes, always. Embeddings from different models live in different, incompatible vector spaces — comparing a query embedded with Model A against documents embedded with Model B produces meaningless similarity scores, even if both models are individually good.
3. "Why not just use keyword search — isn't it simpler?" — Keyword search fails exactly where embeddings shine: when the query and the relevant document use different words for the same concept (see the cosine-similarity example above). In practice, production systems often combine both (hybrid retrieval) — see [[ml/concepts/rag]].
4. "What happens if I embed a very long document as a single vector?" — You lose resolution — a single vector has to compress the "meaning" of the whole document, blurring distinct topics together. This is exactly why RAG chunks documents into smaller pieces before embedding them rather than embedding whole documents — see the chunking strategies table in [[ml/concepts/rag]].

---

## Sources
- [[ml/concepts/embeddings]]
- [[ml/concepts/rag]]
- [[agents/foundations/vector-databases]]
- [[agents/concepts/agentic-rag]]
- [[agents/concepts/memory-architectures]]
