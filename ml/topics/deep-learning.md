# Deep Learning

**Related concepts:** [[ml/concepts/gradient-descent]], [[ml/concepts/backpropagation]], [[ml/concepts/activation-functions]], [[ml/concepts/batch-normalization]], [[ml/concepts/attention-mechanism]], [[ml/concepts/transformers]], [[ml/concepts/regularization]]

## What it is
Learning hierarchical representations via stacked nonlinear transformations. Requires large data and compute; excels at perception tasks (vision, speech, text) and any problem where features are too complex to hand-engineer.

## Architecture Families

### Feedforward / MLP
- Fully connected layers; no structure assumption
- Use for: tabular data, simple regression/classification
- Weakness: doesn't exploit spatial or sequential structure

### Convolutional Neural Networks (CNNs)
- **Key idea**: parameter-sharing via learned filters; translation invariance
- **Architecture pattern**: Conv → BN → ReLU → Pool → repeat → Flatten → FC → softmax
- **Landmark models**: VGG (simple depth), ResNet (skip connections), EfficientNet (compound scaling), ViT (Vision Transformer)
- **Skip connections (ResNet)**: `output = F(x) + x` — gradient flows directly; enables very deep networks (100+ layers) without vanishing gradient

### Recurrent Neural Networks (RNNs)
- **Vanilla RNN**: hidden state carries sequence memory; suffers from vanishing/exploding gradients over long sequences
- **LSTM**: adds cell state + gates (forget, input, output) — can remember over 1000+ steps
- **GRU**: simplified LSTM (2 gates); similar performance, fewer parameters
- **Mostly replaced by Transformers** for NLP; still used for real-time streaming where causal processing matters

### Transformers
See [[ml/concepts/transformers]] for the architecture deep-dive.
- Encoder-only: BERT (bidirectional; classification, NLU)
- Decoder-only: GPT (autoregressive; generation)
- Encoder-Decoder: T5, BART (seq2seq: translation, summarization)

### Graph Neural Networks (GNNs)
- Message passing on graph structure: aggregate neighbor features
- Use for: molecular property prediction, social networks, fraud detection, knowledge graphs
- Key variants: GCN, GraphSAGE, GAT (attention-weighted neighbors)

## Training Deep Networks

### Initialization
- Random: breaks symmetry
- Xavier/Glorot: `std = sqrt(2 / (fan_in + fan_out))` — keeps variance constant through layers (for tanh/sigmoid)
- He/Kaiming: `std = sqrt(2 / fan_in)` — better for ReLU
- Wrong init → vanishing/exploding gradients from layer 1

### Batch Size Trade-offs
| Batch Size | Pros | Cons |
|---|---|---|
| Large (512+) | Fast wall-clock time; stable gradients | Worse generalization; sharp minima |
| Small (16-64) | Better generalization; noisy gradients escape local minima | Slow; high variance |
| **Rule of thumb**: start with 32-256; if underfitting, increase; if overfitting, reduce or add regularization |

### Learning Rate Schedules
- Cosine annealing: smoothly decay; often best
- Warmup + decay: ramp up for N steps (stabilizes early training), then decay
- Cyclical LR (CLR): oscillate between min/max; helps escape local minima
- 1-cycle policy (fast.ai): warmup → peak → cosine decay in one cycle

## Common Interview Angles
- Why do we need depth? (Universal approximation with one layer requires exponential width; depth enables hierarchical feature reuse)
- ResNet skip connections — why do they help? (Gradient highway; identity mapping at initialization is easy to learn)
- CNN vs. ViT — when to prefer which? (CNNs: small data, compute-constrained; ViT: very large data, better at global context)
- How does LSTM solve vanishing gradients? (Cell state has additive updates; gradients flow back through addition, not multiplication)
- What is batch size's effect on generalization? (Large batches → sharp minima → worse generalization per Keskar et al.)

## Sources
- [[ML overview]]
