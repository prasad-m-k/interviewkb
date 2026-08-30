---
uid: c765e1a0-97d8-4376-9673-5f10e5b460b5
---

# AI Gateway Pattern

**Topic:** [[solution-arch/topics/llm-application-architecture]], [[solution-arch/topics/openai-platform-architecture]]
**Related concepts:** [[solution-arch/concepts/api-gateway]], [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/model-context-protocol-mcp]]
**Related patterns:** [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/bulkhead]]

## What it solves

Without a central layer, every application team that wants to call an LLM ends up embedding a model provider's SDK directly, hardcoding API keys, reinventing rate-limit handling, and having no shared visibility into cost or usage. The AI Gateway is the [[solution-arch/concepts/api-gateway]] pattern applied to LLM traffic specifically: a single chokepoint all model calls pass through, giving the organization centralized control over routing, cost, security, and observability without every team rebuilding it.

## Reference Architecture

```
┌────────────────────────────────────────────────────────────┐
│                       AI GATEWAY                                │
├────────────────────────────────────────────────────────────┤
│  Application / Agent teams                                       │
│         │  (single internal API, provider-agnostic)               │
│         ▼                                                          │
│  ┌────────────────────────────────────────────────────┐            │
│  │  AuthN/Z (internal service identity, not per-team       │        │
│  │  provider API keys)                                       │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Model routing (cost/capability-based — see              │        │
│  │  [[solution-arch/topics/openai-platform-architecture]])    │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Guardrails: input/output moderation, PII redaction,        │        │
│  │  prompt-injection classifier (see                             │        │
│  │  [[solution-arch/concepts/ai-guardrails-and-safety]])         │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Rate limiting & quota per team/application (see              │        │
│  │  [[solution-arch/concepts/rate-limiting]])                     │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Response/semantic caching (see                                 │        │
│  │  [[solution-arch/topics/llm-application-architecture]])         │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Circuit breaker + fallback per provider (see                   │        │
│  │  [[solution-arch/patterns/circuit-breaker]])                    │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Cost metering & attribution per team/feature (see              │        │
│  │  [[solution-arch/topics/cost-architecture-finops]])             │        │
│  ├────────────────────────────────────────────────────┤            │
│  │  Full request/response tracing (see                             │        │
│  │  [[solution-arch/concepts/llm-observability-and-evals]])         │        │
│  └────────────────────────────────────────────────────┘            │
│         │                                                            │
│         ▼                                                            │
│  Model providers: OpenAI / Azure OpenAI / Anthropic / open-weight     │
│  (self-hosted) — swappable behind the gateway without callers          │
│  changing anything                                                     │
└────────────────────────────────────────────────────────────┘
```

## Why This Isn't Just "an API Gateway" — What's Genuinely Different

```
Same as a traditional API gateway:
  - Central AuthN/Z, rate limiting, routing, observability
  - Single integration point decoupling callers from backends

New concerns unique to LLM traffic:
  - Cost metering per request is FIRST-CLASS, not an afterthought —
    token cost varies per model/request in a way a typical REST
    call's compute cost doesn't need per-call tracking for
  - Semantic/content-aware guardrails — a traditional gateway
    validates schema/auth; an AI gateway must also classify MEANING
    (is this a jailbreak attempt, does the output leak PII)
  - Model routing is a QUALITY decision, not just a load-balancing
    one — routing to the "wrong" model degrades answer quality,
    unlike routing to any healthy backend replica in a normal LB
  - Streaming responses require different connection handling
    (long-lived chunked connections) than typical request/response
    REST traffic through a gateway built for short calls
```

## Provider Failover Strategy

```
Primary: Azure OpenAI (data residency, existing enterprise AuthN)
         │
         ▼ (circuit breaker trips on sustained errors/timeouts)
Fallback: OpenAI direct, or a secondary region/deployment
         │
         ▼ (all providers unavailable)
Degrade: cached response if semantically similar query exists,
         else a graceful "service temporarily degraded" message —
         NEVER silently return a lower-quality answer without
         surfacing that degradation happened, especially for a
         high-risk-tier use case (see [[solution-arch/topics/ai-governance-responsible-ai]])
```

This is the same resilience-stack thinking as [[solution-arch/topics/microservices]]'s service resilience stack (rate limit → timeout → retry → circuit breaker → bulkhead → fallback), applied with the model provider as the "downstream dependency."

## When to Use

```
Use an AI Gateway when:
  ✅ More than one team/application calls LLM providers — centralizing
     avoids N teams each rebuilding rate-limit handling, cost
     tracking, and guardrails independently
  ✅ The organization needs cost attribution across teams/features
  ✅ Compliance requires a single point where every LLM call can be
     audited/logged consistently
  ✅ You want to be able to swap or add model providers without
     every calling application changing code

A direct SDK integration may be fine when:
  ✅ A single small team, single application, prototype/low-stakes
     use case — the gateway's centralization benefit doesn't
     outweigh its setup cost yet. Revisit once a second team/use
     case appears.
```

## Sources
- [[solution-arch/concepts/api-gateway]]
- [[solution-arch/patterns/circuit-breaker]]
