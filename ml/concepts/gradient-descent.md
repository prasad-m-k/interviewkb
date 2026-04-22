# Gradient Descent

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/backpropagation]], [[ml/concepts/loss-functions]], [[ml/concepts/batch-normalization]]

## What it is
An iterative optimization algorithm that minimizes a loss function by moving model parameters in the direction of steepest descent (negative gradient).

```
θ ← θ - α · ∇L(θ)
```
where α is the learning rate, ∇L(θ) is the gradient of the loss with respect to parameters.

## Variants

| Variant | Gradient computed on | Pros | Cons |
|---|---|---|---|
| **Batch GD** | Full dataset | Stable, accurate gradient | Slow per update; needs all data in memory |
| **Stochastic GD (SGD)** | One sample | Very fast per step; escapes local minima | Noisy; slow to converge |
| **Mini-batch SGD** | Batch of 32-512 | Balance of speed and stability | Requires batch size tuning |

Mini-batch is the default. "SGD" in frameworks typically means mini-batch SGD.

## Optimizers

### Momentum
```
v ← β·v + ∇L
θ ← θ - α·v
```
Accumulates gradient direction. Helps cross flat regions and oscillations. β ≈ 0.9.

### RMSProp
```
s ← β·s + (1-β)·(∇L)²
θ ← θ - α · ∇L / (√s + ε)
```
Adapts learning rate per parameter based on recent gradient magnitude. Good for non-stationary problems (RNNs).

### Adam (Adaptive Moment Estimation)
```
m ← β₁·m + (1-β₁)·∇L           # 1st moment (mean)
v ← β₂·v + (1-β₂)·(∇L)²        # 2nd moment (variance)
m̂ = m / (1-β₁ᵗ)                 # bias correction
v̂ = v / (1-β₂ᵗ)
θ ← θ - α · m̂ / (√v̂ + ε)
```
Defaults: α=0.001, β₁=0.9, β₂=0.999, ε=1e-8. Default for most deep learning.

### AdamW
Adam + decoupled weight decay. Fixes L2 regularization behavior in Adam. **Preferred over Adam** for Transformers.

### Comparison
| Optimizer | Best for | Caveat |
|---|---|---|
| **SGD + momentum** | CNNs, often best final accuracy | Sensitive to LR; needs tuning |
| **Adam** | Transformers, RNNs, fast prototyping | Can converge to sharp minima |
| **AdamW** | Transformers, LLMs | Same as Adam but better generalization |
| **RMSProp** | RNNs | Rarely used with Transformers |

## Learning Rate

The single most important hyperparameter.

- Too high → loss explodes or oscillates
- Too low → training is extremely slow; may never converge

### Learning Rate Schedules
- **Step decay**: reduce by factor at fixed epochs
- **Cosine annealing**: `α = α_min + 0.5·(α_max - α_min)·(1 + cos(πt/T))` — smooth decay to minimum
- **Warmup + decay**: linearly increase for N steps, then decay (standard for Transformers)
- **Cyclical LR**: oscillate; helps escape saddle points

### Learning Rate Finder (fast.ai)
Sweep LR over several orders of magnitude, plot loss vs LR, pick the LR just before the loss starts rising steeply.

## Gradient Problems

### Vanishing Gradients
- Gradients become near zero as they backpropagate through many layers
- Causes: sigmoid/tanh saturate; deep networks compound small gradients via chain rule
- Fixes: ReLU activations, residual connections, LSTM gates, gradient clipping, BatchNorm

### Exploding Gradients
- Gradients become very large
- Causes: poor initialization, deep networks, RNNs on long sequences
- Fix: **gradient clipping** — `if ||g|| > threshold: g ← g · threshold / ||g||`

## Saddle Points vs. Local Minima
In high-dimensional loss landscapes, saddle points (gradient = 0 but not a minimum) are far more common than local minima. SGD's noise helps escape saddle points; Adam's adaptive learning rate also helps.

## Common interview angles
- Why does SGD generalize better than Adam? (noisier gradient → wider minima → better generalization; Adam converges faster but to sharper minima)
- What is gradient clipping and when is it used? (cap gradient norm; essential for RNNs and Transformer training)
- What happens if learning rate is too large? (loss oscillates or explodes; weights diverge to NaN)
- Why warm up the learning rate for Transformers? (early in training, weights are random and gradients are noisy; large LR causes instability; warmup lets the optimizer settle)

## Sources
- [[ML overview]]
