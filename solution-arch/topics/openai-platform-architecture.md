# OpenAI Platform Architecture

**Related:** [[solution-arch/topics/ai-solution-architecture]], [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/llm-application-architecture]]
**Concepts:** [[solution-arch/concepts/function-calling-and-tool-use]], [[solution-arch/concepts/vector-databases]], [[solution-arch/concepts/prompt-engineering-and-context-design]]
**Patterns:** [[solution-arch/patterns/ai-gateway-pattern]]

> Note: API surfaces evolve quickly. The architectural *shapes* below (three call styles, tool-calling contract, fine-tuning economics) are stable interview knowledge even if specific model names/versions move on. Verify current model names/pricing against OpenAI's docs before quoting numbers to an interviewer or client.

---

## The Three API Surfaces — Know the Trade-offs Cold

This is the single most common "gotcha" question for an AI SA interviewing on OpenAI specifically: **when do you use Chat Completions vs the Responses API vs the Assistants API?**

```
┌────────────────────────────────────────────────────────────────┐
│ CHAT COMPLETIONS API — stateless, low-level                     │
├────────────────────────────────────────────────────────────────┤
│ • You manage all state: pass the full message history every call │
│ • You manage tool-call loop yourself (parse tool_calls, execute, │
│   append result, call again)                                     │
│ • Maximum control; maximum boilerplate                           │
│ • Best for: custom orchestration, non-OpenAI-hosted tool loops,  │
│   full control over context window management                   │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ RESPONSES API — stateful-optional, unifies prior surfaces        │
├────────────────────────────────────────────────────────────────┤
│ • Can operate statelessly (like Chat Completions) OR retain      │
│   conversation state server-side via a previous_response_id      │
│ • Built-in tools: web search, file/code retrieval, computer use   │
│   can be invoked without you implementing the tool yourself       │
│ • Designed as OpenAI's converged, forward-looking API surface     │
│ • Best for: new builds wanting built-in tool support without      │
│   managing an Assistants-style persistent object                  │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ ASSISTANTS API — fully stateful, higher-level abstraction         │
├────────────────────────────────────────────────────────────────┤
│ • OpenAI persists threads (conversation), runs (execution),       │
│   and the assistant config (instructions, tools, model) server-  │
│   side as durable objects                                         │
│ • Built-in Code Interpreter, File Search (retrieval), function    │
│   calling — the platform manages the agent loop for you           │
│ • Least control over orchestration internals; fastest to stand up │
│ • Best for: teams that want a managed agent runtime and don't     │
│   need custom orchestration logic                                 │
└────────────────────────────────────────────────────────────────┘

Decision rule an architect should be able to state out loud:
  "How much orchestration control do I need vs how much
   undifferentiated plumbing do I want OpenAI to manage for me?"

  Full custom control, multi-provider portability  → Chat Completions
  Built-in tools, converged surface, new project    → Responses API
  Fastest time-to-value, OK with vendor-managed state → Assistants API
```

**Architectural risk to flag in an interview:** any surface that persists state server-side (Responses with retained state, Assistants threads) creates a **vendor data residency and portability** dependency. If the enterprise has data residency requirements (common at regulated employers — banks, healthcare), stateless Chat Completions with your own state store may be the only compliant option, even though it's more engineering effort. This is exactly the kind of trade-off a Solution Architect is expected to surface that a prompt engineer wouldn't think to ask about.

---

## Function / Tool Calling Contract

```
Request:
  messages: [...]
  tools: [
    { name: "get_order_status",
      description: "Retrieve current status for an order ID",
      parameters: {              ← JSON Schema
        type: "object",
        properties: { order_id: { type: "string" } },
        required: ["order_id"]
      }
    }
  ]

Response (model decides a tool is needed):
  finish_reason: "tool_calls"
  tool_calls: [
    { id: "call_abc", function: {
        name: "get_order_status",
        arguments: "{\"order_id\": \"ORD-4471\"}"   ← STRING, must parse
    }}
  ]

Your code:
  1. Parse arguments JSON (validate against schema — never trust
     blindly, the model can produce malformed or adversarial input)
  2. Execute the actual function/API call
  3. Append a "tool" role message with the result, keyed to call_id
  4. Call the model again with the appended context
```

Full architectural depth (schema design, routing many tools, validation failure modes): [[solution-arch/concepts/function-calling-and-tool-use]].

**Structured Outputs** (`response_format: json_schema` / `strict: true`) constrain the model's output to conform exactly to a supplied JSON Schema at the token-sampling level, not just as a prompted request — this eliminates the "the model almost always returns valid JSON but 1 in 500 times doesn't" class of production bug. Architecturally, prefer Structured Outputs over prompting-for-JSON-and-hoping whenever downstream code parses the response programmatically (tool arguments, form extraction, classification results).

---

## Model Selection Architecture

```
                Does the task require multi-step reasoning,
                math, or code correctness under ambiguity?
                          │
            ┌─────────────┴─────────────┐
           YES                          NO
            │                            │
            ▼                            ▼
   Reasoning-tier model          Is latency/cost the
   (o-series / "thinking"        binding constraint, and
   models): slower, higher       is the task simple
   cost per token, but far       classification/extraction/
   higher accuracy on multi-     short-form generation?
   step logic. Use for:                 │
   - complex planning steps       ┌─────┴─────┐
     in an agent loop            YES          NO
   - code generation with        │             │
     correctness requirements    ▼             ▼
   - financial/legal reasoning  Small/mini   Flagship
                                 model        general model
                                 (fast, cheap, (balanced
                                 good for      cost/quality,
                                 high-volume    good default
                                 simple tasks)  for most
                                                production use)
```

**Cost architecture consequence:** a well-designed system **routes** requests to the cheapest model that clears a quality bar for that request type, rather than sending everything to the most capable (and expensive) model. This routing decision itself is often implemented as a small classifier or a cheap model call — see [[solution-arch/patterns/agentic-workflow-patterns]] (Routing pattern) and [[solution-arch/topics/cost-architecture-finops]].

---

## Fine-Tuning: What It Actually Buys You

```
Supervised Fine-Tuning (SFT)
  → Trains on (prompt, ideal completion) pairs.
  → Buys you: consistent OUTPUT FORMAT/STYLE, a narrower behavior
    (e.g. always respond in a specific JSON shape, adopt a brand
    voice, follow a rigid classification taxonomy), and often lets
    you use a SMALLER base model at equivalent task quality —
    a real cost lever, not just a quality lever.
  → Does NOT reliably buy you: new factual knowledge. The model can
    still hallucinate around fine-tuned domain facts. Use RAG for
    that instead — see decision framework in
    [[solution-arch/topics/ai-solution-architecture]].

Preference fine-tuning (DPO-style)
  → Trains on (prompt, preferred completion, rejected completion)
    triples. Buys you: alignment to a specific quality bar / style
    preference without needing a reward model — see [[ml/patterns/rlhf]]
    for the underlying ML mechanics.

When fine-tuning is the WRONG lever (common interview trap):
  ❌ "Our chatbot doesn't know our return policy" → RAG, not fine-tuning
  ❌ "We need it to stop making up facts" → fine-tuning does not fix
     hallucination reliably; grounding via retrieval + guardrails does
  ✅ "We need consistent structured output at 1/10th the cost of the
     flagship model" → fine-tune a smaller model on your task
  ✅ "We need a specific tone/format across thousands of examples
     where few-shot prompting is eating the context budget" → fine-tune
```

---

## Embeddings & Retrieval

OpenAI's embedding models turn text into fixed-length vectors for semantic similarity search — the retrieval half of RAG. Architectural concerns an SA must own (not the algorithm itself, which lives in [[ml/concepts/embeddings]] and [[ml/concepts/rag]]):

```
- Embedding dimensionality vs storage cost vs recall trade-off
- Batch vs real-time embedding pipelines for a growing document set
- Re-embedding cost when you upgrade embedding model versions
  (embeddings from different model versions are NOT directly
  comparable — a full re-index is required, not incremental)
- Where the vector index lives: managed vector DB vs pgvector
  extension vs in-memory — see [[solution-arch/concepts/vector-databases]]
```

---

## Moderation, Safety, and Rate Limits as Architecture Constraints

```
Moderation endpoint
  → Free/cheap classification pass to flag harmful content in
    user input AND model output. Architect this as a mandatory
    gate BEFORE user input reaches the primary model call, and
    again on output before it reaches the user — same shape as
    the guardrail stack in [[solution-arch/topics/agentic-ai-architecture]].

Rate limits (RPM / TPM / RPD — requests/tokens per minute/day)
  → Tiered by usage history and spend, not fixed. Architecture
    implication: implement client-side request queuing, exponential
    backoff with jitter on 429s, and — at real production scale —
    negotiate a provisioned-throughput / dedicated-capacity tier
    rather than relying on shared pool limits. This is the same
    resilience thinking as [[solution-arch/concepts/rate-limiting]]
    and [[solution-arch/patterns/circuit-breaker]], applied to your
    OWN system as the client of an external rate-limited dependency.

Multi-region / failover
  → If the OpenAI API has a regional outage, does your system
    fail open (degrade to cached/canned responses), fail closed
    (reject requests), or fail over to a secondary provider
    (Azure OpenAI, Anthropic)? This must be an explicit decision,
    documented in an ADR, not an accident of whatever the SDK
    defaults to.
```

---

## OpenAI Direct vs Azure OpenAI Service — An Enterprise Architecture Decision

Highly relevant for any architect working inside a regulated enterprise (banking, healthcare, government):

```
┌─────────────────────────┬────────────────────────────────────┐
│ OpenAI direct API        │ Azure OpenAI Service                │
├─────────────────────────┼────────────────────────────────────┤
│ Fastest access to newest │ Models land slightly after OpenAI   │
│ models                   │ direct; enterprise SLA and support  │
├─────────────────────────┼────────────────────────────────────┤
│ Data processing terms    │ Data stays within Azure tenant       │
│ per OpenAI's own policy  │ boundary; integrates with Azure AD, │
│                          │ Private Link, VNet isolation,       │
│                          │ Azure compliance certifications     │
│                          │ (HIPAA, FedRAMP, SOC 2, etc.)        │
├─────────────────────────┼────────────────────────────────────┤
│ Billing: OpenAI account  │ Billing: existing Azure enterprise  │
│                          │ agreement — often the deciding      │
│                          │ factor for procurement               │
├─────────────────────────┼────────────────────────────────────┤
│ No regional deployment   │ Choose Azure region for data         │
│ guarantee                │ residency compliance                 │
└─────────────────────────┴────────────────────────────────────┘

Enterprise default: if the organization already runs on Azure AD /
MSAL for identity (as in most large regulated enterprises), Azure
OpenAI is usually the default choice — it inherits the existing
IAM, network isolation, and compliance posture instead of building
a parallel one for a single vendor integration.
```

---

## Sources
- [[ml/concepts/llm-fundamentals]]
- [[ml/concepts/embeddings]]
- [[ml/patterns/rlhf]]
