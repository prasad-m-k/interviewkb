# Transformers and Attention — A Primer for Agent Builders

**Topic:** [[agents/topics/agent-fundamentals]]
**Related:** [[agents/foundations/neural-networks]], [[agents/foundations/embeddings-and-similarity]], [[agents/concepts/context-engineering]], [[ml/concepts/attention-mechanism]], [[ml/concepts/transformers]]

---

## Why this page exists

"Context window," "lost in the middle," "attention," "tokens" — these words show up constantly in agent design ([[agents/concepts/context-engineering]], [[agents/concepts/agent-loop]]) and are much easier to reason about once you've seen, even at a conceptual level, what a transformer actually does with the text you send it. This page gives the shape of the mechanism, not the math. For the full mathematical treatment (multi-head attention, positional encoding schemes, complexity analysis), see [[ml/concepts/attention-mechanism]] and [[ml/concepts/transformers]].

---

## The pipeline, end to end

```
                          Tokens
                            │
                            ▼
          Token Embedding + Positional Encoding
                            │
                            ▼
           ┌─────────────────────────────────┐
           │      TRANSFORMER BLOCK (×N)     │
           │                                 │
           │    ┌──────────────────────┐     │
           │    │ Self-Attention       │     │
           │    └───────────┬──────────┘     │
           │                ▼                │
           │    ┌───────────────────────┐    │
           │    │ Feed-Forward Network  │    │
           │    └───────────────────────┘    │
           └────────────────┬────────────────┘
                            │
                            ▼
        Probability distribution over next token
```

A modern LLM stacks dozens of these transformer blocks. Every block does the same two things in sequence: **self-attention** (let each token gather relevant information from every other token) and a **feed-forward network** (the plain neural-network layer from [[agents/foundations/neural-networks]], applied to each token independently). The final output is a probability distribution over every possible next token — the model then samples one, appends it, and repeats the whole pipeline to generate the next token.

---

## Tokens: the actual unit of "text" the model sees

Before any of this happens, text is split into **tokens** — not always whole words. "Understanding" might become `Under` + `stand` + `ing`; common words are often a single token. This is why LLM pricing, context limits, and "context window" are all measured in tokens, not words or characters — see [[agents/foundations/nlp-fundamentals]] for the tokenization mechanics, and [[agents/concepts/context-engineering]] for why this matters when managing an agent's context budget.

---

## Self-attention: the key idea

For each token, self-attention computes how relevant every *other* token in the sequence is to understanding this one, then blends information from all of them weighted by that relevance. The classic illustrating example: resolving what a pronoun refers to.

```
Sentence: "The animal didn't cross the street because it was too tired"

Self-attention when predicting/resolving "it":

  animal              0.71  █████████████████████
  tired               0.15  ████
  street              0.06  ██
  (all other tokens)  0.08  ██
```

The model has learned, from training data, that "it" in this sentence structure is far more likely to refer to "animal" than "street" — so "it"'s representation gets updated to draw heavily on "animal"'s. This happens for *every* token, with respect to *every other* token, at *every* layer — which is also why attention has a computational cost that grows with the square of sequence length, and why very long contexts are expensive (see [[ml/concepts/attention-mechanism]] for the complexity details and mitigations like sliding-window attention).

---

## What this explains about agent behavior

| Attention/transformer fact | Consequence for agents |
|---|---|
| Attention cost grows quadratically with sequence length | Long agent traces get expensive and slow — not just in token count, but in the underlying compute. This is the deeper reason [[agents/concepts/context-engineering]] treats context as an actively managed budget, not just "more is fine." |
| Not all tokens receive equal attention — models can attend less reliably to information buried in the middle of a long context | This is the mechanistic root of the "lost in the middle" failure mode discussed in [[agents/concepts/context-engineering]] — it's not a vague notion, it's a property of how attention weights get distributed over a long sequence. |
| Every generated token is fed back in as input for generating the next one | This is exactly why chain-of-thought and ReAct work at all — see [[agents/concepts/reasoning-and-planning]]: intermediate reasoning tokens genuinely become part of what the next attention pass conditions on. |
| The feed-forward layers (not attention) are where a large fraction of the model's "knowledge" is thought to live | This reinforces why that knowledge is frozen at training time and why RAG ([[agents/concepts/agentic-rag]]) has to supply anything that changed since training, rather than the model "just knowing." |

---

## Anticipated Questions

1. "Is attention the same thing as the model's context window?" — Related but distinct. The context window is *how many tokens* the model can process at once; attention is *the mechanism* that processes them, letting every token look at every other token within that window. A bigger context window means attention has to operate over more tokens, which is why it costs more (quadratically), not just proportionally more.
2. "Why do transformers need positional encoding at all?" — Attention on its own has no inherent sense of order — it just computes relevance between pairs of tokens regardless of where they sit in the sequence. Positional encoding injects "this token is at position 5" information so word order (which changes meaning) isn't lost. See [[ml/concepts/llm-fundamentals]] for RoPE and other modern schemes.
3. "Does the model 'decide' to pay more attention to important context, like a human skimming?" — Not deliberately in the way a human decides to focus — attention weights are a learned mathematical function of the content itself, not a conscious prioritization step. That's exactly why prompt structure and explicit emphasis (repeating key instructions, placing them prominently) measurably help: you're working with a fixed mechanism, not petitioning a reader.
4. "If I understand attention, do I understand why LLMs hallucinate?" — Partially. Attention explains how the model blends information from context; it doesn't have a built-in truth-checking step at any stage. Combined with the fact that training only rewards plausible-sounding continuations (see [[agents/foundations/neural-networks]]), there's no mechanism forcing factual grounding — which is precisely the gap [[agents/concepts/agentic-rag]] and [[agents/concepts/guardrails-and-safety]] are designed to close.

---

## Sources
- [[ml/concepts/attention-mechanism]]
- [[ml/concepts/transformers]]
- [[ml/concepts/llm-fundamentals]]
- [[agents/foundations/neural-networks]]
- [[agents/foundations/nlp-fundamentals]]
