# ML Concepts — Anki Deck

> Cards for [[ml/index]]. Sync with Obsidian_to_Anki plugin.
> Deck: `ML::Concepts`

---

## Fundamentals

START
Basic
What is the **bias-variance decomposition** of prediction error? State the formula and explain each term.
Back:
`Total Error = Bias² + Variance + Irreducible Noise`
- **Bias²**: error from wrong model assumptions; underfitting; model too simple
- **Variance**: error from sensitivity to training data; overfitting; model too complex
- **Irreducible noise**: inherent label randomness; can't be eliminated by any model
- Reducing bias tends to increase variance and vice versa — the fundamental trade-off
Tags: ml fundamentals bias-variance
END

START
Basic
How do you diagnose **high bias vs. high variance** from learning curves?
Back:
- **High bias**: train ≈ val ≈ both high error; gap is small; more data doesn't help much
- **High variance**: large gap between train (low error) and val (high error); more data closes the gap
- Fix bias: larger model, more features, less regularization
- Fix variance: more data, more regularization, simpler model, bagging
Tags: ml fundamentals bias-variance overfitting
END

START
Basic
You have 97% train accuracy and 72% val accuracy. What is this called, and what are 5 fixes?
Back:
**Overfitting** (high variance)
1. Get more training data
2. Data augmentation
3. L2 regularization / weight decay
4. Dropout (neural networks)
5. Early stopping
6. Reduce model complexity (fewer layers, shallower trees)
Tags: ml overfitting regularization
END

START
Basic
You have 68% train accuracy and 70% val accuracy on a balanced binary task (baseline=50%). What do you do?
Back:
**Underfitting / high bias** — model can't represent the task
1. Error analysis: read misclassified examples manually
2. Add better / more features (feature engineering)
3. Use a more complex model (deeper network, less regularized tree)
4. Check if labels are correct (label noise)
5. Check if train/test distributions differ
Tags: ml underfitting bias-variance
END

---

## Optimization

START
Basic
What are the three gradient descent variants? Which is used in practice and why?
Back:
- **Batch GD**: gradient over full dataset; stable but slow; not used in DL
- **Stochastic GD (SGD)**: gradient over one sample; noisy; rarely used alone
- **Mini-batch SGD**: gradient over 32-512 samples; standard in practice
Mini-batch gives the best speed/stability trade-off. Frameworks call it "SGD" but mean mini-batch.
Tags: ml optimization gradient-descent
END

START
Basic
Explain **Adam optimizer**. What are its three components and their defaults?
Back:
Adam = momentum (1st moment) + RMSProp (2nd moment) + bias correction
```
m ← β₁·m + (1-β₁)·∇L    # momentum (β₁=0.9)
v ← β₂·v + (1-β₂)·(∇L)²  # adaptive LR (β₂=0.999)
θ ← θ - α·m̂/(√v̂ + ε)
```
- Default: α=0.001, β₁=0.9, β₂=0.999, ε=1e-8
- **AdamW**: decoupled weight decay; preferred for Transformers
Tags: ml optimization adam
END

START
Basic
What happens when the learning rate is **too high** vs. **too low**? How do you find the right LR?
Back:
- Too high: loss oscillates/explodes; training diverges; NaN weights
- Too low: training is very slow; may get stuck in suboptimal region
- **LR finder**: sweep LR from 1e-6 to 1, plot loss vs. LR, pick LR just before loss rises steeply
- **Warmup + cosine decay**: ramp up for N steps (stability), then cosine anneal (best final performance)
Tags: ml optimization learning-rate
END

START
Basic
What is **gradient clipping** and when is it used?
Back:
Cap the gradient norm before the parameter update:
```python
torch.nn.utils.clip_grad_norm_(parameters, max_norm=1.0)
```
- Used when gradients explode (RNNs, deep networks, early Transformer training)
- Does NOT change direction of gradient, only magnitude
- Essential for LLM training; common in any RNN-based model
Tags: ml optimization gradient-clipping
END

---

## Neural Networks

START
Basic
What is **backpropagation**? Why do you need a forward pass first?
Back:
Efficient algorithm to compute ∂L/∂θ for all parameters using the **chain rule**:
```
∂L/∂W₁ = ∂L/∂a₂ · ∂a₂/∂a₁ · ∂a₁/∂W₁
```
Forward pass is needed first to compute and store intermediate activations — the backward pass uses these to compute local gradients at each layer.
Modern frameworks (PyTorch) automate this via a dynamic computation graph (autograd).
Tags: ml backpropagation deep-learning
END

START
Basic
What is the **vanishing gradient problem**? Which activations cause it, and what are the fixes?
Back:
Gradients become near zero as they backpropagate through many layers.
**Causes**: sigmoid (max derivative 0.25) and tanh multiply small numbers across L layers → gradient → 0
**Fixes**:
1. **ReLU** activations (gradient = 1 for x > 0)
2. **Residual connections** (gradient flows directly through x + F(x))
3. **LSTM gates** (additive updates protect cell state)
4. **Layer normalization** (stabilizes activation magnitudes)
5. **Gradient clipping** (for exploding, not vanishing)
Tags: ml deep-learning vanishing-gradient activation
END

START
Basic
Compare **ReLU, GELU, and Sigmoid**. When do you use each?
Back:
| | ReLU | GELU | Sigmoid |
|---|---|---|---|
| Range | [0, ∞) | (-0.17, ∞) | (0, 1) |
| Gradient | 1 or 0 | Smooth | Saturates |
| Dead neurons? | Yes | No | No |
| Use in | CNN/MLP hidden | Transformer hidden | Output (binary), LSTM gates |

GELU is empirically better than ReLU for language models; smooth and stochastic interpretation.
ReLU is fine for CNNs. Never use sigmoid in hidden layers (vanishing gradient).
Tags: ml activation-functions deep-learning
END

START
Basic
What is **BatchNorm** vs. **LayerNorm**? When do you use each?
Back:
- **BatchNorm**: normalizes over the batch dimension; requires large batch size; different stats at train vs. inference; best for CNNs
- **LayerNorm**: normalizes over the feature dimension per sample; works with any batch size; same at train and inference; best for Transformers and RNNs
**Common bug**: forgetting `model.eval()` at inference with BatchNorm → uses batch stats instead of running stats → wrong predictions
Tags: ml batch-normalization layer-norm deep-learning
END

START
Basic
What is **dropout** and how does it prevent overfitting?
Back:
During training, randomly zero out each neuron with probability p. Scale remaining activations by 1/(1-p) to preserve expected output.
- Forces each neuron to be useful independently (prevents co-adaptation)
- Implicitly trains an ensemble of 2^N thinned networks
- Typical rates: p=0.5 for FC layers; p=0.1-0.3 for Transformers
- **Must be disabled at inference**: `model.eval()` in PyTorch
Tags: ml regularization dropout deep-learning
END

---

## Transformers and NLP

START
Basic
Explain **Scaled Dot-Product Attention**. Why do we scale by √dₖ?
Back:
```
Attention(Q, K, V) = softmax(QKᵀ / √dₖ) · V
```
- Q: what am I looking for? K: what do I have? V: what do I provide?
- QKᵀ computes pairwise similarity; softmax → attention weights; apply to V → attended output
- **Scale by √dₖ**: for large dₖ, dot products grow in magnitude → softmax saturates → gradients vanish; scaling keeps variance at 1
Tags: ml transformers attention nlp
END

START
Basic
BERT vs. GPT — what is the fundamental architectural difference and when do you use each?
Back:
- **BERT**: encoder-only; **bidirectional** (attends to both left and right); pre-trained with MLM + NSP; best for understanding tasks (classification, NER, QA, semantic similarity)
- **GPT**: decoder-only; **causal/autoregressive** (attends left only); pre-trained with next token prediction; best for generation tasks (text completion, summarization, few-shot)
- **T5/BART**: encoder-decoder; bidirectional encoder, causal decoder; best for seq2seq (translation, summarization)
Tags: ml transformers nlp bert gpt
END

START
Basic
What is the time and space complexity of self-attention? What are three solutions for long sequences?
Back:
**O(n²·d) time; O(n²) space** for the attention matrix — quadratic in sequence length.
For n=4096: 4096² = 16M attention pairs → expensive.
Solutions:
1. **Sparse attention** (Longformer, BigBird): attend to local window + global tokens
2. **FlashAttention**: same result but I/O-efficient; tiles computation; 2-4× faster; less HBM memory
3. **Linear attention**: kernel approximation replaces softmax → O(n) time
Tags: ml transformers attention complexity nlp
END

START
Basic
What is **LoRA** and when would you use it for fine-tuning?
Back:
Low-Rank Adaptation: instead of updating the full weight matrix W (d×d), decompose the update:
`ΔW = A·B` where A is (d×r), B is (r×d), r << d
Only A and B are trainable; W is frozen. Rank r=8-16 typical.
**Use when**: large model (7B+ params), limited GPU memory, multiple tasks (swap LoRA adapters per task)
**Effect**: ~10-100× fewer trainable parameters; near full fine-tune quality; can run on a single GPU with QLoRA (quantized base)
Tags: ml transformers fine-tuning lora nlp
END

---

## Evaluation and Metrics

START
Basic
For a binary fraud detection model with 0.1% positive rate: which metric do you use and why NOT accuracy?
Back:
Use **PR-AUC** (Average Precision), not accuracy or even ROC-AUC.
- **Accuracy**: a model predicting all-negative gets 99.9% accuracy → useless
- **ROC-AUC**: FPR denominator is dominated by TN (very many negatives) → looks optimistic even when recall is poor
- **PR-AUC**: focuses on positive class; high PR-AUC means model achieves both high precision and recall; correctly penalizes models that miss positives
Also use: Recall@Precision=X% for business decisions ("catch 80% of fraud at < 1% false positive rate")
Tags: ml metrics evaluation class-imbalance
END

START
Basic
What is the difference between **precision** and **recall**? Give a fraud example for each.
Back:
- **Precision** = TP/(TP+FP): "Of all transactions I flagged as fraud, what % were actually fraud?"
  → High precision = few false alarms; every alert is likely real
- **Recall** = TP/(TP+FN): "Of all actual fraudulent transactions, what % did I catch?"
  → High recall = few missed frauds; most fraud is caught

**The trade-off**: lowering the decision threshold raises recall (catch more fraud) but lowers precision (more false alarms blocking legitimate transactions).

**F1** = harmonic mean = balances both. Use F-beta if costs differ.
Tags: ml metrics precision recall
END

START
Basic
What is **calibration** in ML, and how do you fix a miscalibrated model?
Back:
A model is calibrated if predicted probabilities match true frequencies:
- If model predicts p=0.8 for N examples, ~80% should actually be positive
- Miscalibrated models produce overconfident (p too extreme) or underconfident predictions

**Check**: plot reliability diagram (predicted probability bins vs. actual fraction positive)
**Fix**:
1. **Platt scaling**: fit a logistic regression on model scores
2. **Isotonic regression**: fit a non-decreasing step function (more flexible)
3. **Temperature scaling**: divide logits by T (simpler; works well for neural networks)
Tags: ml metrics calibration
END

---

## Ensemble Methods

START
Basic
Compare **Random Forest and XGBoost** — bias, variance, training, and when to use each.
Back:
| | Random Forest | XGBoost |
|---|---|---|
| Training | Parallel trees | Sequential trees |
| Bias | Higher | Lower |
| Variance | Lower (bagging) | Can be higher |
| Overfitting | Low | Moderate (but regularized) |
| Tuning | Minimal | More hyperparameters |
| Use when | Quick baseline | Max accuracy on tabular |

**Rule**: start with RF as baseline; switch to XGBoost/LightGBM when you need to squeeze more performance. RF is more robust out-of-the-box.
Tags: ml ensemble random-forest xgboost
END

START
Basic
What is **boosting** and how does gradient boosting work mechanically?
Back:
Boosting builds trees **sequentially** — each tree corrects the residual errors of the previous ensemble.
```
F₀ = initial prediction (mean of y)
For m = 1 to M:
    rᵢ = -∂L/∂F_{m-1}(xᵢ)   # pseudo-residuals
    Train tree hₘ on rᵢ
    Fₘ = F_{m-1} + η·hₘ     # η: learning rate / shrinkage
```
- With MSE loss: pseudo-residuals = actual residuals (yᵢ - ŷᵢ)
- With log loss (for classification): residuals are probability residuals
- **Shrinkage (η)**: small learning rate → more trees needed → better generalization
Tags: ml ensemble boosting xgboost
END

---

## Feature Engineering

START
Basic
How do you handle **high-cardinality categorical features** (e.g., user_id, zip_code)?
Back:
Three main approaches:
1. **Target encoding** (best for trees): replace category with mean of target; use out-of-fold to prevent leakage; add Bayesian smoothing for rare categories
2. **Embedding table** (best for neural networks): learn a dense d-dimensional vector per category; handles millions of categories; cold start handled with OOV embedding
3. **Frequency encoding** (quick baseline): replace with log(count); captures rarity; no leakage risk

**Never** use one-hot for high-cardinality → creates huge sparse features.
**Avoid** label-leaking target encoding → always compute out-of-fold.
Tags: ml feature-engineering categoricals
END

START
Basic
What is **feature leakage** and give three concrete examples?
Back:
Leakage: features contain information only available **after** the prediction time, or derived from the target variable.
Examples:
1. **Future data**: "total purchases in month" as a feature for a model predicting whether a user buys today — includes future purchases
2. **Target-derived**: using "loan was repaid in 3 months" as a feature for predicting default
3. **Preprocessing leak**: fitting a StandardScaler on the full dataset before splitting — val/test statistics leak into training normalization
**Detection**: suspiciously high offline metrics; model performance collapses in production
Tags: ml feature-engineering leakage
END
