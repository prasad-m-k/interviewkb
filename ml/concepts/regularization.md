# Regularization

**Topic:** [[ml/topics/supervised-learning]], [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/bias-variance-tradeoff]], [[ml/concepts/overfitting]], [[ml/concepts/dropout-and-batch-norm]]

## What it is
Techniques that reduce model variance (overfitting) by adding constraints, penalties, or noise during training. Shifts the bias-variance tradeoff toward higher bias, lower variance.

## L2 Regularization (Ridge / Weight Decay)
```
L_total = L_task + λ · Σ wᵢ²
∂L/∂w = ∂L_task/∂w + 2λw
```
- Penalizes large weights; shrinks them toward zero but not exactly to zero
- Equivalent to a Gaussian prior on weights
- Used in: Ridge Regression, weight decay in neural networks (AdamW)
- Effect on neural nets: encourages smaller weights → smoother functions → less overfitting

## L1 Regularization (Lasso)
```
L_total = L_task + λ · Σ |wᵢ|
```
- Pushes many weights to exactly zero → **automatic feature selection**
- Equivalent to a Laplace prior on weights
- Use when you expect a sparse solution (few features are truly relevant)
- Harder to optimize: derivative is undefined at 0; use subgradient methods
- L1 + L2 = Elastic Net (best of both: sparsity + stability)

## Dropout
```python
# Training: randomly zero out p fraction of neurons
mask = Bernoulli(1-p) for each neuron
output = activation * mask / (1-p)    # scale by 1/(1-p) to preserve expected output

# Inference: no dropout; full network used
```
- Trains an ensemble of 2^n "thinned" networks implicitly
- Forces each neuron to be useful independently (reduces co-adaptation)
- Typical rates: p=0.5 for FC layers; p=0.1-0.2 for Transformer attention; rarely used in CNNs (BN does the job there)
- **DropConnect**: drop weights (not activations) — less common
- **Variational Dropout**: consistent mask across time steps for RNNs

## Early Stopping
Monitor validation loss and stop training when it starts increasing:
```
best_val_loss = ∞
patience = 10
for epoch in training:
    train(model)
    val_loss = evaluate(val_set)
    if val_loss < best_val_loss:
        best_val_loss = val_loss
        save_checkpoint(model)
        patience_counter = 0
    else:
        patience_counter += 1
        if patience_counter >= patience:
            restore_best_checkpoint()
            break
```
- Cheapest form of regularization
- Works because: training loss continues to decrease while validation loss increases → overfitting has started

## Data Augmentation
Creating modified versions of training examples to increase effective dataset size and teach invariances.
- Images: crop, flip, rotate, color jitter, CutMix, MixUp
- Text: synonym replacement, back-translation, random deletion
- Tabular: Gaussian noise, SMOTE (for class imbalance)
See [[ml/patterns/data-augmentation]]

## Other Regularization Techniques
| Technique | How it works | Where used |
|---|---|---|
| **Label smoothing** | Replace hard 0/1 labels with (ε/(K-1), 1-ε); prevents overconfidence | Classification, especially NLP |
| **Mixup** | Train on convex combinations: x̃ = λxᵢ + (1-λ)xⱼ, ỹ = λyᵢ + (1-λ)yⱼ | Image, NLP |
| **Stochastic depth** | Randomly skip entire residual blocks during training | ResNets; reduces effective depth |
| **Spectral normalization** | Constrain Lipschitz constant of each layer | GANs, stability |
| **Noise injection** | Add Gaussian noise to inputs or activations | General; similar to dropout |
| **Max-norm constraint** | Clip weight vector norms: \|\|w\|\| ≤ c | FC layers; used with dropout |

## Choosing Regularization Strength (λ)
- Higher λ → more regularization → higher bias, lower variance
- Tune λ using cross-validation or a held-out validation set
- For dropout rate: higher rate = more regularization; p=0.5 is a common starting point

## Common interview angles
- L1 vs. L2 — key difference? (L1 → sparsity/feature selection; L2 → shrinkage, all features kept but small)
- Why does dropout help generalization? (forces redundant representations; acts as ensemble; prevents co-adaptation)
- How does weight decay relate to L2 regularization? (weight decay is exact L2 for SGD; for Adam, they differ — AdamW corrects this)
- When would you NOT use dropout? (small CNNs where BatchNorm already regularizes; when data is huge)
- How does early stopping relate to L2 regularization? (equivalent in the sense that both constrain effective model complexity)

## Sources
- [[ML overview]]
