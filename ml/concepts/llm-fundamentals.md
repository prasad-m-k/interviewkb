# LLM Fundamentals

**Topic:** [[ml/topics/deep-learning]], [[ml/topics/nlp]]
**Related:** [[ml/concepts/transformers]], [[ml/concepts/attention-mechanism]], [[ml/concepts/reinforcement-learning]], [[ml/patterns/rlhf]], [[ml/patterns/rag-pattern]]

---

## What it is

Large Language Models (LLMs) are transformer-based models trained on massive text corpora to predict the next token. The same architecture generalizes to understanding, generation, summarization, reasoning, and code — the capability emerges from scale.

---

## Pretraining

**Objective:** next-token prediction (causal language modeling). For each token position, predict the next token from the vocabulary.

**Loss:** cross-entropy over the vocabulary at each position:
```
L = -sum_t log P(x_t | x_1, ..., x_{t-1})
```

**Data:** web crawls (Common Crawl), books, code, scientific papers. Deduplication and quality filtering are critical — garbage in, garbage out at 10^12 tokens.

**Scale:** governed by **Chinchilla scaling laws** (Hoffmann et al., 2022):
- Optimal tokens ≈ 20 × model parameters
- Doubling model size → double training tokens for equal compute efficiency
- Implication: smaller models trained on more tokens often outperform larger undertrained models

**Infrastructure:** see [[ml/concepts/distributed-training]].

---

## Fine-Tuning Approaches

### Supervised Fine-Tuning (SFT)
Train on (instruction, response) pairs. Teaches the model to follow instructions rather than just predict next token. Used in every modern chat model.

### Parameter-Efficient Fine-Tuning (PEFT)
- **LoRA (Low-Rank Adaptation):** Freeze pretrained weights; inject trainable low-rank matrices into attention layers. Drastically reduces trainable parameters (100x-1000x) with minimal quality loss.
- **Prefix Tuning:** Prepend trainable vectors to every layer's key/value. No weight updates to the base model.
- **When to use:** Few-shot data, limited compute budget, need to serve multiple fine-tuned models from one base.

### Full Fine-Tuning
Update all weights. Best quality when data is large. Expensive. Risk of catastrophic forgetting.

---

## RLHF (Reinforcement Learning from Human Feedback)

The standard approach for aligning LLMs to human preferences. See [[ml/patterns/rlhf]] for the full pattern.

1. **Reward Model (RM):** Train a scalar reward model on human preference pairs (A preferred over B).
2. **RL with PPO:** Optimize the LLM's policy to maximize the reward model's score while staying close to the SFT model (KL penalty prevents reward hacking).

**Direct Preference Optimization (DPO)** is a newer, simpler alternative: reformulates RLHF as a classification problem over preference pairs, eliminating the separate RM training step.

---

## Inference Optimization

Inference is the bottleneck in production. Google-scale LLM serving requires specific techniques.

### KV Cache
During autoregressive generation, each new token needs attention over all previous tokens. The KV cache stores pre-computed Key and Value matrices for past tokens. Without it, inference cost is O(n²) per generation; with it, O(n) per token.

**Memory cost:** `2 × num_layers × seq_len × d_model × bytes_per_param`. For Llama-3-70B with 8K context: ~35GB for the KV cache alone.

### Batching Strategies
- **Static batching:** all requests in a batch must complete together. Longest request dictates latency for all.
- **Continuous batching (iteration-level scheduling):** completed requests leave the batch immediately; new requests join mid-generation. Standard in production systems (vLLM, TGI).

### Quantization
Reduce model weight precision to shrink memory and increase throughput:
- **FP16 / BF16:** standard for training and inference
- **INT8:** ~2x memory reduction, minimal quality loss for inference
- **INT4 (GPTQ, AWQ):** ~4x reduction, some quality degradation; fine for many tasks
- **Speculative decoding:** use a small draft model to generate token candidates; large model verifies in parallel. 2-3x throughput improvement for long generation.

### Latency vs. Throughput
- **Time to first token (TTFT):** dominated by the prefill phase (processing the input prompt)
- **Tokens per second (TPS):** dominated by memory bandwidth, not compute
- **Optimization levers:** larger batch sizes increase throughput; more GPU memory reduces swapping; tensor parallelism splits model across GPUs for large models

---

## Context Window and Positional Encoding

- **Absolute positional encoding (original Transformer):** not generalizable beyond training length
- **Rotary Position Embedding (RoPE):** encodes relative position via rotation in embedding space; extrapolates better to longer sequences. Used in LLaMA, GPT-NeoX, Gemma
- **ALiBi:** penalizes attention scores based on distance; also generalizes to longer contexts

Long-context models (128K+ tokens) require efficient attention variants:
- **Flash Attention:** reorders attention computation to minimize HBM reads/writes; same result, 3-5x faster
- **Sliding Window Attention (Mistral):** local attention with a fixed window; global tokens via rolling buffer

---

## Evaluation

| Metric | What it measures | Notes |
|---|---|---|
| Perplexity | How well the model predicts held-out text | Lower is better; not correlated with downstream task quality |
| MMLU | Broad knowledge across 57 domains | Favors factual recall |
| HumanEval | Code generation pass@k | Favors instruction-following and code accuracy |
| MT-Bench | Multi-turn conversation quality | GPT-4 judged; gameable |
| LMSYS Chatbot Arena | Human preference via blind comparison | Best signal for real-world quality |
| TruthfulQA | Avoids common misconceptions | Important for safety evaluation |

**Key point for interviews:** perplexity is a training-time metric, not a deployment metric. Always discuss downstream task evaluation.

---

## Common Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Hallucination | Token prediction without grounding | RAG [[ml/patterns/rag-pattern]], citation training |
| Repetition | Degenerate decoding | Repetition penalty, nucleus sampling |
| Prompt injection | Malicious instructions override system prompt | Input sanitization, system prompt hardening |
| Context length overflow | Input exceeds window | Chunking, summarization, retrieval |
| Reward hacking | RM exploited by RL policy | KL penalty, RM ensemble, human oversight |
| Sycophancy | Model agrees with user even when wrong | Adversarial training examples, diverse RLHF data |

---

## When to use LLMs vs. Smaller Models

| Situation | Recommendation |
|---|---|
| Low latency requirement (<50ms) | Distilled model, logistic regression, GBM |
| Tabular structured data | GBM (XGBoost/LightGBM) — LLMs rarely beat GBMs on tabular |
| Open-ended text generation | LLM |
| Complex reasoning over long context | Large LLM with sufficient context window |
| Specific domain with fine-tuning data | Fine-tuned smaller model often beats large general model |
| Edge / on-device | Quantized small model (Gemma-2B, Phi-3-mini) |

---

## Common Interview Angles

1. "What is the difference between pretraining and fine-tuning? When would you use each?"
2. "Explain how the KV cache works and why it matters for production serving."
3. "What is LoRA and why is it preferred over full fine-tuning in many scenarios?"
4. "How would you detect and reduce hallucination in a deployed LLM?"
5. "What are the tradeoffs between RAG and fine-tuning for domain adaptation?"
6. "Walk me through RLHF. What can go wrong?"
7. "How do scaling laws inform training budget decisions?" (Research Engineer track)

---

## Sources
- [[ml/concepts/transformers]]
- [[ml/concepts/attention-mechanism]]
- [[ml/patterns/rlhf]]
- [[ml/patterns/rag-pattern]]
- [[ml/concepts/distributed-training]]
