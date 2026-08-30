# Neural Networks — A Primer for Agent Builders

**Topic:** [[agents/topics/agent-fundamentals]]
**Related:** [[agents/foundations/transformers-and-attention]], [[agents/foundations/embeddings-and-similarity]], [[ml/concepts/gradient-descent]], [[ml/concepts/backpropagation]], [[ml/concepts/activation-functions]]

---

## Why this page exists

Every "Think" step in the agent loop ([[agents/concepts/agent-loop]]) is, mechanically, a forward pass through a very large neural network. Students walking into agent design without ML background tend to treat the LLM as an opaque oracle — which is fine for using it, but makes several agent-design realities (why context length has a real cost, why hallucination happens, why fine-tuning vs. prompting are different tools) feel arbitrary instead of consequences of how the underlying model actually works. This page gives just enough of the mechanism to make those later concepts click. For the full depth (backprop math, optimizers, regularization), see [[ml/concepts/gradient-descent]], [[ml/concepts/backpropagation]], and [[ml/concepts/activation-functions]] in the ML knowledge base.

---

## What it is

A neural network is a function made of stacked layers of simple units ("neurons"), each of which computes a weighted sum of its inputs, adds a bias, and passes the result through a non-linear **activation function**. Stack enough of these layers and the network can approximate extremely complex functions — including "predict the next word" at the scale of an LLM.

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ INPUT LAYER   │───►│ HIDDEN LAYER  │───►│ OUTPUT LAYER  │
│ (3 units)     │    │ (4 units)     │    │ (1 unit)      │
└───────────────┘    └───────────────┘    └───────────────┘
```

Every unit in a layer is connected to every unit in the next layer ("fully connected" / "dense"). Modern LLMs replace plain dense layers with more specialized structures (see [[agents/foundations/transformers-and-attention]]), but the core building block — weighted sum → activation — is the same everywhere.

---

## What a single neuron computes

```
x1 ──w1──┐
x2 ──w2──┼──► z = Σ(wi·xi) + b ──► activation f(z) ──► output
x3 ──w3──┘
```

- `x1, x2, x3` — inputs (could be raw features, or the outputs of the previous layer)
- `w1, w2, w3` — learned weights: how much each input matters
- `b` — a learned bias: an offset independent of the inputs
- `f` — a non-linear activation function (ReLU, GELU, etc. — see [[ml/concepts/activation-functions]])

**Why the non-linearity matters:** without it, stacking layers would collapse mathematically into a single linear function no matter how many layers you add — the network could only ever learn straight-line relationships. The activation function is what lets depth actually buy the network more expressive power.

---

## How it learns

A network starts with random weights and produces garbage. Training is the process of nudging every weight and bias so the network's output gets closer to the correct answer, repeated across millions or billions of examples.

```
1. Forward pass:   input → prediction
2. Compute loss:   how wrong was the prediction? (see [[ml/concepts/loss-functions]])
3. Backward pass:  compute how much each weight contributed to that error
                    (backpropagation — see [[ml/concepts/backpropagation]])
4. Update weights: nudge each weight slightly to reduce the error
                    (gradient descent — see [[ml/concepts/gradient-descent]])
5. Repeat, millions of times, over a large dataset
```

For an LLM specifically, the "correct answer" at each training step is simply *the actual next token in the training text* — the model is trained to predict what comes next, over and over, across a huge corpus. Nothing more exotic than that is happening at training time; the sophistication is entirely in scale and architecture (see [[agents/foundations/transformers-and-attention]]) rather than in the training objective itself. Full depth on this: [[ml/concepts/llm-fundamentals]].

---

## Why this matters for agent design

| Neural-network fact | Consequence for agent builders |
|---|---|
| The model only ever produces a probability distribution over the next token | "Hallucination" isn't a bug being triggered — it's the model doing exactly what it was trained to do (predict a plausible next token) without any built-in fact-checking step. This is *why* grounding mechanisms like RAG ([[agents/concepts/agentic-rag]]) and guardrails ([[agents/concepts/guardrails-and-safety]]) have to be engineered on top, not assumed. |
| Everything the model "knows" is encoded in its weights, fixed at training time | This is exactly the staleness problem RAG solves — see [[ml/concepts/rag]] and [[agents/concepts/agentic-rag]] for why retrieval exists at all. |
| A forward pass has a real, scaling compute cost per token | This is why context length isn't free — see [[agents/concepts/context-engineering]] and [[ml/concepts/llm-fundamentals]] (KV cache, inference cost). |
| The network's behavior can be changed either by changing its weights (fine-tuning) or by changing its input (prompting/context) | This is the fundamental reason RAG and fine-tuning are different tools solving different problems — see the comparison table in [[ml/concepts/rag]]. |

---

## Neural networks vs. the LLM inside your agent

An LLM is a neural network, but a specific, very large kind: a **transformer**, trained on the specific objective of next-token prediction over enormous text corpora. Not every neural-network fact applies with equal force at LLM scale (a lot of interesting behavior — in-context learning, emergent reasoning — shows up only once models get big enough), but the base mechanism (weighted sums, non-linear activations, gradient-based learning) is unchanged. The architecture that made this scale practical for language is covered next: [[agents/foundations/transformers-and-attention]].

---

## Anticipated Questions

1. "Do I need to understand backpropagation to build agents?" — No. You need to understand *what the model is* (a function trained to predict plausible continuations) well enough to reason about its failure modes. The math is only necessary if you're training or fine-tuning models yourself — see [[ml/concepts/backpropagation]] if you want that depth.
2. "Is a bigger neural network always a better one?" — Not automatically — see the Chinchilla scaling-law discussion in [[ml/concepts/llm-fundamentals]]: model size and training data size both have to scale together for a model to be compute-optimal. A huge undertrained model can underperform a smaller, properly-trained one.
3. "Why can't we just tell the network the right answer once and have it remember?" — Weight updates require many training examples and a full training run; you can't durably teach a deployed model a new fact by "telling it" mid-conversation. That's exactly the gap RAG and agent memory ([[agents/concepts/memory-architectures]]) exist to fill without retraining.
4. "If it's just predicting the next token, how can it reason step by step (ReAct, chain-of-thought)?" — Multi-step reasoning emerges from the same next-token-prediction mechanism applied repeatedly: each generated token becomes part of the input for predicting the next one, so intermediate reasoning tokens genuinely influence the final answer. See [[agents/concepts/reasoning-and-planning]].

---

## Sources
- [[ml/concepts/gradient-descent]]
- [[ml/concepts/backpropagation]]
- [[ml/concepts/activation-functions]]
- [[ml/concepts/llm-fundamentals]]
- [[agents/foundations/transformers-and-attention]]
