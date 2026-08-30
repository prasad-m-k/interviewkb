---
uid: bb3edfba-d0c3-4b00-a6e9-dcd1dbae364b
---

# Human-in-the-Loop (HITL) Pattern

**Topic:** [[solution-arch/topics/agentic-ai-architecture]], [[solution-arch/topics/ai-governance-responsible-ai]]
**Related concepts:** [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/concepts/idempotency]]
**Related patterns:** [[solution-arch/patterns/agentic-workflow-patterns]], [[solution-arch/patterns/multi-agent-orchestration]]

## What it solves

Autonomous agents can take wrong, high-cost, or irreversible actions with full confidence — an LLM has no innate signal for "I'm not sure, a human should check this." HITL inserts an explicit checkpoint where a human must approve, correct, or override an agent's decision before it takes effect, for exactly the class of actions where the cost of an error is asymmetric (much worse than the cost of a short delay for approval).

## Where to Place the Checkpoint

```
┌────────────────────────────────────────────────────────────┐
│                  AGENT ACTION RISK TIERS                      │
├────────────────────────────────────────────────────────────┤
│ Low risk (auto-execute, no HITL):                              │
│   - Read-only lookups (check order status, search knowledge      │
│     base)                                                          │
│   - Reversible, low-value actions (send an informational           │
│     notification)                                                   │
├────────────────────────────────────────────────────────────┤
│ Medium risk (auto-execute + post-hoc audit sample):              │
│   - Bounded-value actions (refund under $50, reschedule a           │
│     meeting) — execute automatically, but sample a % for           │
│     human audit to catch systematic errors                          │
├────────────────────────────────────────────────────────────┤
│ High risk (HITL checkpoint REQUIRED before execution):           │
│   - Irreversible actions (delete data, cancel a contract)           │
│   - High monetary value (refund over a threshold, large              │
│     financial transfer)                                              │
│   - Actions affecting protected/regulated outcomes (credit           │
│     decisions, employment-related actions, medical guidance)          │
│   - Any action the agent itself reports low confidence on            │
└────────────────────────────────────────────────────────────┘
```

## Implementation Shape

```
Agent decides on action ──▶ Risk classifier (rule-based or a
                              small model) ──▶ is this HIGH RISK?
                                    │
                          ┌─────────┴─────────┐
                         NO                   YES
                          │                    │
                          ▼                    ▼
                    Execute now        Pause agent state (persist
                                        context so the loop can
                                        resume exactly where it
                                        left off — same durability
                                        requirement as any long-
                                        running workflow), surface
                                        to human reviewer with full
                                        context (why the agent wants
                                        to take this action)
                                              │
                                    ┌─────────┴─────────┐
                                  APPROVE              REJECT /
                                    │                   MODIFY
                                    ▼                     │
                              Execute action        Feed correction
                                                      back to agent
                                                      context, continue
                                                      loop OR log as a
                                                      training/eval
                                                      example
```

**Idempotency requirement:** because the agent loop pauses and later resumes (possibly after a long delay, or after a retry if the pause/resume infrastructure itself fails), the eventual execution must be safe to attempt exactly once even under retry — same idempotency-key discipline as [[solution-arch/concepts/idempotency]] and as applied to tool calls in [[solution-arch/concepts/function-calling-and-tool-use]].

## Escalation Design (What Makes HITL Actually Usable)

```
A poorly designed HITL surfaces raw agent reasoning/logs to a human
reviewer, who then has to reconstruct context from scratch — this
kills the throughput benefit of automating the rest of the workflow.

A well-designed HITL surfaces:
  - WHAT action is proposed, in plain terms
  - WHY the agent proposed it (the reasoning/evidence, not just
    the tool-call arguments)
  - WHAT HAPPENS if approved vs rejected
  - A one-click approve/reject/modify action, not a free-form
    "go investigate this" task
```

## When to Use

```
Use HITL when:
  ✅ The action is high-cost-of-error per the risk tiers above
  ✅ Regulatory requirements mandate human oversight for the
     decision class (see [[solution-arch/topics/ai-governance-responsible-ai]])
  ✅ The system is new/unproven — HITL on a broader set of actions
     initially, narrowing the scope as production evals build
     confidence (a maturity ramp, not a permanent state)

Avoid over-applying HITL when:
  ❌ Gating every single action defeats the purpose of automation —
     reserve it for the genuinely high-risk tier, and rely on
     guardrails + post-hoc audit sampling for the rest
  ❌ The "human in the loop" is a rubber stamp with no real context
     to evaluate the decision — this creates automation bias
     (see [[solution-arch/topics/ai-governance-responsible-ai]]) while
     giving false confidence that oversight exists
```

## Sources
- [[solution-arch/topics/ai-governance-responsible-ai]]
- [[solution-arch/sources/building-effective-agents-anthropic]]
