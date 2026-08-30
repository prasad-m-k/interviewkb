# Prompt Engineering & Context Design

**Topic:** [[solution-arch/topics/llm-application-architecture]], [[solution-arch/topics/agentic-ai-architecture]]
**Related:** [[solution-arch/concepts/function-calling-and-tool-use]], [[solution-arch/concepts/ai-guardrails-and-safety]], [[ml/concepts/llm-fundamentals]]

## What it is

At the architecture level, "prompt engineering" is the discipline of designing **what enters the model's context window, in what structure, and how it evolves as interaction grows** — not just wordsmithing a single instruction. An AI SA treats the prompt as an architectural artifact with its own versioning, testing, and change-management process, not a string literal buried in application code.

## How it works

```
Anatomy of a production prompt (architecture view, not wording tips):

┌─────────────────────────────────────────────────────────┐
│ SYSTEM PROMPT (static — highest cache value)               │
│  - Role/persona, behavioral constraints, output format      │
│  - Tool definitions (if any) — see function-calling-and-    │
│    tool-use                                                  │
├─────────────────────────────────────────────────────────┤
│ FEW-SHOT EXAMPLES (semi-static)                             │
│  - 1-5 example (input, ideal output) pairs — dramatically   │
│    improves format compliance and edge-case handling vs      │
│    zero-shot instructions alone                              │
├─────────────────────────────────────────────────────────┤
│ RETRIEVED CONTEXT (variable per request)                    │
│  - RAG results, tool outputs, relevant memory — must be      │
│    clearly delimited from instructions (see prompt           │
│    injection defense in ai-guardrails-and-safety)             │
├─────────────────────────────────────────────────────────┤
│ CONVERSATION HISTORY (grows over session)                   │
│  - Managed via sliding window, summarization, or retrieval   │
│    — see context window budget in llm-application-architecture│
├─────────────────────────────────────────────────────────┤
│ CURRENT USER TURN (the actual task/question)                │
└─────────────────────────────────────────────────────────┘

Ordering matters for TWO reasons:
  1. Cache efficiency: static content first maximizes provider-side
     prefix caching hit rate (system prompt + tools should never
     change between calls in the same session)
  2. Recency/attention effects: models tend to weight the most
     recent context more heavily — put the actual instruction/
     question LAST, after supporting context, not buried in the middle
```

### Core prompting techniques (architecture-relevant subset)

```
Zero-shot        → instruction only, no examples. Cheapest, works
                    for well-understood tasks the base model already
                    handles well.

Few-shot         → 1-5 examples included. Use when output FORMAT
                    consistency matters more than raw capability —
                    often more reliable than lengthy instructions.

Chain-of-thought → ask the model to reason step-by-step before the
(CoT)               final answer. Improves multi-step reasoning
                    accuracy; costs more output tokens and latency.
                    Reasoning-tier models do this internally —
                    explicit CoT prompting matters most on
                    general-purpose models.

Structured output → constrain output to a JSON Schema at the
                    sampling level (not just "please return JSON").
                    Architecturally superior to prompted-JSON for
                    any output a program will parse — see
                    [[solution-arch/topics/openai-platform-architecture]].

Role/persona      → sets tone and behavioral boundaries. Also the
                    natural place to encode WHAT THE MODEL MUST
                    REFUSE — a guardrail mechanism, not just style.
```

## Complexity

Not applicable in the algorithmic sense — the relevant "cost" is context window tokens (fixed $ and latency cost per token) and iteration cost (how many prompt versions before an eval-passing version is reached).

## When to use

```
Invest in careful prompt/context architecture when:
  ✅ The prompt is shared infrastructure across many callers
     (changing it has blast radius — treat like a shared library,
     version it, gate changes with evals)
  ✅ Output format compliance is a hard requirement (downstream
     code parses it) → use Structured Outputs, not prompted JSON
  ✅ The task is multi-step / agentic → context design determines
     whether the agent has the right information at each step

Don't over-invest when:
  ❌ A single internal, low-stakes, human-reviewed use case — a
     simple, well-tested prompt is enough; formal eval infrastructure
     is overkill for a one-off internal tool
```

## Common interview angles

```
Q: "Your prompt works great in testing but degrades in production
    — why?"
A: Usually one of: (1) context window creeping over time as
   conversation history grows unmanaged, pushing earlier
   instructions out or diluting attention; (2) retrieved context in
   production is noisier/lower-quality than curated test examples;
   (3) a silent model version upgrade behind the provider's API
   changed behavior. Diagnose with production tracing (see
   [[solution-arch/concepts/llm-observability-and-evals]]), not by
   re-testing the same prompt in isolation.

Q: "How do you prevent a prompt from growing unmaintainably as you
    add more instructions/edge cases over time?"
A: Same discipline as code: modularize (separate system prompt
   sections), version control the prompt with the eval suite that
   validates it, and periodically refactor/prune rather than only
   ever appending "and also handle this edge case" indefinitely —
   accumulated ad-hoc instructions conflict with each other and
   confuse the model.

Q: "How do you keep untrusted retrieved content from being
    interpreted as instructions?"
A: Explicit delimiters (e.g. XML-like tags) around untrusted
   content, explicit system-prompt instruction that content within
   those delimiters is DATA to reason about, never a command to
   follow — foundational prompt-injection defense, detailed in
   [[solution-arch/concepts/ai-guardrails-and-safety]].
```

## Examples

```
Customer support agent system prompt structure:
  [Role + tone + refusal boundaries]
  [Available tools + when to use each]
  [2-3 few-shot examples of ideal escalation behavior]
  [Retrieved KB articles relevant to this query, delimited]
  [Conversation history, summarized beyond last 10 turns]
  [Current user message]
```

## Sources
- [[ml/concepts/llm-fundamentals]]
- [[solution-arch/sources/building-effective-agents-anthropic]]
