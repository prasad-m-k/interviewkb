# System Design: AI Foundations & Large-Scale Data Platforms

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/federated-query-engines]], [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/idempotency]], [[solution-arch/concepts/distributed-consensus]]
**Patterns:** [[solution-arch/patterns/rag-enterprise-integration]], [[solution-arch/patterns/ai-gateway-pattern]]

> Track: **AI Foundations** / large-scale data systems intersecting with AI — querying massive unstructured/structured data platforms (F1 Query-style distributed SQL, vector-relational hybrids). Interviewers here are testing whether you can bridge analytical engines, distributed databases, and high-throughput ML workloads — not whether you know a specific vendor's API. Expect the distributed-scale and consistency challenges of a system like F1 combined with the storage, embedding-retrieval, and low-latency demands of modern AI infra. Background on the query-engine half of this: [[solution-arch/concepts/federated-query-engines]].

---

## 1. Multimodal RAG Metadata Store

**The Prompt:** Design a globally distributed metadata and vector-indexing catalog to back a real-time enterprise RAG system serving millions of concurrent LLM inference requests. The system must tie structured user permissions and relational business data to unstructured vector embeddings stored across petabytes.

**What they want to hear:**
- Extending relational co-location concepts — table interleaving, per [[solution-arch/concepts/federated-query-engines]] — to store chunked-text vectors physically alongside their parent document's metadata, minimizing network hops during hybrid searches that combine a SQL filter with vector similarity.
- A consistency story for permissions: when a document's ACL changes in the transactional layer, the vector index must reflect it without global lock contention. In practice this means the ACL check happens as a filter joined at query time from the authoritative transactional source (or a bounded-staleness replica with an explicit staleness SLA), never baked into the vector index itself — baking permissions into embeddings means a revoked ACL requires a full re-index instead of a metadata update.

**Key design decision:** keep authorization out of the embedding space entirely. Treat the vector index as a similarity-ranking engine and the transactional store as the single source of truth for "who can see this," joined at read time. This is the same access-control-aware pattern used in [[solution-arch/patterns/rag-enterprise-integration]].

---

## 2. High-Throughput Feature Store Backed by Distributed SQL

**The Prompt:** Design a real-time ML feature store that handles millions of online feature lookups per second for low-latency model inference, while simultaneously supporting massive batch write pipelines from offline training jobs.

**What they want to hear:**
- Read/write isolation: sharded key spaces plus optimistic concurrency control (see [[solution-arch/concepts/database-sharding]] and [[solution-arch/concepts/distributed-consensus]]) so a large batch backfill from an offline training job cannot starve the online serving path's read latency.
- Hotspot handling: when one entity (a viral video ID, a trending user profile) suddenly draws millions of simultaneous feature lookups, the fix is the same one used for any hot key — request coalescing, a short-TTL cache in front of the store (see [[solution-arch/concepts/distributed-caching]]), and/or splitting the hot key across virtual sub-keys — not just "add more shards," which doesn't help a single hot key.

**Key design decision:** physically separate the online (low-latency, point-lookup) and offline (high-throughput, batch-write) serving paths — typically two stores fed by the same pipeline — rather than one store serving both access patterns, which is where the read/write skew comes from in the first place.

---

## 3. Federated Query Engines over Heterogeneous AI Data Sources (F1 Query Style)

**The Prompt:** Design a distributed, declarative query processing engine (similar to F1 Query) that can execute a single SQL query joining real-time relational transactional data, object storage data lakes (Parquet/ORC files), and vector embeddings residing in remote model-serving endpoints.

**What they want to hear:**
- Cost-based optimization: how the optimizer decides whether to push a predicate — a vector similarity threshold, a metadata filter — down to the remote storage system or evaluate it in-memory in a worker node, based on selectivity estimates and what the remote source can natively filter.
- Tail-latency handling: what happens when an external model endpoint or remote object store hangs mid-DAG. Hedged requests, per-leaf timeout budgets, and partial-result streaming, so one slow dependency doesn't block the whole distributed plan.

Full mechanics: [[solution-arch/concepts/federated-query-engines]].

---

## 4. Zero-Downtime Schema Evolution for Rapidly Changing AI Models & Embeddings

**The Prompt:** ML teams are constantly iterating on embedding models, changing vector dimensions (e.g. 768-dim → 3072-dim) across billions of rows in a production database. Design a multi-phase migration strategy that avoids taking the AI platform offline.

**What they want to hear:**
- Multi-phase asynchronous schema change: add the new column/index in a non-blocking phase, backfill in the background, only cut over reads once the backfill is verified complete — never a blocking ALTER on a billion-row table.
- Dual-write during the transition: new writes populate both the old and new vector columns; background workers re-index existing partitions incrementally; inference services are able to query either structure during the transition window, selected by a feature flag or a per-row "embedding_version" marker rather than a hard cutover.
- Rollback path: because both structures coexist during the transition, a bad new embedding model can be rolled back by flipping the version flag back, without a data-loss migration in reverse. This is the same idempotent-migration discipline as [[solution-arch/concepts/idempotency]] applied to schema, not just to individual writes.

---

## 5. LLM Inference Workload Isolation and Resource Quotas

**The Prompt:** A multi-tenant enterprise platform shares a distributed database and query execution layer between standard OLTP applications and heavy analytical AI agent workflows. An automated agent triggers a massive recursive query graph that saturates cluster IOPS. Design hard isolation and resource governance.

**What they want to hear:**
- Token-bucket rate limiting (see [[solution-arch/concepts/rate-limiting]]) and priority queues that structurally separate latency-sensitive transactional execution from batch/analytical vector scans, so one tenant's runaway agent can't starve another tenant's OLTP traffic.
- Pre-flight query cost estimation: reject unindexed or unbounded analytical queries *before* they run and consume cluster-wide memory, rather than detecting the problem after the cluster is already saturated. This is the same bulkhead idea as [[solution-arch/patterns/bulkhead]] applied to query admission rather than service calls.
- For agent-specific runaway risk specifically (an agent recursively generating queries with no bound), a hard cap on agent-issued query count/cost per session is a second, independent backstop — don't rely on cluster-level rate limiting alone to catch a single agent's degenerate loop.

---

## Sources
- Prompted scenario set on AI Foundations / F1 Query-style system design tracks

*Tags: #google #ai-foundations #f1-query #system-design #vector-database*
