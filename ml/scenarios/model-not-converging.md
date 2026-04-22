# Scenario: Model Not Converging

**Category:** Training Debugging
**Difficulty:** Medium
**Seen at:** Google, Meta, Amazon, Microsoft ML interviews

## The Scenario
Your training loss is not decreasing, or it oscillates wildly and never settles. Or it was decreasing and then suddenly exploded to NaN.

## Systematic Debug Steps

### Step 1: Check the loss curve shape

| Pattern | Likely Cause |
|---|---|
| Loss stays flat from the start | Learning rate too low, wrong optimizer, vanishing gradient, data loading bug |
| Loss oscillates and diverges | Learning rate too high |
| Loss decreases then explodes to NaN | Learning rate too high for later training; gradient explosion |
| Loss decreases very slowly | Learning rate too low |
| Loss decreases then plateaus early | Model too small; underfitting |

### Step 2: Sanity check — can the model overfit one batch?
```python
# THE MOST IMPORTANT DEBUG STEP
# Take ONE batch of 32 examples; train on it for 100+ steps
for i in range(100):
    loss = model(batch_x, batch_y)
    loss.backward()
    optimizer.step()
    print(f"Step {i}: loss={loss.item():.4f}")
# Expected: loss should go to near 0 within 50 steps
# If not: bug in model architecture, loss function, or data pipeline
```

### Step 3: Check gradients
```python
# Check for vanishing or exploding gradients
for name, param in model.named_parameters():
    if param.grad is not None:
        print(f"{name}: grad_norm={param.grad.norm():.4f}, "
              f"weight_norm={param.norm():.4f}")
```
- Gradient norms near 0 everywhere → vanishing gradient
- Gradient norms > 1000 → exploding gradient → add gradient clipping
- NaN gradients → overflow; check loss function, data, initialization

### Step 4: NaN / Inf hunting
```python
# Add NaN checks in forward pass
import torch
def check_nan(tensor, name):
    if torch.isnan(tensor).any():
        print(f"NaN detected in {name}")
    if torch.isinf(tensor).any():
        print(f"Inf detected in {name}")
```
Common NaN sources:
- `log(0)` in loss function — use `log(x + ε)` or `BCEWithLogitsLoss` instead of `BCELoss(sigmoid(x))`
- Division by zero in normalization (variance = 0) — add ε
- `sqrt(0)` in distance computation — add ε

### Step 5: Learning rate diagnosis
```python
# Learning rate finder: sweep LR, plot loss
lrs = np.logspace(-6, 0, 100)
for lr in lrs:
    optimizer.param_groups[0]['lr'] = lr
    loss = train_one_step(model, batch)
    losses.append(loss.item())
# Pick LR just before loss starts increasing
```

### Step 6: Check data pipeline
```python
# Verify a batch looks correct
batch_x, batch_y = next(iter(train_loader))
print(f"Input shape: {batch_x.shape}")
print(f"Input range: [{batch_x.min():.2f}, {batch_x.max():.2f}]")
print(f"Label distribution: {batch_y.unique(return_counts=True)}")
assert not torch.isnan(batch_x).any(), "NaN in input data"
```
Common data bugs: labels are shifted by 1, input is not normalized, all labels are the same class.

## Fix Strategies by Root Cause

| Root Cause | Fix |
|---|---|
| LR too high | Reduce by 10×; use LR warmup |
| LR too low | Increase; use LR finder |
| Vanishing gradient | ReLU activations; residual connections; gradient clipping; better init |
| Exploding gradient | Gradient clipping: `torch.nn.utils.clip_grad_norm_(params, 1.0)` |
| NaN in loss | Fix log/sqrt of zero; use numerically stable implementations |
| Wrong initialization | He init for ReLU; Xavier for tanh |
| Batch normalization in eval mode | Always call `model.train()` during training |
| Data pipeline bug | Verify one batch manually; check label distribution |

## Sources
- [[ml/concepts/gradient-descent]]
- [[ml/concepts/backpropagation]]
- [[ML overview]]
