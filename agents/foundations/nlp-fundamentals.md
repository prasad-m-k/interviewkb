# NLP Fundamentals — A Primer for Agent Builders

**Topic:** [[agents/topics/agent-fundamentals]], [[agents/topics/memory-and-context]]
**Related:** [[agents/foundations/transformers-and-attention]], [[agents/concepts/context-engineering]], [[ml/topics/nlp]]

---

## Why this page exists

"Tokens," "context window," "128K context" — every cost and capacity conversation about agents ([[agents/concepts/context-engineering]], [[agents/concepts/agent-loop]]) is denominated in a unit that isn't words or characters. This page explains that unit and the handful of other NLP basics an agent builder actually leans on day to day. For the full NLP survey (architectures, fine-tuning strategies, task-specific approaches), see [[ml/topics/nlp]].

---

## Tokenization: the actual unit of text an LLM processes

Before a model sees any text, it's split into **tokens** using a fixed vocabulary — and tokens are frequently sub-word pieces, not whole words.

```
Text:   "Understanding agent memory"
Tokens: [Under][stand][ing][ agent][ memory]
Count:  5 tokens for 3 words — subword tokenization splits uncommon words
```

Common words are often a single token; rarer or longer words get split into pieces the model has seen often enough during training to represent well. This is done with an algorithm like **BPE (Byte-Pair Encoding)** or **WordPiece**, which builds the vocabulary by iteratively merging the most frequently co-occurring character/subword pairs across a huge training corpus (see [[ml/topics/nlp]] for the algorithmic detail).

**Why every agent builder needs to know this:**
- **Cost and limits are token-based, not word-based** — a "128K context window" means 128,000 tokens, which is roughly 90,000–100,000 English words, but varies by language and content (code and non-English text often tokenize less efficiently, using *more* tokens per word).
- **Context budget math in [[agents/concepts/context-engineering]] is entirely in tokens** — every tool result, every turn of history, every retrieved document chunk consumes part of the same fixed token budget.
- **Truncation and chunking operate on token boundaries**, not word or sentence boundaries by default — this is why chunking strategies in [[ml/concepts/rag]] explicitly measure chunk size in tokens.

---

## The language modeling objective (why the model generates text at all)

At the core of every LLM is one training objective: **predict the next token**, given everything before it. See [[agents/foundations/neural-networks]] for how the model itself is structured, and [[ml/concepts/llm-fundamentals]] for the full pretraining picture (data, scale, scaling laws).

```
Given: "The capital of France is"
Predict the next token:  "Paris" (high probability)
                          "London" (very low probability)
                          "the" (low probability)
```

Every capability that looks like "understanding" — answering questions, following instructions, writing code — emerges from this single repeated mechanism applied at massive scale, not from a separate module for each skill. This is why prompting, fine-tuning, and RAG all work by *shaping what comes before* the prediction (the context) or *reshaping the prediction function itself* (the weights) — there is no other lever.

---

## Sequence-to-sequence framing, briefly

Some NLP tasks (translation, summarization) are framed as: given an input sequence, produce an output sequence. Modern LLM-based agents mostly don't need this distinction explicitly — a single decoder-only model (the architecture behind GPT-style LLMs, see [[ml/topics/nlp]] for the encoder/decoder-only comparison) handles input and output as one continuous token stream: the prompt is simply the beginning of the sequence, and the response is its continuation.

---

## What this explains about agent behavior

| NLP fact | Consequence for agents |
|---|---|
| Tokenization is sub-word, not sub-word-boundary-aware for humans | Character-counting or word-counting to estimate cost/context usage is unreliable — always measure in actual tokens (most SDKs expose a tokenizer/token-counting utility) |
| Different content types tokenize at different densities (code, non-English text often use more tokens per "unit of meaning") | An agent that reasons heavily in code or non-English languages will hit context/cost limits sooner than raw word count would suggest |
| The model has no separate "memory module" — it only ever predicts the next token from what's in context | This is *why* [[agents/concepts/memory-architectures]] and [[agents/concepts/context-engineering]] have to explicitly re-inject anything the model needs to "remember" — there is no other channel |

---

## Anticipated Questions

1. "Is a token roughly a word?" — Roughly, for common English text — about 0.75 words per token on average — but this breaks down badly for code, non-English languages, rare words, and numbers, all of which tend to tokenize into more, smaller pieces. Always check actual token counts for cost/context-critical work, don't estimate from word count.
2. "Why does the same sentence sometimes cost different numbers of tokens in different languages?" — Tokenizer vocabularies are built from training data that's disproportionately English, so common English words often get a single token while equivalent words in other languages get split into more pieces — a real, practical cost difference for multilingual agents.
3. "If the model is 'just' predicting the next token, why does it seem to plan ahead?" — Apparent planning is consistent with pure next-token prediction because the model has learned, from vast training data, statistical patterns that correspond to coherent multi-step structure — and techniques like chain-of-thought ([[agents/concepts/reasoning-and-planning]]) make this explicit by having the model generate its "plan" as tokens it then conditions on.
4. "Does chunking for RAG need to respect sentence or paragraph boundaries?" — It doesn't *have* to (fixed-size token chunking is simplest and common), but chunks that cut mid-sentence or mid-idea tend to retrieve worse and read worse once inserted into a prompt — see the chunking strategy tradeoffs in [[ml/concepts/rag]].

---

## Sources
- [[ml/topics/nlp]]
- [[ml/concepts/llm-fundamentals]]
- [[ml/concepts/rag]]
- [[agents/foundations/transformers-and-attention]]
- [[agents/foundations/neural-networks]]
