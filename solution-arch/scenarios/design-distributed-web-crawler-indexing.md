# System Design: Large-Scale Distributed Web Crawler & Incremental Indexing

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/cqrs]], [[solution-arch/concepts/database-sharding]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

---

## Step 1: Requirements

**Functional:**
- Crawl and discover new/changed pages across a URL frontier at billions-of-pages scale
- Respect per-domain politeness limits (never overwhelm a single site's server)
- Deduplicate near-identical content (mirrors, tracking-parameter variants, boilerplate-only diffs)
- Incrementally update the search index as new/changed content is crawled — not a full rebuild per crawl cycle
- Serve a consistent index snapshot to search queries even while updates are in flight

**Non-Functional:**
- Crawl throughput sufficient to cover billions of pages within a bounded overall freshness window
- **Adaptive freshness** — bounded staleness proportional to each page's *observed* change rate, not one fixed global TTL for the whole web
- **Index consistency** — a search query must never see a partially-applied update (some new content indexed, some stale, inconsistently) even while ingestion is continuously running
- Fault tolerance: a crawler-fleet or indexer-node failure degrades throughput (crawl gets slower), never correctness (search never returns wrong or half-updated results)

---

## Step 2: High-Level Architecture

```
URL Frontier (priority queue)
   │  priority = f(estimated change frequency, page importance)
   ▼
Crawler fleet (parallel workers, sharded by domain)
   │  per-domain rate limiter enforces politeness
   ▼
Fetched page
   │
   ├─▶ Dedup check (content hash / shingling) ──▶ near-duplicate? discard
   │
   ▼
Content parsing + link extraction
   │  new links discovered ──▶ back into Frontier
   ▼
Incremental Indexer (write side)
   │  builds new index segment/delta
   ▼
Index snapshot versioning
   │  atomic pointer swap to new consistent snapshot
   ▼
Search-serving fleet (read side) ──▶ query results
```

The write path (crawl → parse → index) and the read path (serve queries) are explicitly separated — the same [[solution-arch/concepts/cqrs]] shape applied to a search index: writes build up a new, internally-consistent version of the index; reads only ever see a *published*, complete version, never a version still being assembled.

---

## Step 3: Frontier Management — Priority, Not FIFO

```
Naive: crawl URLs in discovery order (FIFO). Wastes crawl budget
       re-fetching low-value, rarely-changing pages at the same
       rate as high-value, fast-changing ones.

Better: priority score per URL = f(estimated importance, estimated
        change frequency, time since last crawl)

  A news homepage: high change frequency → recrawled every few minutes
  A static reference page: low change frequency → recrawled weekly/monthly

Change-frequency estimate is itself learned from crawl history —
each observed diff (or lack of one) updates that page's estimate,
so the schedule adapts rather than being manually tuned per site.
```

---

## Step 4: Politeness — Per-Domain Rate Limiting

```
Uncontrolled crawl concurrency against a single small site's server
looks indistinguishable from a denial-of-service attack.

Fix: a per-domain token bucket / rate limiter, independent of
global crawl throughput — the crawler fleet's AGGREGATE throughput
can be enormous while any single domain sees a bounded, polite
request rate.
```

See [[solution-arch/concepts/rate-limiting]] for the general token-bucket/sliding-window mechanics; here the key is applying the limiter's key per-domain rather than globally or per-crawler-worker, so no single worker accidentally exceeds a domain's budget by working in parallel with other workers on the same domain.

---

## Step 5: Deduplication — Near-Duplicate Content

```
Exact-duplicate detection: content hash (fast, catches byte-identical pages
                            — mirrors, cached copies)
Near-duplicate detection:  shingling + similarity hashing (e.g. minhash)
                            — catches pages that differ only in boilerplate,
                            ads, or tracking-parameter variants of the
                            same URL

Dedup happens BEFORE indexing, not after — indexing near-duplicate
content wastes index storage and, worse, can make near-identical pages
compete with each other in search ranking for the same query.
```

---

## Step 6: Index Consistency Under Node Failure

```
Problem: an indexer node crashes mid-update. If the read-serving
         fleet queries the index directly and sees a partially-applied
         delta, results can be internally inconsistent — some pages
         reflect the new crawl, others don't, with no coherent
         "as-of" point in time.

Fix: index updates build a new, complete snapshot/segment off to
     the side; the read-serving fleet is switched to the new
     snapshot only via an ATOMIC pointer swap, after the full
     segment is verified complete. A crash mid-build simply means
     the new snapshot never gets published — the old snapshot keeps
     serving unaffected. Slightly-older-but-internally-consistent
     beats fresher-but-partially-updated, every time.
```

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Crawl scheduling | Priority by importance × change-frequency, not FIFO | Concentrates crawl budget where freshness actually matters |
| Politeness enforcement | Per-domain rate limiter, independent of global throughput | Aggregate crawl scale doesn't come at the cost of any single site's availability |
| Dedup | Content-hash + shingling before indexing | Avoids wasted index storage and ranking competition between near-identical pages |
| Index publishing | Atomic snapshot pointer swap, not in-place mutation | Read-serving fleet never observes a partially-applied update |
| Node-failure handling | Failed indexer build simply doesn't publish | Correctness preserved by construction — a crash can only delay freshness, never corrupt served results |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #web-crawler #search-index #system-design #nalsd*
