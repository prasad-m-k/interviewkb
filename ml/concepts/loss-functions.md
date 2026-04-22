# Loss Functions

**Topic:** [[ml/topics/supervised-learning]], [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/gradient-descent]], [[ml/concepts/activation-functions]], [[ml/concepts/class-imbalance]]

## What it is
A function measuring the discrepancy between model predictions and true labels. The objective of training is to minimize this function via gradient descent. The choice of loss function encodes your assumptions about the problem.

## Classification Losses

### Binary Cross-Entropy (BCE / Log Loss)
```
L = -[y·log(p) + (1-y)·log(1-p)]
```
- y ∈ {0, 1}, p = sigmoid(logit) ∈ (0, 1)
- Penalizes confident wrong predictions exponentially
- Use with sigmoid output for binary classification
- Numerically stable implementation: `BCEWithLogitsLoss` (combines sigmoid + BCE)

### Categorical Cross-Entropy
```
L = -Σᵢ yᵢ · log(pᵢ)
```
- yᵢ is one-hot; pᵢ = softmax(logitᵢ)
- For K-class classification
- Equivalent to negative log-likelihood of the true class

### Focal Loss
```
L = -αₜ · (1 - pₜ)^γ · log(pₜ)
```
- Modulates cross-entropy: down-weights easy examples via `(1-p)^γ` factor
- γ=0: standard cross-entropy; γ=2: common default
- αₜ: class balancing weight
- Use when: **severe class imbalance** + hard/easy example imbalance (object detection, fraud)

### Hinge Loss (SVM)
```
L = max(0, 1 - y · f(x))    where y ∈ {-1, +1}
```
- Used in SVMs; maximizes margin
- Not differentiable at 0 (use subgradient)
- Rarely used in deep learning

## Regression Losses

| Loss | Formula | Use when |
|---|---|---|
| **MSE / L2** | mean((y-ŷ)²) | Gaussian noise assumption; outliers should be penalized |
| **MAE / L1** | mean(\|y-ŷ\|) | Outliers should not dominate; robust to noise |
| **Huber (Smooth L1)** | MSE for \|e\|<δ, MAE otherwise | Best of both: smooth near 0, robust far from 0 |
| **Log-cosh** | log(cosh(y-ŷ)) | Similar to Huber but fully differentiable |
| **Quantile** | τ·max(e,0) + (1-τ)·max(-e,0) | Predict percentile (τ=0.5 → median) |

## Ranking / Metric Learning Losses

### Contrastive Loss (Siamese Networks)
```
L = y · d² + (1-y) · max(margin - d, 0)²
```
- y=1: same class, minimize distance d
- y=0: different class, push distance above margin

### Triplet Loss
```
L = max(d(anchor, pos) - d(anchor, neg) + margin, 0)
```
- Minimize distance to positive example, maximize distance to negative
- Requires hard negative mining to be effective

### InfoNCE (Contrastive, used in SimCLR, CLIP)
```
L = -log [ exp(q·k⁺/τ) / Σᵢ exp(q·kᵢ/τ) ]
```
- Temperature τ controls sharpness
- k⁺: positive key; kᵢ: all keys including negatives in batch
- Used in: CLIP (image-text alignment), SimCLR (image self-supervised)

### BCEWithLogits vs. Sigmoid + BCE
Always prefer `BCEWithLogitsLoss` over `Sigmoid + BCELoss`:
```python
# Numerically unstable:
p = torch.sigmoid(logit)
loss = F.binary_cross_entropy(p, target)

# Numerically stable (uses log-sum-exp trick):
loss = F.binary_cross_entropy_with_logits(logit, target)
```

## Handling Class Imbalance in Loss
```python
# PyTorch: weight positive class by ratio neg/pos
pos_weight = torch.tensor([neg_count / pos_count])
loss = F.binary_cross_entropy_with_logits(logit, target, pos_weight=pos_weight)
```
See also: [[ml/concepts/class-imbalance]]

## Common interview angles
- Why is MSE bad for classification? (probability outputs between 0-1; MSE gradient is flat near 0 and 1 for sigmoid; cross-entropy has steeper gradient → faster learning)
- Why does log loss penalize confident wrong predictions so heavily? (log(0) = -∞; predicting p=0.01 for y=1 contributes a very large loss)
- When would you use Huber loss over MSE? (when training data has outliers that would dominate MSE; Huber limits their influence)
- What is label smoothing and which loss does it modify? (replace one-hot with (ε/(K-1), 1-ε); modifies cross-entropy; prevents overconfidence)
- How does focal loss help with class imbalance? (down-weights easy negatives via (1-p)^γ; model focuses gradient on hard examples)

## Sources
- [[ML overview]]
