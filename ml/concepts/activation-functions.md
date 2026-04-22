# Activation Functions

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/backpropagation]], [[ml/concepts/batch-normalization]], [[ml/concepts/gradient-descent]]

## What it is
A nonlinear function applied element-wise after each linear layer. Without activation functions, a deep network is equivalent to a single linear transformation. Nonlinearity enables learning complex functions.

## Common Activation Functions

### Sigmoid
```
σ(x) = 1 / (1 + e^(-x))     range: (0, 1)
σ'(x) = σ(x)(1 - σ(x))       max derivative: 0.25
```
- Output interpretable as probability
- **Vanishing gradient**: saturates for large |x|; gradient ≈ 0 → early layers learn very slowly
- Use in: binary output layer (not hidden layers), LSTM gates

### Tanh
```
tanh(x) = (eˣ - e⁻ˣ) / (eˣ + e⁻ˣ)    range: (-1, 1)
tanh'(x) = 1 - tanh²(x)               max derivative: 1
```
- Zero-centered (better than sigmoid for hidden layers)
- Still saturates → still vanishing gradient problem
- Use in: RNN/LSTM hidden states

### ReLU (Rectified Linear Unit)
```
ReLU(x) = max(0, x)
ReLU'(x) = 1 if x > 0 else 0
```
- Computationally simple: no exp
- No vanishing gradient for x > 0
- **Dead ReLU problem**: if a neuron receives negative input persistently, gradient = 0 → neuron never updates → "dead"
- Fix: careful initialization (He), lower learning rates, Leaky ReLU

### Leaky ReLU / PReLU
```
LeakyReLU(x) = x if x > 0 else αx   (α typically 0.01)
PReLU: α is a learnable parameter
```
- Solves dead ReLU: small gradient α for x < 0

### ELU (Exponential Linear Unit)
```
ELU(x) = x if x > 0 else α(eˣ - 1)
```
- Smooth at 0; mean activations closer to zero → faster learning
- More expensive (exp computation)

### GELU (Gaussian Error Linear Unit)
```
GELU(x) ≈ x · σ(1.702x)
```
- Used in BERT, GPT, and most modern Transformers
- Smooth approximation of ReLU; slightly stochastic
- Empirically outperforms ReLU on language tasks

### SwiGLU
```
SwiGLU(x, W, V, b, c) = Swish(xW + b) ⊙ (xV + c)
Swish(x) = x · σ(x)
```
- Gated variant; 8/3 expansion factor instead of 4
- Used in LLaMA, PaLM; slightly better than GELU on language tasks

### Softmax
```
softmax(xᵢ) = eˣⁱ / Σⱼ eˣʲ      range: (0, 1), sums to 1
```
- Multi-class output layer; produces probability distribution
- Gradient: `∂softmax(xᵢ)/∂xⱼ = softmax(xᵢ)(δᵢⱼ - softmax(xⱼ))`
- **Temperature softmax**: divide logits by T before softmax. T<1: more peaked; T>1: more uniform

## Choosing an Activation Function

| Layer type | Recommended |
|---|---|
| Hidden layers (MLP, CNN) | ReLU or GELU |
| Transformer hidden layers | GELU or SwiGLU |
| RNN/LSTM hidden states | Tanh |
| LSTM gates | Sigmoid |
| Binary output | Sigmoid |
| Multiclass output | Softmax |
| Regression output | None (linear) |

## The Dying ReLU Problem
If too many neurons die (output = 0 permanently), model capacity shrinks. Signs:
- Loss stops decreasing early in training
- Most neurons output 0 (check with hooks in PyTorch)

Prevention:
- He initialization: `std = sqrt(2/fan_in)`
- Learning rate not too large
- Use Leaky ReLU or ELU if dying is a persistent problem
- Batch normalization stabilizes activations

## Common interview angles
- Why ReLU instead of sigmoid for hidden layers? (no vanishing gradient for x>0; computationally cheap; empirically works better)
- What is the dying ReLU problem and three ways to fix it? (neuron receives only negative inputs → gradient 0 → no update; fix: He init, Leaky ReLU, lower LR)
- Why is GELU preferred over ReLU in Transformers? (empirically better on NLP; smooth; stochastic interpretation)
- What is temperature in softmax and when do you tune it? (divides logits; T→0 = argmax; T→∞ = uniform; tune for generation diversity, knowledge distillation, contrastive learning)
- What is the vanishing gradient problem and which activation functions cause it? (gradients shrink to 0 through many sigmoid/tanh layers; ReLU solves it for positive inputs)

## Sources
- [[ML overview]]
