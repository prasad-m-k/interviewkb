# System Design: Distributed File Storage and Sync System (Google Drive-Style)

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/bulkhead]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

---

## Step 1: Requirements

**Functional:**
- Upload and sync files across devices, transferring only changed data on subsequent syncs (not the whole file)
- Support concurrent edits to the same file from multiple devices/users with a defined conflict-resolution behavior
- Maintain a version history per file, restorable to any prior version
- Serve reads (file open, metadata listing) from the nearest region to the requesting client
- Share files/folders across users with access-control enforcement

**Non-Functional:**
- **Durability:** target class-leading durability (commonly stated as "11 nines" for this class of system) — justified by replication factor and failure-domain independence, not asserted
- **Sync latency:** a small incremental change (a few KB edited in a large file) should propagate to other devices in low single-digit seconds, not re-upload the whole file
- **Availability during a region loss:** reads and writes for unaffected users continue; users pinned to the lost region fail over without data loss
- **Consistency:** per-file **causal/session consistency** is enough (a user's own writes are immediately visible to their own next read); cross-user global strong consistency is not required and would be too costly
- **Storage efficiency:** deduplicate identical content across users' files via content-addressed chunking, not per-user copies
- **Scale:** billions of files, exabytes of data, sustained high write fan-in during business-hours peaks

---

## Step 2: High-Level Architecture

```
Client (desktop/mobile sync agent)
   │  1. chunk changed file locally (content-addressed, rolling hash)
   │  2. diff chunk hashes against last-known-synced manifest
   ▼
Sync API (regional, nearest to client)
   │  3. upload only NEW/changed chunks (dedup: skip chunks that
   │     already exist anywhere in the chunk store)
   ▼
Chunk Store (content-addressed blob storage, replicated cross-region)
   │
   ▼
Metadata Service (file → chunk-list manifest, version history, ACLs)
   │  replicated via a consensus-backed metadata store
   │  (see [[solution-arch/concepts/distributed-consensus]])
   ▼
Change Feed  ──publish──▶  other devices subscribed to this file/folder
                            (pull the new manifest, fetch only missing chunks)
```

Two independent replication concerns: **chunk data** (large, immutable once written, replicated for durability) and **metadata** (small, mutable, needs consensus for correct ordering of concurrent edits). Splitting them lets the system replicate terabytes of blob data with simple multi-region copy while running consensus only over small, frequently-updated metadata records.

---

## Step 3: Chunking and Deduplication

```
File is split into variable-size chunks using a rolling hash
(content-defined chunking, not fixed-size blocks) so a small
insertion near the start of a file shifts chunk BOUNDARIES
locally instead of shifting every subsequent fixed-size block.

Each chunk is addressed by its content hash (e.g. SHA-256).
  - Two users uploading the same file (or the same embedded
    asset in different files) produce identical chunk hashes
    → the second upload is a no-op after the hash check.
  - Sync of a small edit only needs to upload the handful of
    chunks whose hash changed, not the whole file.
```

This is the same content-addressing idea as [[solution-arch/patterns/outbox]]'s event dedup, applied to file bytes instead of message IDs: identity is derived from content, so duplicate detection is a hash lookup, not a full comparison.

---

## Step 4: Replication and the Consistency Choice

```
Chunk data:  replicate to N regions (e.g. 3+) asynchronously after
             write acknowledgment from the primary region — favors
             availability and write latency (BASE); a chunk is
             immutable once written, so async replication has no
             lost-update risk.

Metadata (file manifest, current version pointer):
             needs a total order per file — two concurrent edits
             from different devices must resolve to ONE next
             version, not two divergent ones. This is where
             [[solution-arch/concepts/acid-vs-base]] and
             [[solution-arch/concepts/cap-theorem]] actually bite:
             the metadata write path chooses consistency (a
             consensus-backed metadata store) over unconditional
             write availability, because an unresolved fork in
             the manifest itself — not just a slow write — is
             the failure mode being avoided.
```

**Durability math:** replication factor 3 across 3 independent failure domains (data centers on independent power/network) with independent annual failure probability p per replica gives an expected data-loss probability on the order of p³ — this is the standard justification for "11 nines," not a marketing number pulled from nowhere.

---

## Step 5: Conflict Resolution for Concurrent Edits

```
Two devices edit the same file offline, then both come back online.

Chosen approach: version-vector detection + conflict COPY on
divergence (what Drive/Dropbox actually do), not silent
last-write-wins and not full CRDT merge:

  1. Each device tracks the last-synced version it based its edit on.
  2. On sync, if the server's current version has moved past what
     the device based its edit on → this is a genuine conflict,
     not a fast-forward.
  3. Server accepts whichever edit arrives first as the new version;
     the LOSING edit is saved as a separate "conflicted copy" file
     rather than silently discarded.
  4. User resolves manually by comparing the two files.

Why not silent last-write-wins: silently discarding a user's edit
is a data-loss bug from the user's perspective, even though the
system behaved "correctly." Why not full CRDT/OT merge: that's the
right call for structured co-editing (e.g. a live document editor)
but is overkill and behaviorally surprising for opaque binary files
where the system can't safely auto-merge content it doesn't understand.
```

---

## Step 6: Versioning

```
Every accepted write appends a new version entry to the file's
manifest history rather than overwriting in place:

  file_id, version_n → chunk-list manifest, timestamp, author

Old versions are retained per a retention policy (e.g. 30 days of
full history, then thin out to daily snapshots) and garbage-collected:
a chunk is only deleted from the chunk store once no manifest version
across the whole system still references it (reference-counted GC).
```

---

## Step 7: Recovery from a Full Region Loss

```
Region loss detected (health checks + control-plane consensus):
  1. Traffic for that region's users fails over to the nearest
     healthy region via DNS/GSLB.
  2. Metadata store's consensus group had a quorum member in the
     lost region — remaining quorum in surviving regions continues
     serving; no metadata is lost because writes were only
     acknowledged after quorum, never after a single region.
  3. Chunk data: since replicas exist in ≥2 surviving regions
     (replication factor 3+ across regions), no chunk is
     unrecoverable.
  4. Once the region recovers, it re-joins as a replica target and
     catches up via normal replication — not a special-cased
     "disaster recovery" data path, just the steady-state
     replication mechanism running for longer.
```

The core discipline that makes this recoverable rather than catastrophic: never acknowledge a write as durable based on a single region's copy.

---

## Step 8: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Chunking | Content-defined (rolling hash), not fixed-size | Small edits shift local boundaries only; enables cross-file dedup |
| Metadata replication | Consensus-backed (strong ordering) | A forked manifest is a correctness bug, not just a staleness inconvenience |
| Chunk data replication | Async, multi-region, eventual | Chunks are immutable once written — no lost-update risk, so async is safe and cheaper |
| Conflict resolution | Conflict-copy fork, not silent LWW or full CRDT | Never silently discards user data; avoids CRDT complexity for opaque binary content |
| Versioning | Append-only manifest history + reference-counted GC | Restore-to-any-version without unbounded storage growth |
| Region-loss recovery | Quorum-based metadata + multi-region chunk replicas | No single-region write acknowledgment anywhere in the durable path |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #distributed-storage #consistency #file-sync #system-design #nalsd*
