# A Good System Design Tackles the Hot Path First

**Type:** article / newsletter
**Author:** System Design Classroom (newsletter)
**URL:** https://newsletter.systemdesignclassroom.com/p/a-good-system-design-tackles-down-the-hot-path-first
**Ingested:** 2026-07-28
**Topics covered:** [[solution-arch/topics/scalability-and-reliability]], [[solution-arch/topics/nfr-quality-attributes]]

## Summary

Argues that system design should start by answering "where is the system actually hot?" rather than by reaching for a standard toolbox (Redis, read replicas, load balancers, microservices) and applying it uniformly. The article's central lens is separating reads (which want speed and can often tolerate staleness) from writes (which want correctness and must not compromise on it), then further splitting reads into **display reads** (product pages, search, recommendations — eventual consistency is invisible to the user) versus **decision reads** (checkout, inventory confirmation — staleness causes real financial/trust damage). It walks through a running e-commerce case study (99% browsing reads vs 1% catalog-management writes) to show how this classification drives concrete architecture: display reads get cache + read replicas; decision reads get a dedicated consistency path reading from the primary; writes go through a transactional [[solution-arch/patterns/outbox]]-based invalidation pipeline so cache invalidation events are published reliably alongside the write itself, with TTL retained as a safety net rather than the primary consistency mechanism. The article closes by contrasting a "tool-collecting" mindset (adding infrastructure because it's best practice) against an "architecture-first" mindset (every component justified by a named requirement), summarized as "you do not scale the whole system, you scale what is hot."

## Key Takeaways

- Reads want speed, writes want correctness — this split, decided per-endpoint, is the primary lens for every subsequent architecture decision
- Not all reads are equal: classify as display reads (eventual consistency OK) vs decision reads (must be fresh) — conflating them is what causes the classic bug where a user sees one price browsing and is charged another at checkout
- The outbox pattern applies directly to cache invalidation, not just generic event publishing: write the invalidation event in the SAME transaction as the data change, publish asynchronously, keep consumers idempotent
- TTL should be treated as a safety net / backstop for invalidation failures, never as the primary correctness mechanism
- A concrete endpoint classification matrix (traffic volume × freshness requirement × scaling strategy × revenue impact) is the actual design artifact — more useful in an interview than a generic component diagram
- "Scale what is hot" — adding caching/replicas to a low-traffic admin endpoint is pure complexity with no benefit; the same component on a high-traffic endpoint is the entire point

## What it updated
- Created: [[solution-arch/patterns/hot-path-first-design]] (full methodology, 5 recreated architecture diagrams, endpoint classification matrix, 5-step process)
- Informed: [[solution-arch/concepts/distributed-caching]] (thundering herd / stale-read-window framing), [[solution-arch/concepts/rest-api-design-principles]] (GET-as-cacheable-display-path framing)
- Cross-referenced (not duplicated): [[solution-arch/patterns/outbox]] — this article's cache-invalidation use case is now noted as an application of that existing pattern, not a new mechanism
