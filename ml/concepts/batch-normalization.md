# Batch Normalization (and Friends)

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/gradient-descent]], [[ml/concepts/activation-functions]], [[ml/concepts/regularization]]

## What it is
Normalization techniques that stabilize the distribution of layer inputs during training, enabling higher learning rates, faster convergence, and reduced sensitivity to initialization.

## Batch Normalization (BatchNorm)

```
# For a mini-batch B of activations x:
μ_B = (1/m) Σ xᵢ                    # batch mean
σ²_B = (1/m) Σ (xᵢ - μ_B)²          # batch variance
x̂ᵢ = (xᵢ - μ_B) / √(σ²_B + ε)      # normalize
yᵢ = γ · x̂ᵢ + β                     # scale and shift (learned)
```

γ (scale) and β (shift) are learnable parameters — allows the network to undo normalization if needed.

**During inference**: use running mean/variance tracked during training (not batch statistics).

**Why it works:**
- Reduces internal covariate shift (distribution of inputs to each layer stays stable)
- Acts as regularization: each example's normalization depends on the batch → noise → implicit regularization
- Allows larger learning rates

**Placement:** typically between linear/conv and activation:
`Conv → BN → ReLU` (not `Conv → ReLU → BN`)

## Layer Normalization (LayerNorm)

```
# Normalize over the feature dimension, not the batch dimension
μ = (1/H) Σ xᵢ        # mean over H features
σ² = (1/H) Σ (xᵢ-μ)²
x̂ = (x - μ) / √(σ² + ε)
y = γ · x̂ + β
```

- Independent of batch size → works with batch_size=1
- **Preferred for Transformers and RNNs** (variable-length sequences, small batches)
- Each token's features are normalized independently

## Comparison: BatchNorm vs. LayerNorm

| | BatchNorm | LayerNorm |
|---|---|---|
| **Normalizes over** | Batch dimension | Feature dimension |
| **Requires large batch** | Yes (unstable with small batches) | No |
| **Training ≠ inference** | Yes (different stats) | No (same computation) |
| **Best for** | CNNs, MLPs | Transformers, RNNs |
| **Handles variable length** | No | Yes |

## Instance Normalization
- Normalize over H × W (spatial) for each sample and channel independently
- Used in style transfer: preserves style info in channel statistics while normalizing content

## Group Normalization
- Divide channels into groups; normalize within each group
- Works well for small batches (detection, segmentation); between BN and LN

## RMSNorm
```
y = x / RMS(x) · γ,   where RMS(x) = √(mean(x²))
```
- Simplified LayerNorm: drop mean subtraction step
- Faster; empirically close to LayerNorm
- Used in LLaMA, T5, and most modern LLMs

## Training vs. Inference with BatchNorm
```python
model.train()   # BatchNorm uses batch statistics; dropout is active
model.eval()    # BatchNorm uses running statistics; dropout disabled
```
**Common bug**: forgetting `model.eval()` at inference → batch statistics from test batch contaminate results.

## Common interview angles
- Why can't you use BatchNorm with batch size 1? (can't compute stable mean/variance from a single example; variance = 0 → numerically unstable)
- Why do Transformers use LayerNorm instead of BatchNorm? (variable-length sequences; small effective batch sizes per token; LN doesn't depend on other samples)
- What are γ and β in BatchNorm and why do we need them? (learnable scale and shift; allow the model to undo normalization if a non-normalized distribution is more useful)
- What is the train/eval mode difference for BatchNorm? (train: batch stats; eval: running stats tracked during training; forgetting .eval() is a common source of inference bugs)
- Does BatchNorm act as regularization? Why? (yes — normalization using batch stats introduces noise, similar to dropout; why BN and dropout are sometimes redundant)

## Sources
- [[ML overview]]
