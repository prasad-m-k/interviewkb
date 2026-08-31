# Federated Query Engines (F1 Query Style)

**Topic:** [[solution-arch/topics/ai-solution-architecture]], [[solution-arch/topics/data-architecture]]
**Related:** [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/scenarios/ai-data-platform-system-design]]

## What it is
A federated query engine is a distributed, declarative SQL processing layer that executes a single query across heterogeneous storage systems — transactional relational stores, object-storage data lakes, and increasingly vector/embedding stores — without requiring the data to be physically co-located first. Google's F1 Query is the canonical example: it compiles one SQL statement into a distributed execution DAG (directed acyclic graph) of worker-node operators spanning Spanner-backed OLTP tables, analytical tables, and external sources. In AI-infrastructure interviews the same shape gets asked with a twist: the third data source is a vector index or a remote model-serving endpoint instead of (or alongside) a data lake.

## How it works
- **Table interleaving.** Related rows across parent/child tables — e.g. a document row and its chunk-embedding rows — are physically co-located on the same server/shard, so a join that would otherwise require a network shuffle becomes a local scan.
- **Distributed execution DAG.** The optimizer compiles SQL into a graph of operators (scan, filter, join, aggregate) distributed across worker nodes; independent branches of the DAG run in parallel.
- **Cost-based optimization (CBO).** Per predicate, the optimizer decides whether to push the filter down to the remote source or evaluate it in-memory in a worker node, based on estimated selectivity and what the remote source can natively filter. For a hybrid AI query this includes deciding whether a similarity threshold or a metadata filter should run inside the ANN index (pre-filter) or after retrieval (post-filter).
- **Tail-latency tolerance.** A federated plan depends on multiple remote systems, including — for AI queries — model-serving endpoints. One slow leaf can stall the whole DAG. Hedged requests (fire a duplicate request to a second replica after a latency threshold), per-leaf timeout budgets, and partial/streaming results keep one hung dependency from blocking the entire query.

## Complexity
Not a single asymptotic figure — the design goal is minimizing network hops and the DAG's critical-path latency, not the Big-O of any one operator. Table interleaving turns an O(shards) shuffle join into an O(1) local join for co-located data.

## When to use
- A single logical answer requires joining structured/relational data with unstructured or vector data that lives in different systems for good reasons — e.g. RAG metadata + embeddings + ACLs, where the vector store is optimized for ANN search and the OLTP store for transactional consistency.
- Denormalizing everything into one system isn't viable because independent teams own independent storage for independent reasons.

## Common interview angles
- "How would the optimizer decide whether to push a vector similarity filter down to the ANN index vs. apply it after retrieval?" — selectivity estimation plus whatever pre-filter support the ANN index natively offers.
- "A remote model-serving endpoint in the query plan hangs — what happens to the rest of the DAG?" — timeout budgets, hedged requests, partial-result streaming; never block the whole query on one leaf.
- "How do you avoid a network hop per row when joining a document table to its embedding rows?" — table interleaving / physical co-location.

## Examples
A single query — "return the top 5 most similar chunks to this embedding, filtered to documents the user has ACL access to, joined with last-modified metadata from the transactional store" — spans a vector index, an ACL table, and a metadata table. A federated query engine plans this as one distributed DAG instead of three round-trips glued together in application code.

## Sources
- [[solution-arch/scenarios/ai-data-platform-system-design]]
