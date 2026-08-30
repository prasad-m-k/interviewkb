# Agentic AI Reference Diagrams

ASCII reference diagrams for the AI Solution Architect topics — companion to [[common-system-patterns]] and [[c4-model]], specific to LLM/agentic systems. See [[solution-arch/topics/ai-solution-architecture]] for the narrative context behind each.

---

## 1. Single-Agent ReAct Loop

```
   User Input
       │
       ▼
 ┌──────────────┐
 │  Perceive     │  conversation history + retrieved docs + tool
 │  (context)    │  results so far
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │   Reason      │  LLM decides: tool call, or final answer
 └──────┬───────┘
        │
   ┌────▼─────┐        NO
   │Tool call? ├──────────────▶ Return final answer
   └────┬─────┘
       YES
        ▼
 ┌──────────────┐
 │     Act       │  execute tool (validated input)
 └──────┬───────┘
        ▼
 ┌──────────────┐
 │   Observe     │  append result to context
 └──────┬───────┘
        │
        └──────────► back to Reason (bounded by max_iterations)
```

---

## 2. RAG Pipeline (Ingestion + Serving)

```
INGESTION (offline)                    SERVING (online, per query)

Source docs                            User query
    │                                       │
    ▼                                       ▼
Chunk                                   Embed query
    │                                       │
    ▼                                       ▼
Embed ──▶ Vector store ◀── ANN search (top-K, filtered by ACL)
                                              │
                                              ▼
                                        Rank + truncate to
                                        context budget
                                              │
                                              ▼
                                        LLM call (query +
                                        retrieved chunks,
                                        delimited as data)
                                              │
                                              ▼
                                        Answer + citations
```

---

## 3. Multi-Agent Supervisor Topology

```
                    ┌─────────────────┐
                    │   Supervisor     │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   ┌─────────────┐   ┌─────────────┐    ┌─────────────┐
   │ Specialist   │   │ Specialist   │    │ Reviewer     │
   │ Agent A      │   │ Agent B      │    │ Agent (no    │
   │ (own tools,  │   │ (own tools,  │    │ write tools) │
   │  own context)│   │  own context)│    │              │
   └─────────────┘   └─────────────┘    └─────────────┘
```

---

## 4. AI Gateway (LLM Traffic Chokepoint)

```
  App / Agent teams
         │
         ▼
┌─────────────────────────────────┐
│  AuthN/Z                         │
│  Model routing (cost/capability) │
│  Guardrails (in/out)               │
│  Rate limiting                      │
│  Response/semantic caching           │
│  Circuit breaker + fallback            │
│  Cost metering                          │
│  Tracing                                 │
└─────────────────────────────────┘
         │
         ▼
  OpenAI / Azure OpenAI / Anthropic / self-hosted
```

---

## 5. Guardrail Pipeline (Input → Model → Output)

```
User/tool input
      │
      ▼
Moderation classifier ──▶ [flagged? block/sanitize]
      │
      ▼
Injection/jailbreak classifier
      │
      ▼
MODEL CALL (untrusted content clearly delimited)
      │
      ▼
Structured/schema validation
      │
      ▼
Output moderation (harmful content, PII leakage)
      │
      ▼
[If tool call: permission check + idempotency + optional
 human-in-the-loop gate]
      │
      ▼
Response delivered
```

---

## 6. Human-in-the-Loop Checkpoint

```
Agent proposes action
      │
      ▼
Risk classifier ──▶ LOW/MEDIUM risk ──▶ Execute (+ audit sample)
      │
   HIGH risk
      │
      ▼
Pause agent state (persist context)
      │
      ▼
Surface to human: WHAT + WHY + consequence of approve/reject
      │
   ┌──┴──┐
 APPROVE REJECT/MODIFY
   │        │
   ▼        ▼
Execute   Feed correction back, resume loop
(idempotent)
```

---

## 7. MCP Integration Layer

```
  Host A ──┐                           ┌──▶ Tool Server 1 (MCP)
  Host B ──┼──▶ MCP protocol (JSON-RPC)┼──▶ Tool Server 2 (MCP)
  Host C ──┘        over stdio/HTTP    └──▶ Tool Server 3 (MCP)

  N hosts + M tool servers = N + M integrations, not N × M
```

---

## 8. LLMOps Eval-Gated Deploy Pipeline

```
Change proposed (prompt/model/retrieval config)
      │
      ▼
Run against fixed eval set (golden examples + regression cases)
      │
   ┌──┴──┐
 PASS   FAIL ──▶ Block deploy, surface diff to reviewer
   │
   ▼
Canary rollout (small % of live traffic)
   │
   ▼
Monitor production metrics (latency, cost, thumbs-down rate,
escalation rate)
   │
   ▼
Full rollout, or rollback if canary regresses
```

## Sources
- [[solution-arch/topics/agentic-ai-architecture]]
- [[solution-arch/topics/llm-application-architecture]]
