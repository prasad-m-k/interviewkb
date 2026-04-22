# Attention Mechanism

**Topic:** [[ml/topics/deep-learning]], [[ml/topics/nlp]]
**Related:** [[ml/concepts/transformers]], [[ml/concepts/embeddings]]

## What it is
A mechanism that allows a model to dynamically weight the importance of different input positions when computing each output representation. Enables capturing long-range dependencies without the sequential bottleneck of RNNs.

## Scaled Dot-Product Attention

```
Attention(Q, K, V) = softmax(QKᵀ / √dₖ) · V
```

- **Q (Query)**: what information am I looking for?
- **K (Key)**: what information does each position contain?
- **V (Value)**: what information does each position provide if selected?

**Intuition**: QKᵀ computes similarity scores between all query-key pairs. After softmax, these are weights. Apply weights to values → weighted sum = attended output.

**Why √dₖ scaling?**: For large dₖ, dot products grow in magnitude → softmax saturates → gradients vanish. Dividing by √dₖ keeps variance at 1.

## Multi-Head Attention

```
MultiHead(Q, K, V) = Concat(head₁, ..., headₕ) · Wₒ
where headᵢ = Attention(Q·Wᵢᴼ, K·Wᵢᴷ, V·Wᵢᵛ)
```

- Run h attention heads in parallel, each with different learned projections
- Each head can attend to different aspects: syntax, coreference, semantics, etc.
- Concatenate outputs, project down → same dimension as input

Typical: 8-16 heads for base models; 32-64 for large models.

## Self-Attention vs. Cross-Attention

| Type | Q from | K, V from | Used in |
|---|---|---|---|
| **Self-attention** | Same sequence | Same sequence | Encoder (BERT), Decoder (GPT) |
| **Cross-attention** | Decoder state | Encoder output | Seq2seq decoder (T5, BART) |

Self-attention: each token attends to every other token in the same sequence → capture long-range dependencies.

## Attention Masks

### Padding Mask
Set attention scores to -∞ for padding tokens → softmax → 0 weight for padding. Prevents padding from influencing representations.

### Causal (Future) Mask
In autoregressive generation (GPT), mask future positions:
```
[1, 0, 0, 0]    token 1 can only see itself
[1, 1, 0, 0]    token 2 sees tokens 1-2
[1, 1, 1, 0]    token 3 sees tokens 1-3
[1, 1, 1, 1]    token 4 sees all tokens
```
Implemented by adding a lower-triangular matrix of -∞ before softmax.

## Positional Encoding
Self-attention is permutation-invariant (no inherent order). Positional encodings add position information:

**Sinusoidal (original Transformer):**
```
PE(pos, 2i)   = sin(pos / 10000^(2i/dmodel))
PE(pos, 2i+1) = cos(pos / 10000^(2i/dmodel))
```
- Fixed; no parameters; generalizes to longer sequences than seen in training

**Learned positional embeddings (BERT, GPT):**
- Learnable embedding for each position index
- More flexible; but doesn't generalize beyond max training length

**Relative position (RoPE, ALiBi):**
- Encode relative distance between tokens rather than absolute position
- Better length generalization; used in LLaMA, Falcon, modern LLMs

## Complexity: The O(n²) Problem
Self-attention computes pairwise similarities between all n tokens: O(n²) time and memory. For n=512, this is manageable. For n=16K (long documents), this is expensive.

Solutions:
- **Sparse attention** (BigBird, Longformer): attend to local window + global tokens
- **Linear attention**: kernel approximation to reduce O(n²) to O(n)
- **FlashAttention**: exact but I/O-efficient; tiles QKV to reduce HBM reads/writes; 2-4× faster in practice

## Common interview angles
- Why is scaling by √dₖ necessary? (prevents softmax saturation for large dₖ)
- What does each attention head learn? (different heads capture different linguistic phenomena; one might capture coreference, another subject-verb agreement)
- What is the time/space complexity of self-attention? (O(n²·d) time; O(n²) space for the attention matrix)
- Why can Transformers capture longer dependencies than LSTMs? (every token directly attends to every other token; no information bottleneck through sequential state)
- What is FlashAttention and why is it important? (I/O-aware exact attention; same result but much less GPU memory bandwidth usage; enables longer sequences and larger batches)

## Sources
- [[ML overview]]
