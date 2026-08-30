# System Design: Enterprise RAG Assistant

**Difficulty:** Medium-Hard
**Concepts:** [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/prompt-engineering-and-context-design]], [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/concepts/llm-observability-and-evals]]
**Patterns:** [[solution-arch/patterns/rag-enterprise-integration]], [[solution-arch/patterns/ai-gateway-pattern]]

---

## Step 1: Requirements

**Functional:**
- Employees ask natural-language questions; system answers grounded in internal docs (HR policy, engineering wikis, product docs) spread across SharePoint, Confluence, and a ticketing system
- Answers must cite source documents
- Must respect each source system's existing access controls (an employee should never see content they couldn't already see in the source system)
- Supports follow-up questions (multi-turn conversation)

**Non-Functional:**
- 5,000 employees, ~2 queries/day average → ~10k queries/day, bursty around business hours
- P95 time-to-first-token < 2s (streaming), full answer < 8s
- Documents update continuously; staleness tolerance: 24 hours for most content, near-real-time for a small "live" subset (policy changes, active incidents)
- Auditable: must reconstruct exactly what was retrieved and answered for any given query, for 1 year
- Cost-bounded: target < $0.05 average cost per query

---

## Step 2: High-Level Design

```
┌──────────────────────────────────────────────────────────────┐
│                        INGESTION PIPELINE                        │
├──────────────────────────────────────────────────────────────┤
│  SharePoint / Confluence / Ticketing (source systems, each with   │
│  their own ACLs)                                                    │
│         │                                                            │
│         ▼                                                            │
│  Connector layer (scheduled + event-driven sync)                     │
│         │                                                            │
│         ▼                                                            │
│  PII/sensitive-data scan + redaction                                  │
│         │                                                            │
│         ▼                                                            │
│  Chunking (semantic, ~500 tokens, with overlap)                       │
│         │                                                            │
│         ▼                                                            │
│  Embed → Vector store, metadata includes:                              │
│    { source_system, source_id, allowed_groups[], last_updated }         │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                          SERVING PATH                             │
├──────────────────────────────────────────────────────────────┤
│  Employee ──▶ AuthN (Azure AD/MSAL — existing enterprise identity)  │
│         │                                                            │
│         ▼                                                            │
│  AI Gateway (see [[solution-arch/patterns/ai-gateway-pattern]]):       │
│    - rate limit per user                                               │
│    - input guardrail (moderation, injection classifier)                 │
│         │                                                            │
│         ▼                                                            │
│  Retrieval service:                                                     │
│    - embed query                                                        │
│    - ANN search, FILTERED by user's allowed_groups (enforced at         │
│      query time, never trusted to the LLM — see                          │
│      [[solution-arch/patterns/rag-enterprise-integration]])              │
│    - re-rank top-K, truncate to context budget                           │
│         │                                                            │
│         ▼                                                            │
│  Prompt assembly (system prompt + retrieved chunks, clearly              │
│  delimited as untrusted data + conversation history, summarized          │
│  beyond last 6 turns)                                                     │
│         │                                                            │
│         ▼                                                            │
│  LLM call (routed: cheap/fast model for simple factual lookups,           │
│  flagship model when the query is classified as complex/ambiguous —       │
│  see [[solution-arch/topics/cost-architecture-finops]])                    │
│         │                                                            │
│         ▼                                                            │
│  Output guardrail (PII leakage scan, format validation)                    │
│         │                                                            │
│         ▼                                                            │
│  Streamed response + citations to source documents                         │
│         │                                                            │
│         ▼                                                            │
│  Full trace logged (prompt, retrieved chunks, model version, output,       │
│  cost) — see [[solution-arch/concepts/llm-observability-and-evals]]        │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 3: Deep Dive — Access Control Enforcement

This is the question a strong interviewer will push hardest on for enterprise RAG:

```
At ingestion: capture the source system's ACL (which AD groups can
view this SharePoint page / Confluence space / ticket) as metadata
on every indexed chunk.

At query time: the retrieval query itself is filtered — 
  "top-K nearest vectors WHERE allowed_groups INTERSECTS user's groups"
  — not "retrieve top-K, then check if the user should have seen it."
  Filtering before/during the ANN search (not after) also avoids the
  failure mode where all top-K results get discarded post-hoc, leaving
  the model with too little context.

Group membership changes (an employee leaves a team): the system must
re-check current group membership at QUERY time, not rely on
potentially stale group data cached in the vector store's metadata —
cache group membership with a short TTL, not indefinitely.
```

## Step 4: Deep Dive — Freshness Strategy

```
Most content (policy docs, wikis): daily batch re-embed of changed
documents (source system's own "last modified" webhook/API triggers
re-index of just that document, not a full re-embed of the corpus).

"Live" subset (active incidents, today's policy changes): a separate,
small, frequently-refreshed index checked FIRST, with the main index
as fallback — OR exposed as a tool call to a live API instead of
RAG retrieval, since RAG content is only as fresh as the last index
run (see [[solution-arch/concepts/function-calling-and-tool-use]]).
```

## Step 5: Trade-offs & Failure Modes

```
| Decision | Trade-off |
|---|---|
| Model routing (cheap vs flagship) | Cost savings vs risk of the classifier mis-routing a complex query to a cheap model that under-answers |
| 24h staleness tolerance | Simpler, cheaper ingestion vs risk of answering with outdated info for that window |
| Citations required | Better trust/verifiability vs added prompt engineering + occasional citation-formatting failures needing a guardrail check |
| Semantic caching for repeated queries | Cost/latency win vs risk of returning a stale-but-similar cached answer to a subtly different question |
```

## Step 6: How to Know It's Actually Working

```
- Eval set of 200+ real historical questions with verified correct
  answers + expected source documents, run before every prompt/
  model/chunking change ships (see
  [[solution-arch/concepts/llm-observability-and-evals]])
- Production signal: thumbs-down rate, "escalate to a human" rate,
  citation-click-through (do users actually verify against sources?)
- Access-control audit: periodic sampling to confirm no query ever
  surfaced a chunk outside the asking user's permissions
```

## Sources
- [[solution-arch/patterns/rag-enterprise-integration]]
- [[ml/patterns/rag-pattern]]
