# Backpropagation

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/gradient-descent]], [[ml/concepts/activation-functions]], [[ml/concepts/loss-functions]]

## What it is
An efficient algorithm to compute gradients of the loss with respect to all parameters in a neural network, using the chain rule of calculus. Enables gradient-based optimization of deep networks.

## How it works

### Forward pass
Compute the output and loss layer by layer:
```
x → [W₁, b₁] → a₁ → [W₂, b₂] → a₂ → ... → ŷ → L(ŷ, y)
```

### Backward pass (chain rule)
Compute ∂L/∂W for each layer by propagating gradients backward:
```
∂L/∂W₁ = ∂L/∂a₂ · ∂a₂/∂a₁ · ∂a₁/∂W₁
```

At each layer:
```
∂L/∂Wᵢ = δᵢ · aᵢ₋₁ᵀ
∂L/∂bᵢ = δᵢ
δᵢ₋₁   = Wᵢᵀ · δᵢ ⊙ σ'(zᵢ₋₁)    # δ: error signal; σ': activation derivative
```

### Computational Graph
Modern frameworks (PyTorch, JAX) build a dynamic computation graph during the forward pass, then walk it backward to compute all gradients automatically — this is **autograd**.

## Why Deep Networks Have Gradient Problems

### Vanishing Gradient
When using sigmoid or tanh, the derivative is at most 0.25 (sigmoid) or 1 (tanh), and it can be much smaller for saturated neurons. Multiplying many small numbers across L layers → gradient ≈ 0 at early layers.

```
σ'(x) = σ(x)(1-σ(x)) ≤ 0.25
Gradient at layer 1 ∝ (0.25)^L → 0 as L grows
```

**Fix:** ReLU (derivative = 1 for x > 0), residual connections, LSTM gates.

### Exploding Gradient
When weight matrices have large singular values, gradients can grow exponentially. Common in RNNs.
**Fix:** gradient clipping, careful initialization, BatchNorm.

## Automatic Differentiation (Autograd)
```python
import torch

x = torch.tensor([2.0], requires_grad=True)
y = x ** 3 + 2 * x      # forward pass builds the graph
y.backward()             # backprop computes dy/dx
print(x.grad)            # dy/dx = 3x² + 2 = 14 at x=2
```

PyTorch uses **dynamic graphs** (define-by-run) — the graph is rebuilt each forward pass, which is flexible but has overhead. TensorFlow 1.x used static graphs (define-then-run).

## Backprop Through Common Operations

| Operation | Forward | Backward (∂L/∂x) |
|---|---|---|
| Addition (z=x+y) | z = x + y | ∂L/∂x = ∂L/∂z |
| Multiplication (z=x·y) | z = x·y | ∂L/∂x = y · ∂L/∂z |
| ReLU (z=max(0,x)) | z = max(0,x) | ∂L/∂x = ∂L/∂z if x>0 else 0 |
| Softmax | complex | ∂L/∂z = ŷ - y (for cross-entropy) |
| BatchNorm | normalizes | gradient through mean/variance |

## Complexity
- Forward pass: O(parameters)
- Backward pass: O(parameters) — approximately 2× forward pass cost
- Memory: need to cache all activations during forward pass for backward → memory bottleneck for large models
  - Solution: **gradient checkpointing** — recompute activations during backward instead of storing them; trades compute for memory

## Common interview angles
- Explain backprop in one sentence. (Apply chain rule from loss to each parameter, propagating gradient signals backward through the computation graph.)
- Why can't you backprop through argmax? (Argmax has zero gradient almost everywhere; not differentiable; solutions: softmax approximation, straight-through estimator, Gumbel-Softmax)
- What is gradient checkpointing? (Recompute intermediate activations during backward pass instead of storing them; reduces memory from O(L) to O(√L) at cost of extra compute)
- Why does backprop need a forward pass first? (Need activations at each layer to compute local gradients; chain rule requires intermediate values)

## Sources
- [[ML overview]]
