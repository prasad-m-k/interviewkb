# Customer Support Agent — System Design

**Difficulty:** Medium
**Topic:** [[agents/topics/agent-architectures]]
**Pattern:** [[agents/patterns/react-pattern]]
**Related:** [[agents/concepts/guardrails-and-safety]], [[agents/concepts/memory-architectures]]

---

## Problem

Design an AI agent that handles customer support conversations: answering account questions, looking up orders, and processing refunds, escalating to a human when appropriate.

---

## Clarifying Questions

- What actions can the agent take autonomously vs. must escalate? (informational lookups vs. refunds/cancellations)
- What's the acceptable latency for a response? (interactive chat implies seconds, not minutes)
- Is this single-turn or does it need to remember earlier parts of the conversation, and across sessions?
- What's the cost of a wrong autonomous action? (a wrong refund is expensive; a wrong FAQ answer is a minor annoyance)

---

## Requirements

| Type | Requirement |
|---|---|
| Functional | Answer account/order questions, process eligible refunds, escalate ambiguous or high-risk cases |
| Non-functional | Low latency (interactive chat), auditable action log, graceful escalation |
| Safety | No irreversible action without either a clear policy match or human approval |

---

## Architecture

```
User message
     │
     ▼
┌────────────────────────────────────────────────────────────┐
│                       SUPPORT AGENT                        │
│                        (ReAct loop)                        │
│                                                            │
│ Tools available:                                           │
│ - lookup_order(order_id)                [read-only]        │
│ - lookup_account(user_id)               [read-only]        │
│ - check_refund_eligibility(order_id)    [read-only]        │
│ - process_refund(order_id, amount)      [WRITE — gated]    │
│ - escalate_to_human(reason)             [always available] │
└──────────────────────────────┬─────────────────────────────┘
                               │
                   ┌───────────┴───────────┐
                   ▼                       ▼
           Refund request within   Refund request outside
           policy AND under $50    policy, or amount ≥ $50,
           threshold               or eligibility unclear
                   │                       │
                   ▼                       ▼
           process_refund() runs   escalate_to_human()
           automatically           human reviews before
                                   any write action
```

---

## Tool Design

Per the tool design checklist in [[agents/topics/tool-use]]: keep read tools broadly available, gate write tools tightly.

| Tool | Risk | Guardrail |
|---|---|---|
| `lookup_order` / `lookup_account` | Low (read-only) | None needed beyond standard access scoping |
| `check_refund_eligibility` | Low (read-only, deterministic policy check) | None |
| `process_refund` | High (moves money, irreversible) | Auto-allowed only below a $ threshold *and* a clean eligibility check; otherwise requires human approval — see [[agents/concepts/guardrails-and-safety]] |
| `escalate_to_human` | None (safe default) | Always available; the agent should be biased toward using this when uncertain |

---

## Memory Design

- **Short-term (within session):** full conversation history in context — see [[agents/concepts/memory-architectures]].
- **Long-term (across sessions):** account/order history retrieved via tool calls (source of truth lives in the backend systems, not duplicated into agent memory), plus optionally a summarized record of past support interactions for context ("this user has had 2 prior refund requests this quarter" — useful for the human reviewer, and for fraud-pattern detection).

---

## Guardrails Applied

1. **Tiered autonomy** — read actions unrestricted, low-value refunds automatic, everything else gated on human approval (see the general pattern in [[agents/concepts/guardrails-and-safety]]).
2. **Prompt injection defense** — if the agent ever reads user-supplied text as data (e.g., a pasted order confirmation), that text must be clearly delineated as data, not instructions.
3. **Escalation bias** — the system prompt should explicitly instruct the agent to escalate rather than guess when eligibility or policy is ambiguous; err toward the safe default.
4. **Full action logging** — every tool call, especially `process_refund`, logged with the reasoning trace for audit — this is also the raw material for [[agents/concepts/agent-evaluation]].

---

## Evaluation

- **Task success rate:** did the agent resolve the request correctly (informational accuracy, correct refund amount/eligibility)?
- **Escalation precision/recall:** did it escalate when it should have, and *not* escalate unnecessarily (over-escalation defeats the purpose of automation)?
- **Safety violations:** any unauthorized write action, or policy-violating refund — should be zero, monitored continuously.
- **CSAT / human eval:** tone, helpfulness — best captured with sampled human review or LLM-as-judge against a rubric ([[agents/concepts/agent-evaluation]]).

---

## Key Insight

The design problem here is not "can the agent answer questions" — it's **where exactly to draw the autonomy boundary** between actions the agent can take unsupervised and actions that require a human, and making that boundary enforced by code (tool gating), not by hoping the model behaves.

---

## Sources
- [[agents/patterns/react-pattern]]
- [[agents/concepts/guardrails-and-safety]]
- [[agents/concepts/memory-architectures]]
- [[agents/concepts/agent-evaluation]]
