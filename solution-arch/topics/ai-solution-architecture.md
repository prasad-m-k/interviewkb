# AI Solution Architecture — Role & Skill Map

**Related:** [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/openai-platform-architecture]], [[solution-arch/topics/llm-application-architecture]], [[solution-arch/topics/ai-governance-responsible-ai]], [[solution-arch/topics/enterprise-architecture-frameworks]], [[solution-arch/topics/cost-architecture-finops]]
**Scenarios:** [[solution-arch/scenarios/ai-solution-architect-interview-questions]]

---

## What an AI Solution Architect Is (and Isn't)

An **AI Solution Architect (AI SA)** is a Solution Architect who extends the classic SA discipline — NFRs, trade-offs, integration, security, cost — to systems where an LLM or agent is a first-class runtime component, not just a feature bolted onto a CRUD app. The job is still "design the blueprint that satisfies the business problem within constraints." What changes is the **failure model**: LLM outputs are probabilistic, non-deterministic, and can be adversarially manipulated (prompt injection) in ways a REST endpoint never could be.

```
Traditional SA problem:           AI SA problem:
"Will this API respond            "Will this API respond correctly,
 within 200ms at 10k RPS?"          and can I prove it didn't just
                                    hallucinate a wrong answer,
                                    leak a system prompt, or call
                                    a tool it shouldn't have?"
```

An AI SA is **not** the same role as:

| Role | Owns | Differs from AI SA by |
|------|------|------------------------|
| **ML Engineer** | Training pipelines, model architecture, feature stores | AI SA rarely trains models; consumes them via API/fine-tune, focuses on system integration |
| **Data Scientist** | Experimentation, model selection, statistical validity | AI SA cares about production reliability and cost, not offline metrics alone |
| **Prompt Engineer** | Prompt wording/iteration for a single use case | AI SA designs the *system* around prompts: guardrails, orchestration, fallback, observability |
| **MLOps Engineer** | CI/CD for model deployment, model registries | AI SA designs the architecture MLOps deploys into; overlaps at the LLMOps layer |

If you already hold `[[solution-arch/topics/architectural-styles]]`, `[[solution-arch/topics/nfr-quality-attributes]]`, and `[[solution-arch/topics/security-architecture]]` cold, this page is the delta on top — not a replacement.

---

## The Skill Map

```
┌───────────────────────────────────────────────────────────────────┐
│                     AI SOLUTION ARCHITECT SKILL MAP                │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 1 — Foundation (unchanged from classic SA)                  │
│  • Architectural styles, NFRs, distributed systems fundamentals    │
│  • Security architecture (AuthN/Z, zero trust, encryption)          │
│  • Integration patterns (sync/async, API design, messaging)        │
│  • Cost/FinOps, Well-Architected frameworks                        │
│  • Stakeholder communication, ADRs, trade-off framing              │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 2 — AI/LLM Platform Literacy (new, non-negotiable)           │
│  • Model landscape: capability tiers, context windows, cost/token  │
│  • Provider APIs: OpenAI, Azure OpenAI, Anthropic, open-weight      │
│  • Fine-tuning vs RAG vs prompting — when each is the right lever   │
│  • Embeddings & vector search fundamentals                         │
│  • Tokenization, context window economics                          │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 3 — Agentic Systems Design (the new frontier, weighted       │
│            heavily in 2025-2026 interviews)                        │
│  • Agent loop design (ReAct, plan-execute, reflection)              │
│  • Tool/function calling architecture, schema design                │
│  • Multi-agent orchestration (supervisor, handoff, blackboard)      │
│  • Memory architecture (short-term, long-term, episodic)            │
│  • MCP (Model Context Protocol) as the emerging tool-integration    │
│    standard                                                          │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 4 — Production Hardening (where most candidates fail)        │
│  • Guardrails: input/output filtering, jailbreak & injection defense│
│  • Evaluation: offline evals, LLM-as-judge, regression suites       │
│  • Observability: tracing, token/cost telemetry, drift detection    │
│  • Human-in-the-loop escalation design                              │
│  • AI governance: model risk mgmt, EU AI Act, data residency        │
└───────────────────────────────────────────────────────────────────┘
```

**Interview signal:** weak candidates can describe Layer 2 and 3 (they've built a demo). Strong candidates can describe Layer 4 in the same depth — because that's where enterprise AI projects actually die (uncontrolled cost, hallucinations reaching customers, no eval gate before prompt changes ship, no audit trail for a regulator).

---

## The AI SA Maturity Model (use this to frame any "how would you approach this" question)

```
Level 0 — Chatbot wrapper
  Single LLM call, no memory, no tools, no eval.
  Fails silently in production; no way to detect regression.

Level 1 — RAG-augmented assistant
  Retrieval + LLM. Reduces hallucination on closed-domain Q&A.
  Still single-turn, no tool use, manual prompt iteration.

Level 2 — Tool-using agent
  Function/tool calling added. Agent can query APIs, write to
  systems. Requires guardrails — this is where risk jumps sharply.

Level 3 — Multi-agent orchestrated system
  Specialized agents (retrieval, planning, execution, review)
  coordinated by an orchestrator. Requires formal evals, tracing,
  cost budgets per request, and human escalation paths.

Level 4 — Governed, self-improving platform
  Continuous eval pipelines, automated regression detection,
  model-agnostic routing, full audit trail, human-in-the-loop
  feedback closes back into eval sets. This is the enterprise target.
```

Most companies asking AI SA interview questions in 2026 are somewhere between Level 1 and Level 2, trying to get to Level 3. Frame your answers around **how you'd move a system one level up**, not just "how agents work in theory."

---

## Decision Framework: Prompting vs RAG vs Fine-Tuning vs Agents

This is one of the most commonly asked AI SA questions — treat it like the CAP theorem of this domain.

```
                    Does the model need NEW KNOWLEDGE
                    it wasn't trained on, or FRESH data?
                              │
              ┌───────────────┴───────────────┐
             YES                              NO
              │                                │
              ▼                                ▼
     Is the knowledge base          Does the task require
     large / changes often?         a new output FORMAT or
              │                     STYLE, or a narrow skill
     ┌────────┴────────┐            the base model does
    YES                NO           poorly (e.g. domain
     │                  │           classification)?
     ▼                  ▼                  │
   RAG            Put it in the      ┌──────┴──────┐
  (retrieval)     system prompt     YES            NO
                  (static context)   │               │
                                     ▼               ▼
                              Fine-tuning      Prompt engineering
                              (SFT / LoRA)     (few-shot, CoT,
                                                structured output)

                    Does the task require MULTIPLE STEPS,
                    TOOL CALLS, or DECISIONS about what to
                    do next based on intermediate results?
                              │
                             YES
                              │
                              ▼
                    Agentic architecture
                    (see [[solution-arch/topics/agentic-ai-architecture]])
```

**Key interview trap:** candidates reach for fine-tuning to "teach the model facts." Fine-tuning changes *behavior/style*, not *knowledge* reliably — it's notoriously bad at reliably injecting new factual knowledge (the model can still hallucinate around fine-tuned facts). RAG is almost always the right answer for "the model doesn't know about our internal docs."

---

## What's Genuinely New vs What's Relabeled

A senior interviewer will probe whether you understand this distinction — it separates architects who've internalized AI systems from those parroting vocabulary.

| Concept | Genuinely new | Old SA concept it maps to |
|---|---|---|
| AI Gateway | Partially | [[solution-arch/concepts/api-gateway]] + response caching + cost metering |
| Agent orchestrator | Partially | Saga orchestration ([[solution-arch/patterns/saga]]) but with a non-deterministic decision-maker instead of fixed workflow |
| Vector database | Yes (as a category) | Conceptually a specialized index, like [[solution-arch/concepts/database-sharding]] applies specialized partitioning |
| Guardrails | Yes | Closest analogue: input validation + WAF, but must catch semantic attacks, not just syntactic ones |
| Prompt injection | Yes | No prior analogue — a new OWASP-recognized threat class (see [[solution-arch/topics/ai-governance-responsible-ai]]) |
| Circuit breaker on a model call | No | Same [[solution-arch/patterns/circuit-breaker]], applied to a model provider as the "downstream dependency" |
| Idempotency for agent actions | No | Same [[solution-arch/concepts/idempotency]] principle, more critical because agents retry and can double-execute side effects (e.g., double-charging a refund tool) |

---

## Beyond Technical Depth: What Senior AI SA Interviews Actually Probe

```
1. Trade-off articulation
   "We could use GPT-4o for higher accuracy or a smaller model
    for 1/10th the cost — walk me through how you'd decide."
   → Answer with a decision framework, not a model name.

2. Failure mode enumeration
   "What happens when the LLM call times out mid-agent-loop?"
   → Answer with the full resilience stack (retries, circuit
     breaker, fallback to cached/deterministic response), same as
     [[solution-arch/patterns/circuit-breaker]] but applied to model calls.

3. Build vs buy vs integrate
   "Would you build your own agent framework or use LangGraph/
    a vendor's Assistants API?"
   → Weigh vendor lock-in, control over orchestration logic,
     team skill, and total cost of ownership — the same build-vs-buy
     lens applied in any SA interview, extended to AI tooling.

4. Governance and audit
   "A regulator asks why the AI denied a customer's claim. What do
    you show them?"
   → This is where AI governance ([[solution-arch/topics/ai-governance-responsible-ai]])
     becomes a hard architectural requirement, not an afterthought:
     full trace of prompt, retrieved context, model version, and output.

5. Cost at scale
   "Your RAG assistant costs $80k/month in token spend. Reduce it
    without hurting quality."
   → See [[solution-arch/topics/cost-architecture-finops]]: caching,
     prompt compression, smaller model routing, retrieval-result
     truncation.
```

---

## Sources
- [[solution-arch/sources/building-effective-agents-anthropic]]
- [[ml/concepts/llm-fundamentals]]
- [[ml/concepts/rag]]
