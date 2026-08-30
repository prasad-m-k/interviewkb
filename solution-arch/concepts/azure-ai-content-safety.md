# Azure AI Content Safety

**Topic:** [[solution-arch/topics/ai-governance-responsible-ai]]
**Related:** [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/topics/openai-platform-architecture]], [[solution-arch/companies/microsoft-coreai-responsible-ai]]
**Tags:** #ResponsibleAI

## What it is

Azure AI Content Safety is Microsoft's managed content-moderation product — the concrete, productized implementation of the generic guardrail pipeline described in [[solution-arch/concepts/ai-guardrails-and-safety]]. It's owned by the same CoreAI org that also ships Azure OpenAI, Model as a Service, and Azure ML, so an interviewer for a Responsible AI role at Microsoft will expect you to know it as a specific product, not just the general "guardrails" concept. It ships as a standalone API (usable with any model, not just Azure OpenAI) and is also wired in as the default, hard-to-fully-disable content filter sitting in front of every Azure OpenAI model call.

## How it works

```
┌────────────────────────────────────────────────────────────────────┐
│               AZURE AI CONTENT SAFETY — CAPABILITIES               │
├────────────────────────────────────────────────────────────────────┤
│ Text/Image moderation                                              │
│    4 harm categories: Hate, Sexual, Violence, Self-Harm            │
│    Severity: 0 / 2 / 4 / 6 (grouped low→high, not a continuous     │
│    0-100 score) — each category scored independently, so content   │
│    can be high-severity Violence but zero Hate                     │
├────────────────────────────────────────────────────────────────────┤
│ Prompt Shields                                                     │
│    Two sub-detectors:                                              │
│    - User Prompt attacks: direct jailbreak attempts in the user's  │
│      own input ("ignore previous instructions...")                 │
│    - Document attacks: indirect/injected instructions hidden in    │
│      retrieved documents, emails, web content the model ingests —  │
│      the productized defense for the indirect-injection threat in  │
│      [[solution-arch/concepts/ai-guardrails-and-safety]]           │
├────────────────────────────────────────────────────────────────────┤
│ Groundedness detection                                             │
│    Checks whether a model's output is actually supported by the    │
│    source document(s) it was given (RAG context) — a productized   │
│    hallucination detector, distinct from a general fact-check      │
├────────────────────────────────────────────────────────────────────┤
│ Protected material detection                                       │
│    Flags output that reproduces copyrighted text (code, lyrics,    │
│    articles) verbatim or near-verbatim — an IP-risk guardrail, not │
│    a harm-category one                                             │
├────────────────────────────────────────────────────────────────────┤
│ Custom categories                                                  │
│    Define an org-specific harm category (e.g., "financial advice   │
│    without disclaimer" for a fintech) beyond the 4 built-in ones   │
└────────────────────────────────────────────────────────────────────┘
```

### Where it sits in the request path

```
User input
    │
    ▼
Prompt Shields (jailbreak / injection check)  ──► blocked if flagged
    │
    ▼
Model call (Azure OpenAI or any other model)
    │
    ▼
Output harm-category classification            ──► filtered/annotated
    │                                                if severity ≥ threshold
    ▼
Groundedness check (if RAG context was used)    ──► flagged if ungrounded
    │
    ▼
Protected material check                        ──► flagged if verbatim match
    │
    ▼
Response returned to user (or blocked/redacted, per configured
severity threshold per category)
```

On Azure OpenAI specifically: the harm-category filter is **on by default and not fully removable** without a Microsoft-approved modified-content-filter application — a deliberate platform-level decision, not just an app-level opt-in. This is the concrete mechanism behind the "governance-by-design" framing in [[solution-arch/topics/ai-governance-responsible-ai]]: the provider, not just the customer, enforces a guardrail floor.

## Complexity

Not algorithmic — it's a synchronous classifier API call, so the architectural cost is the same as any guardrail hop in [[solution-arch/concepts/ai-guardrails-and-safety]]: added latency (typically tens of milliseconds per check, multiplied by however many checks run in the pipeline) and a precision/recall tuning decision per severity threshold.

## When to use

```
- Any Azure OpenAI deployment: the harm-category filter is running
  whether you explicitly integrated it or not — know its default
  behavior even if you never call the standalone API directly.
- Any RAG system on Azure: groundedness detection is the direct
  product answer to "how do you catch hallucination in production,"
  distinct from eval-time testing in [[solution-arch/concepts/llm-observability-and-evals]].
- Any agent ingesting untrusted documents/web content: Prompt
  Shields' document-attack detector is the productized version of
  the "treat retrieved content as untrusted data" principle.
- Multi-model / multi-provider systems: because it's a standalone
  API, it can front a non-Azure-OpenAI model too, making it a
  candidate guardrail layer even in a heterogeneous model estate.
```

## Common interview angles

```
Q: "Why 4 discrete severity levels (0/2/4/6) instead of a continuous
    score?"
A: Discrete levels map cleanly to policy: "block at severity ≥ 4 for
   Hate, but allow up to 6 for Violence in a fiction-writing app" is
   an auditable, explainable policy decision. A continuous score
   pushes the threshold-setting judgment call onto every consuming
   team instead of standardizing it — the same reasoning as picking
   a fixed operating point on a precision/recall curve
   ([[ml/concepts/precision-recall-auc]]) rather than shipping the
   raw probability.

Q: "A customer complains a legitimate creative-writing request got
    blocked by the Violence filter. What's your first move?"
A: Frame it as the same false-positive/false-negative cost trade-off
   from [[solution-arch/concepts/ai-guardrails-and-safety]] — check
   if the severity threshold is miscalibrated for this use case
   (creative writing legitimately touches higher-severity content
   than, say, customer support), and whether a custom category or
   per-scenario threshold override is the right fix versus loosening
   the global default.

Q: "How does groundedness detection differ from just re-running an
    eval?"
A: Evals ([[solution-arch/concepts/llm-observability-and-evals]]) are
   typically run offline/pre-deploy against a curated test set.
   Groundedness detection runs inline, per production request,
   against that specific request's actual retrieved context — it
   catches hallucination the offline eval suite never saw because
   the eval set can't cover every real-time retrieval combination.
```

## Examples

```
Config for a customer-support RAG bot on Azure OpenAI:
  - Prompt Shields: ON (both user + document attack detection)
  - Hate/Violence/Self-Harm: block at severity ≥ 2 (low tolerance —
    general consumer audience)
  - Sexual: block at severity ≥ 2
  - Groundedness: flag (not hard-block) responses scoring ungrounded,
    route to human review queue instead of auto-blocking (avoids
    over-blocking valid answers phrased differently than the source)
  - Protected material: block on any match (IP risk, zero tolerance)
```

## Sources
- [[solution-arch/concepts/ai-guardrails-and-safety]]
- [[solution-arch/topics/openai-platform-architecture]]
- [[solution-arch/topics/ai-governance-responsible-ai]]
