# LLM Application Architecture (LLMOps & Enterprise Integration)

**Related:** [[solution-arch/topics/ai-solution-architecture]], [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/openai-platform-architecture]]
**Concepts:** [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/prompt-engineering-and-context-design]], [[solution-arch/concepts/llm-observability-and-evals]]
**Patterns:** [[solution-arch/patterns/rag-enterprise-integration]], [[solution-arch/patterns/ai-gateway-pattern]]
**ML depth:** [[ml/concepts/rag]], [[ml/patterns/rag-pattern]], [[ml/scenarios/llm-service-design]]

> This page is the **architecture/enterprise-integration view** of LLM applications — how an LLM component fits into a larger system that has SLAs, security boundaries, cost controls, and existing infrastructure. For retrieval algorithm depth (chunking strategy detail, embedding model selection, evaluation metrics like recall@k), go to [[ml/concepts/rag]] and [[ml/patterns/rag-pattern]] — this page will not repeat that content.

---

## Where the LLM Sits in a Larger System

The most common AI SA mistake: designing the RAG/agent pipeline in isolation and forgetting it has to plug into an existing enterprise system with auth, logging, and change management already in place.

```
┌──────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE LLM APPLICATION                      │
├──────────────────────────────────────────────────────────────────┤
│  Client (web / mobile / internal tool)                             │
│         │                                                          │
│         ▼                                                          │
│  API Gateway (existing enterprise gateway — AuthN/Z via Azure AD/  │
│  MSAL, rate limiting, request logging — see [[solution-arch/concepts/api-gateway]])│
│         │                                                          │
│         ▼                                                          │
│  AI Gateway / LLM Proxy layer (NEW — see                            │
│  [[solution-arch/patterns/ai-gateway-pattern]]): model routing,     │
│  PII scrubbing, cost metering, response caching, guardrails         │
│         │                                                          │
│         ▼                                                          │
│  Orchestration layer: prompt assembly, retrieval calls, tool         │
│  calls, agent loop (see [[solution-arch/topics/agentic-ai-architecture]])│
│         │                            │                             │
│         ▼                            ▼                             │
│  Vector store / retrieval    Enterprise systems of record          │
│  (see [[solution-arch/concepts/vector-databases]])                  │
│         │                     (Oracle/JPQL backends, internal APIs,│
│         ▼                      existing microservices)              │
│  Model provider (OpenAI / Azure OpenAI / Anthropic — behind the     │
│  AI Gateway, swappable without changing callers)                    │
├──────────────────────────────────────────────────────────────────┤
│  Cross-cutting: Observability (tracing every hop, token/cost per   │
│  request), Eval pipeline (gates prompt/model changes before deploy)│
└──────────────────────────────────────────────────────────────────┘
```

**Interview framing:** if you're asked "design a RAG chatbot for our internal docs," the naive answer stops at "embed docs, retrieve, call LLM." The senior answer places that pipeline inside this larger diagram — auth, existing gateway, cost metering, observability — because that's what makes it a *Solution Architecture* answer instead of an ML prototype.

---

## Context Engineering: The Architecture Discipline Behind "Prompting"

"Prompt engineering" is often (mis)understood as wordsmithing a single prompt. At the system level, the actual discipline is **context engineering**: deciding *what information enters the model's context window, in what order, and how it's compressed* as a conversation or agent task grows. Deep mechanics: [[solution-arch/concepts/prompt-engineering-and-context-design]]. Architecture-level concerns:

```
Context window is a FINITE, EXPENSIVE, SHARED resource:

┌─────────────────────────────────────────────────────┐
│                 CONTEXT WINDOW BUDGET                 │
├─────────────────────────────────────────────────────┤
│ System prompt / instructions        (fixed cost)      │
│ Tool definitions                    (fixed cost,       │
│                                       scales with       │
│                                       # of tools)       │
│ Retrieved documents (RAG)           (variable —        │
│                                       truncate/rank)    │
│ Conversation history                (grows over time — │
│                                       must summarize    │
│                                       or window)        │
│ Tool call results so far            (grows per agent   │
│                                       loop iteration)    │
│ Room for the model's own output     (reserved,         │
│                                       must not be       │
│                                       squeezed to zero) │
└─────────────────────────────────────────────────────┘

Every token in every one of these categories is a LATENCY cost and
a MONEY cost, every single request. This is why "just add more
context" is not architecturally free the way adding a database
column is.
```

**Techniques an architect chooses between (not implements by hand):**
```
- Sliding window: keep only the last N turns verbatim
- Summarization: periodically compress old turns into a running
  summary, injected instead of raw history
- Retrieval-based memory: don't keep full history in context at
  all — retrieve only the relevant past turns via vector search
  when needed (same mechanics as RAG, applied to conversation
  history instead of documents)
- Prompt caching: providers cache the (unchanging) prefix of a
  prompt (system prompt + tool defs) server-side, cutting cost
  and latency on repeated calls that share a prefix — architect
  prompts with STATIC content first, VARIABLE content last, to
  maximize cache hit rate
```

---

## LLMOps: The CI/CD Equivalent for LLM Applications

Traditional CI/CD gates code changes with tests. LLM applications need an equivalent gate for **prompt changes, model version upgrades, and retrieval config changes** — because all three can silently degrade quality with no compiler error to catch it.

```
┌────────────────────────────────────────────────────────────┐
│                     LLMOps PIPELINE                          │
├────────────────────────────────────────────────────────────┤
│  Change proposed (prompt edit / model swap / new tool /      │
│  chunking strategy change)                                   │
│         │                                                     │
│         ▼                                                     │
│  Run against FIXED EVAL SET (golden examples + edge cases +  │
│  known past failure cases — a regression suite)               │
│         │                                                     │
│         ▼                                                     │
│  Score: exact-match / rubric-based / LLM-as-judge             │
│  (see [[solution-arch/concepts/llm-observability-and-evals]]) │
│         │                                                     │
│    ┌────┴────┐                                                │
│   PASS      FAIL                                              │
│    │          │                                               │
│    ▼          ▼                                               │
│  Canary    Block deploy, surface diff to reviewer              │
│  rollout                                                       │
│  (small %                                                      │
│  of live                                                       │
│  traffic)                                                      │
│    │                                                            │
│    ▼                                                            │
│  Monitor production metrics (latency, cost/request,             │
│  user feedback/thumbs-down rate, escalation rate)                │
│    │                                                            │
│    ▼                                                            │
│  Full rollout, or rollback if canary regresses                  │
└────────────────────────────────────────────────────────────┘
```

This is directly analogous to [[solution-arch/patterns/blue-green-canary]] — same deployment-safety principle, applied to a prompt/model artifact instead of a code artifact.

---

## Streaming and Latency Architecture

LLM responses are generated token-by-token; user-perceived latency architecture differs from a typical REST API:

```
Without streaming:
  Request ──────────(full generation time, e.g. 8s)──────────▶ Response
  User sees NOTHING for 8 seconds → feels broken even if
  "technically" within an SLA.

With streaming (Server-Sent Events / chunked response):
  Request ──▶ token ▶ token ▶ token ▶ ... ▶ token ▶ [done]
  User sees output appearing within ~300ms-1s (time-to-first-token)
  even though total generation still takes 8s.

Architecture implication:
  - Time-to-first-token (TTFT) and tokens-per-second are the two
    latency metrics that matter for LLM APIs — NOT just total
    request latency, which is what traditional APM tools assume.
  - Streaming requires your API gateway / load balancer to support
    long-lived chunked connections (disable response buffering,
    watch idle timeouts) — a real infra gotcha when retrofitting
    an existing enterprise gateway built for short REST calls.
  - Agentic loops with multiple sequential LLM calls compound
    latency — each tool-call round trip adds a full model-latency
    hop. Architect for this with parallelization (see
    [[solution-arch/patterns/agentic-workflow-patterns]]) wherever
    steps don't have a data dependency on each other.
```

---

## Caching Strategy for LLM Applications

```
Layer 1 — Exact-match response cache
  Identical (prompt + context) → return cached response.
  High hit rate for FAQ-style, low for open conversation.

Layer 2 — Semantic cache
  Cache keyed by embedding similarity, not exact string match —
  "What's your return policy?" and "How do returns work?" hit the
  same cached answer. Requires a similarity threshold tuned to
  avoid returning a stale answer to a subtly different question.

Layer 3 — Provider-side prompt caching
  The model provider caches the unchanging PREFIX of your prompt
  (system instructions, tool defs, static context) — cuts cost/
  latency without any semantic risk since it's an exact-prefix
  mechanism, not approximate matching.

Layer 4 — Retrieval cache
  Cache embedding lookups / retrieved chunks for repeated or
  similar queries, separate from caching the final LLM response.
```

Same general caching taxonomy as [[solution-arch/concepts/caching]], with an added semantic-similarity layer that has no analogue in traditional web caching — and a new risk: a stale semantic-cache hit is a *silent correctness bug*, not a stale-page annoyance, so cache invalidation policy needs the same rigor as the guardrail stack in [[solution-arch/topics/agentic-ai-architecture]].

---

## Sources
- [[ml/concepts/rag]]
- [[ml/patterns/rag-pattern]]
- [[ml/scenarios/llm-service-design]]
- [[solution-arch/sources/designing-data-intensive-applications]]
