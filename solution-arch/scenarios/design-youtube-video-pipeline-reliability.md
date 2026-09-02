# System Design: Global Video Upload, Processing & Delivery Pipeline (YouTube-Scale)

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/database-sharding]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/bulkhead]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This page is the **upload → transcode → CDN delivery** pipeline for raw video — the subsystem that populates the catalog. It is distinct from two sibling pages: [[solution-arch/scenarios/design-youtube-recommendation-system]] (ranking/serving recommendations — reads *from* the catalog this pipeline builds, a different subsystem entirely) and [[solution-arch/scenarios/design-video-image-understanding-pipeline-reliability]] (semantic *understanding*/feature-extraction over media, not encode/delivery). All three could plausibly sit behind YouTube in an interview, but each answers a different question.

---

## Step 1: Requirements

**Functional:**
- Accept video upload via resumable/chunked transfer (large files over unreliable client networks must not restart from zero on a dropped connection)
- Transcode the source into multiple renditions/bitrates for adaptive streaming
- Package renditions for adaptive delivery (HLS/DASH manifest generation)
- Serve video via CDN with popularity-aware placement
- Remain playable while transcoding for additional renditions is still in progress — progressive availability, lowest-quality rendition first, not "wait for all renditions"

**Non-Functional:**
- **Time-to-first-playable-rendition**, not full-transcode-completion, is the latency SLO that matters to the uploader — a creator should see "processing, but here's a low-res preview" quickly, not a blank wait
- **Durability:** an accepted upload must never be lost, even if transcoding fails and must be retried
- **Playback availability must stay decoupled from transcoding-pipeline health** — an outage in the transcoding fleet must not affect playback of already-processed video
- CDN cost efficiency for long-tail content (most of the catalog, by volume, gets little traffic — caching it broadly everywhere is wasted spend)
- Graceful degradation under region failure or spike traffic: transcoding backlog growing is acceptable; existing playback breaking is not

---

## Step 2: High-Level Architecture

```
Upload client
   │ resumable chunked upload
   ▼
Ingest service ──▶ durable object storage (source-of-truth, replicated)
   │ emits "video uploaded" event
   ▼
Message queue (priority: low-res-first jobs ahead of full-rendition-set jobs)
   ▼
Transcoding fleet (parallel workers, one job per rendition)
   │ each rendition written independently
   ├─▶ lowest-res rendition ready ──▶ video marked "playable" (partial)
   ├─▶ mid-res rendition ready ──────▶ manifest updated
   └─▶ full rendition set ready ─────▶ video marked "fully processed"
   ▼
Packaging (HLS/DASH manifest) ──▶ Origin storage
   ▼
CDN (popularity-aware placement)
   ▼
Playback client
```

The critical design move: the source upload lands in durable storage **before** any transcoding is attempted, and playability is a per-rendition state, not an all-or-nothing gate — so a transcoding failure on one rendition delays only that rendition, not first-playability.

---

## Step 3: Transcoding as a Parallelizable, Priority-Queued Batch Job

```
Per video: N independent rendition jobs (e.g. 144p, 360p, 720p, 1080p, 4K)
Each rendition is embarrassingly parallel — no cross-rendition dependency.

Priority ordering in the queue:
  1. Lowest-res rendition for a brand-new upload  ← unblocks first playability fastest
  2. Remaining renditions for a new upload
  3. Bulk re-transcode jobs (e.g. new codec rollout across the whole catalog)

Without explicit priority, a bulk re-transcode job (millions of videos,
no user waiting) would starve new-upload jobs (one creator, actively
waiting) in a FIFO queue — the queue must distinguish "is anyone
waiting on this" from "is this just backlog work."
```

This is a job-scheduling/priority problem structurally identical to [[solution-arch/scenarios/design-distributed-job-scheduler]] — cross-reference for the general scheduling mechanics (priority, retries, resource isolation).

---

## Step 4: CDN Strategy — Hot vs. Long-Tail Content

```
Content popularity follows a power law: a small fraction of videos
account for most view traffic; the long tail (most of the catalog,
by count) gets sparse, unpredictable traffic.

Hot content:  proactively pushed to edge caches broadly —
              predictable demand justifies the replication cost
Long-tail:    served from origin or a regional cache tier only —
              caching broadly at every edge would waste storage
              on content that's rarely re-requested from any
              given edge location

Popularity is not static — a video can go from long-tail to hot
(viral spike) in hours. The CDN placement decision must be
re-evaluated on observed traffic, not fixed at upload time.
```

See [[solution-arch/concepts/caching]] for the general eviction/placement mechanics this borrows from.

---

## Step 5: Storage Durability — Replication vs. Erasure Coding

```
Hot, recently-uploaded source video:  plain replication (e.g. 3x)
  — optimizes for fast read/rebuild, accepts higher storage cost

Cold, long-tail archived source video: erasure coding
  — lower storage overhead than full replication for the same
    durability target, at the cost of more expensive reconstruction
    on a failure (acceptable since cold data is rarely read urgently)

The crossover point (when to migrate a video's source from
replicated to erasure-coded storage) is itself a cost/access-pattern
decision, typically triggered by an "hasn't been accessed in N days"
signal.
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Playability gating | Per-rendition, lowest-res-first | Unblocks the uploader fastest; decouples full-transcode completion from first playability |
| Transcode job priority | New-upload renditions > bulk re-transcode backlog | Prevents catalog-wide maintenance jobs from starving actively-awaited uploads |
| CDN placement | Popularity-driven, re-evaluated continuously | Long-tail content broadly cached everywhere wastes spend; hot content undercached hurts latency |
| Cold storage | Erasure coding over full replication | Lower storage cost for content rarely read urgently, acceptable slower reconstruction |
| Failure isolation | Transcoding-fleet outage doesn't affect playback | Already-processed video's serving path has no runtime dependency on transcoding-fleet health |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #youtube #video-pipeline #cdn #system-design #nalsd*
