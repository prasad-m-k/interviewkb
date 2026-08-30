# RAG — Enterprise Integration Pattern

**Topic:** [[solution-arch/topics/llm-application-architecture]]
**Related concepts:** [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/caching]], [[ml/concepts/rag]]
**Related patterns:** [[solution-arch/patterns/ai-gateway-pattern]], [[ml/patterns/rag-pattern]]

> **Scope note:** [[ml/patterns/rag-pattern]] covers the ML-engineering view — chunking strategy, embedding model choice, retrieval evaluation metrics (recall@k, MRR). This page covers the **enterprise architecture view**: how RAG integrates with existing systems of record, security boundaries, data governance, and operational concerns an SA owns that an ML engineer typically doesn't. Read both; they don't overlap.

## What it solves

Enterprises have decades of institutional knowledge locked in existing systems — SharePoint, Confluence, ticketing systems, relational databases, PDF policy documents — that a base LLM has no access to and can't be reliably fine-tuned to "know" (see the decision framework in [[solution-arch/topics/ai-solution-architecture]]). RAG lets an LLM answer questions grounded in that live, changing knowledge base without retraining, by retrieving relevant content at query time and injecting it into the prompt.

## Enterprise RAG Reference Architecture

```
┌────────────────────────────────────────────────────────────┐
│                 INGESTION (offline pipeline)                   │
├────────────────────────────────────────────────────────────┤
│  Source systems (SharePoint, Confluence, DB exports, PDFs)      │
│         │                                                        │
│         ▼                                                        │
│  Access-control-aware extraction — CRITICAL: preserve the        │
│  SOURCE SYSTEM's permission model, don't flatten it away          │
│         │                                                        │
│         ▼                                                        │
│  PII/sensitive-data classification + redaction pass                │
│         │                                                        │
│         ▼                                                        │
│  Chunking (see [[ml/patterns/rag-pattern]] for strategy detail)   │
│         │                                                        │
│         ▼                                                        │
│  Embed ─▶ Vector store (with metadata: source, access tier,       │
│           last-updated timestamp) — see                             │
│           [[solution-arch/concepts/vector-databases]]               │
│         │                                                        │
│         ▼                                                        │
│  Scheduled re-sync (documents change — a stale index silently      │
│  serves outdated policy/pricing/compliance information, which        │
│  is a correctness AND a compliance risk, not just staleness)         │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                  SERVING (online, per-request)                  │
├────────────────────────────────────────────────────────────┤
│  User query ──▶ AuthN/Z (existing enterprise identity — Azure     │
│  AD/MSAL) ──▶ retrieve ONLY documents this user is permitted        │
│  to see (access-control filter applied at retrieval time, NOT       │
│  trusted to the LLM to enforce post-hoc)                             │
│         │                                                            │
│         ▼                                                            │
│  Rank/filter top-K, truncate to context budget (see                  │
│  [[solution-arch/topics/llm-application-architecture]])               │
│         │                                                            │
│         ▼                                                            │
│  LLM call with retrieved context, delimited as untrusted DATA          │
│  (prompt injection defense — see                                      │
│  [[solution-arch/concepts/ai-guardrails-and-safety]])                  │
│         │                                                            │
│         ▼                                                            │
│  Response + CITATIONS back to source documents (enterprise           │
│  users need to verify/trust the answer — surfacing "which             │
│  document" is often a harder requirement than the answer itself)      │
└────────────────────────────────────────────────────────────┘
```

## The Access-Control Problem (the #1 enterprise RAG mistake)

```
WRONG (common failure mode):
  Index ALL documents into one shared vector store, regardless of
  who can see what in the source system. At query time, the LLM
  might retrieve and surface content the ASKING USER was never
  authorized to see in the source system — a silent, severe data
  leak that has no equivalent in a traditional search UI (where
  permission filtering is usually already enforced at the search
  index layer, but is easy to forget to replicate for a NEW RAG
  pipeline built as a separate project).

RIGHT:
  Propagate source-system ACLs into the vector store as metadata
  (e.g. allowed_groups: [...]) and enforce the filter as part of
  the retrieval query itself — never rely on the LLM to decline to
  use content it shouldn't have seen; it may not reliably do so,
  and even if it does, the sensitive content already left the
  source system's trust boundary and entered your LLM provider's
  request.
```

## Freshness & Re-Indexing Strategy

```
Static/rarely-changing corpora (finalized policy documents):
  → Batch re-index on a schedule (daily/weekly) is sufficient

Frequently-changing corpora (active tickets, live pricing):
  → Near-real-time sync (event-driven re-embed on source-system
    change events) or a hybrid: RAG over stable content + a
    TOOL CALL (not retrieval) to a live API for anything that
    must be current-as-of-now (see [[solution-arch/concepts/function-calling-and-tool-use]]
    — a live price lookup is a tool call, not a RAG retrieval,
    precisely because RAG content is only as fresh as the last
    re-index)
```

## Sources
- [[ml/patterns/rag-pattern]]
- [[ml/concepts/rag]]
- [[solution-arch/topics/security-architecture]]
