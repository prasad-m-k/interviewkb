# Transformers

**Topic:** [[ml/topics/deep-learning]], [[ml/topics/nlp]]
**Related:** [[ml/concepts/attention-mechanism]], [[ml/concepts/embeddings]], [[ml/concepts/batch-normalization]]

## What it is
A neural network architecture based entirely on attention mechanisms, without convolution or recurrence. Introduced in "Attention Is All You Need" (Vaswani et al., 2017). Foundation of all modern LLMs.

## Architecture

### Encoder Block (used in BERT, RoBERTa)
```
Input tokens
  → Token embedding + Positional encoding
  → [Multi-Head Self-Attention → Add & Norm
     → Feed-Forward Network → Add & Norm] × N layers
  → Output representations for each token
```

**Add & Norm**: residual connection (x + sublayer(x)) then LayerNorm. Residual connection ensures gradient flow; LayerNorm stabilizes training.

### Decoder Block (used in GPT)
Same as encoder, but with a **causal mask** in self-attention (each token only attends to past tokens). Used autoregressively — generate one token at a time.

### Encoder-Decoder (used in T5, BART)
```
Encoder reads full input (bidirectional)
Decoder generates output token by token:
  - Causal self-attention (attend to previously generated tokens)
  - Cross-attention (attend to encoder output)
  - FFN
```

## The Feed-Forward Network (FFN)
```
FFN(x) = max(0, xW₁ + b₁)W₂ + b₂
```
- Two linear layers with ReLU (or GELU in modern models) in between
- Dimension expands 4× in the middle: d_model=768 → d_ff=3072
- Operates **independently per position** (unlike attention which mixes positions)
- Modern view: FFN layers act as "memory" — they store factual knowledge

## Pre-training Objectives

| Objective | Models | Mechanism |
|---|---|---|
| **Masked Language Model (MLM)** | BERT | Randomly mask 15% of tokens; predict masked tokens |
| **Next Token Prediction (CLM)** | GPT | Predict next token from prefix; autoregressive |
| **Span corruption** | T5 | Replace contiguous spans with sentinel; reconstruct spans |
| **Replaced Token Detection** | ELECTRA | Generator corrupts tokens; discriminator detects replaced ones; sample-efficient |

## Model Families and Scale

| Model | Type | Params | Context | Key use |
|---|---|---|---|---|
| BERT-base | Encoder | 110M | 512 | NLU tasks |
| BERT-large | Encoder | 340M | 512 | NLU tasks |
| GPT-2 | Decoder | 117M-1.5B | 1024 | Generation |
| GPT-3 | Decoder | 175B | 2048 | Few-shot learning |
| T5-base | Enc-Dec | 220M | 512 | Seq2seq |
| LLaMA 2 7B | Decoder | 7B | 4096 | Open-weight LLM |
| LLaMA 2 70B | Decoder | 70B | 4096 | Open-weight LLM |

## Key Architectural Choices in Modern LLMs

| Choice | Old (original Transformer) | Modern (LLaMA, Mistral) |
|---|---|---|
| **Normalization** | Post-LN (after residual) | Pre-LN (before sublayer) — more stable |
| **Normalization type** | LayerNorm | RMSNorm (simpler, faster) |
| **Positional encoding** | Sinusoidal or learned | RoPE (Rotary Position Embedding) |
| **Activation** | ReLU | SwiGLU or GELU |
| **Attention** | Standard | GQA (Grouped Query Attention) — fewer KV heads, efficient inference |

## Fine-tuning Strategies
See [[ml/topics/nlp]] for full breakdown.
- Full fine-tune: best performance, high cost
- LoRA: low-rank weight updates; typically rank=8-16; ~10× fewer trainable params
- Prompt tuning: train only a soft prefix; frozen model
- RLHF (Reinforcement Learning from Human Feedback): align generation with human preferences; used in ChatGPT, Claude

## Scaling Laws
- Loss ∝ N^(-0.076) where N is parameters (Chinchilla: compute-optimal training means more data per parameter than GPT-3)
- Chinchilla rule: for compute budget C, optimal N ∝ C^0.5, tokens ∝ C^0.5 (train smaller model on more tokens)
- Emergent abilities: some capabilities (multi-step reasoning, few-shot learning) appear suddenly at scale thresholds

## Common interview angles
- BERT vs. GPT — what determines the choice? (BERT: understanding/classification (bidirectional); GPT: generation (causal))
- What is LayerNorm and why is it preferred over BatchNorm in Transformers? (BatchNorm normalizes across batch; unstable for variable-length sequences; LayerNorm normalizes across the feature dimension, works per-token)
- What is the FLOPs cost of a forward pass? (≈ 2ND for N parameters, D tokens — used for compute budgeting)
- What is KV cache and why does it matter for inference? (during autoregressive generation, cache Keys/Values from previous tokens to avoid recomputing; reduces inference from O(n²) per token to O(n) amortized)
- Why use Pre-LN over Post-LN? (Pre-LN has more stable gradients early in training; Post-LN can diverge for very deep models without careful LR warmup)

## Sources
- [[ML overview]]
