# ML Scenarios — Interview Question Flashcards

**Topic:** [[ml/index]]
**Created:** 2026-04-21

Format: read the scenario, formulate your answer, then expand to check.

---

## FUNDAMENTALS

### Card F1
**Q: Walk me through how you would approach a new ML problem from scratch. What are your steps?**

> [!note]- Answer
> 1. **Understand the business objective** — what KPI does this system optimize? What does a false positive vs. false negative cost?
> 2. **Define the ML task** — classification, regression, ranking, generation? What is the label?
> 3. **Understand the data** — how much? What quality? What is the positive rate? Label collection method?
> 4. **Choose a metric** — F1/PR-AUC for imbalanced, RMSE for regression, NDCG for ranking
> 5. **Establish a baseline** — majority class predictor, mean, simple rule-based system
> 6. **Feature engineering** — start simple; add domain knowledge; check for leakage
> 7. **Model selection** — LightGBM for tabular, BERT for text, ResNet for images; simplest model that meets requirements
> 8. **Evaluate honestly** — cross-validation; never touch the test set during development
> 9. **Error analysis** — read the mistakes; fix the most common failure mode
> 10. **Production considerations** — latency, drift monitoring, retraining schedule
> *See:* [[ml/patterns/model-selection]], [[ml/topics/ml-system-design]]

---

### Card F2
**Q: Your model achieves 99.5% accuracy but your manager is unhappy. The dataset has 0.5% positive rate. What went wrong and how do you fix it?**

> [!note]- Answer
> The model likely predicts "negative" for everything — 99.5% accuracy = 100% negative rate.
>
> **Diagnosis:**
> ```python
> print(classification_report(y_test, y_pred))
> # precision and recall for class 1 = 0.00
> ```
>
> **Fixes:**
> 1. **Change the metric**: use PR-AUC or F1 on the positive class
> 2. **Class weights**: `scale_pos_weight = neg_count / pos_count` in XGBoost
> 3. **Threshold tuning**: default 0.5 is wrong for imbalanced data; tune on PR curve
> 4. **SMOTE** if class weights insufficient
>
> *See:* [[ml/concepts/class-imbalance]], [[ml/concepts/precision-recall-auc]]

---

### Card F3
**Q: Your model performs great in development but poorly in production. List 5 possible causes.**

> [!note]- Answer
> 1. **Data leakage in development** — features contain future or target-derived information; model learns leak, not signal
> 2. **Train/test distribution shift** — production data has different statistical properties than training data (different time period, user segment, region)
> 3. **Feature unavailability in production** — features available in training logs are not available at inference time (latency, privacy, system constraint)
> 4. **Overfitting to test set** — evaluated on test set too many times during development; test set is effectively a validation set
> 5. **Training-serving skew** — features computed differently in offline training vs. online serving (different SQL, different timezone handling, different missing value handling)
>
> *See:* [[ml/concepts/overfitting]], [[ml/topics/ml-system-design]]

---

## TRAINING DEBUGGING

### Card T1
**Q: Your loss is not decreasing at all in the first epoch. What do you check?**

> [!note]- Answer
> **Step 1: One-batch overfit test** (most important)
> ```python
> # Model MUST be able to overfit one batch to near-zero loss
> for i in range(200):
>     loss = model(single_batch_x, single_batch_y)
>     loss.backward(); optimizer.step()
> # If loss doesn't go to near 0 → bug in model, loss, or data
> ```
>
> **Step 2: Check gradient norms**
> ```python
> for name, p in model.named_parameters():
>     print(name, p.grad.norm() if p.grad is not None else 'None')
> # All zeros → vanishing gradient or wrong loss
> ```
>
> **Causes:**
> - Learning rate too low
> - Vanishing gradients (use ReLU; check initialization)
> - Loss function bug (e.g., labels wrong class/shape)
> - Data loader bug (returns same batch; labels wrong)
> - `model.eval()` left on (dropout/BN wrong mode)
>
> *See:* [[ml/scenarios/model-not-converging]]

---

### Card T2
**Q: Your training loss suddenly jumps to NaN at step 200. Walk through your debug process.**

> [!note]- Answer
> **Step 1: Find where NaN first appears**
> ```python
> # Add hooks to each layer
> for name, module in model.named_modules():
>     module.register_forward_hook(
>         lambda m, i, o, name=name: print(f"{name}: NaN={torch.isnan(o).any()}")
>     )
> ```
>
> **Common NaN sources:**
> - `log(0)` in loss: use `BCEWithLogitsLoss` instead of `BCELoss(sigmoid(x))`
> - Division by zero in BN when variance = 0 (all-same-value batch): ensure ε in denominator
> - `sqrt(0)` in distance: add small ε
> - Exploding gradients → NaN: add gradient clipping `clip_grad_norm_(..., 1.0)`
> - Input data contains NaN: `assert not torch.isnan(x).any()`
>
> *See:* [[ml/scenarios/model-not-converging]]

---

## SYSTEM DESIGN

### Card S1
**Q: Design a recommendation system for a music streaming platform with 50M users and 80M songs. What are the key architectural decisions?**

> [!note]- Answer
> **Two-stage pipeline:**
>
> **Stage 1 — Retrieval** (50ms budget, 80M → 500 candidates):
> - Two-tower model: encode user (listening history, demographics) + song (embeddings, metadata) separately
> - ANN search (Faiss HNSW) over song embeddings at query time
> - Also: collaborative filter, trending songs, recently added from liked artists
>
> **Stage 2 — Ranking** (20ms budget, 500 → 20 candidates):
> - GBM or neural ranker with 100+ features
> - Predict P(skip in first 30s) as negative label; P(full listen) as positive
>
> **Key challenges:**
> - Cold start (new users): use metadata, onboarding, popularity fallback
> - Cold start (new songs): use audio embeddings (pre-trained on audio) + creator history
> - Feedback loop: don't only recommend what was listened to → add exploration
>
> *See:* [[ml/scenarios/recommendation-system-design]]

---

### Card S2
**Q: How do you evaluate a recommendation system? What are the tradeoffs between offline and online metrics?**

> [!note]- Answer
> **Offline metrics** (fast, cheap, proxy):
> - NDCG@10: ranking quality; position-weighted; industry standard
> - Hit Rate@K: is the correct item in top-K results?
> - PR-AUC: if framed as binary classification
> - Limitation: based on historical data; doesn't capture what users would click on a fresh recommendation
>
> **Online metrics** (ground truth but expensive):
> - Engagement: streams, listen time, saves
> - Diversity: entropy of genres/artists in recommendations
> - Retention: 7-day and 30-day active users
> - Limitation: A/B test takes weeks; novelty effect; interference
>
> **The gap:** offline NDCG can improve while online metrics stay flat (distribution shift) or vice versa. Never ship based on offline metrics alone.
>
> **A/B testing considerations:** run for minimum 2 weeks (novelty effect); check guardrail metrics (don't regress on song diversity); compute required sample size upfront.

---

### Card S3
**Q: Walk me through designing a fraud detection system. What are the top 5 challenges specific to fraud and how do you address each?**

> [!note]- Answer
> 1. **Severe class imbalance (0.1-1% positive rate)**
>    → PR-AUC metric; class weights or focal loss; SMOTE; threshold tuning
>
> 2. **Label delay** (ground truth comes days/weeks after transaction)
>    → Use rule-based labels for fast feedback; model chargebacks when available; stratify by label type
>
> 3. **Survivorship bias** (blocked transactions never get labeled)
>    → 1-5% random pass-through of suspicious transactions; reject inference for the rest
>
> 4. **Concept drift** (fraud patterns change every few weeks as fraudsters adapt)
>    → Frequent model retraining (weekly); monitor score distribution and recall on labeled data
>
> 5. **Real-time latency constraint** (< 100ms)
>    → Lightweight model (XGBoost on pre-computed features from feature store); graph/deep models async
>
> *See:* [[ml/scenarios/fraud-detection-design]]

---

## DEEP LEARNING

### Card D1
**Q: What happens during fine-tuning that causes "catastrophic forgetting" and how do you prevent it?**

> [!note]- Answer
> **Catastrophic forgetting**: fine-tuning on a new task overwrites the weights encoding the pre-trained knowledge. The new task's gradient updates destroy general representations.
>
> **Prevention strategies:**
> 1. **Small learning rate** (2e-5 instead of 1e-3) — small gradient updates; slow overwriting
> 2. **Progressive unfreezing** — start with only the head; gradually unfreeze lower layers with smaller LR
> 3. **LoRA** — freeze base weights entirely; only update low-rank adapters; pre-trained knowledge is preserved
> 4. **Elastic Weight Consolidation (EWC)** — adds a penalty for changing weights that were important for the original task
> 5. **Mix old task data with new** — continual fine-tuning; prevents forgetting by keeping old distribution in training
>
> *See:* [[ml/patterns/transfer-learning]]

---

### Card D2
**Q: You're training a Transformer language model and the loss is much higher than expected after 1000 steps. Walk through your debug process.**

> [!note]- Answer
> **Step 1: Check the basics**
> ```python
> # Is the model in train mode?
> assert model.training == True
> # Is gradient flowing?
> for name, p in model.named_parameters():
>     assert p.grad is not None, f"No gradient for {name}"
> ```
>
> **Step 2: Check perplexity vs. expected**
> - Random prediction: perplexity = vocab_size (50K for GPT-style)
> - After 1000 steps on a reasonable dataset: should be < 500
> - If still near vocab_size: gradients not flowing, LR too low, wrong tokenization
>
> **Step 3: Check training data**
> ```python
> # Is the tokenizer working correctly?
> tokens = tokenizer.encode("Hello world")
> decoded = tokenizer.decode(tokens)
> assert decoded == "Hello world"
> ```
>
> **Step 4: Confirm causal mask (for GPT)**
> - Mask must be lower triangular; check `model.generate("test")` returns non-random text
>
> **Common issues:** wrong positional encoding, causal mask not applied, tokenizer mismatch, learning rate too low for Transformer (should warmup to ~1e-4 then decay).

---

## CONCEPTS

### Card C1
**Q: Explain the attention mechanism in one paragraph and explain why Transformers can capture longer dependencies than LSTMs.**

> [!note]- Answer
> Self-attention computes a weighted sum of value vectors, where weights are determined by how similar a query vector is to each key vector in the sequence: `Attention(Q,K,V) = softmax(QKᵀ/√dₖ)V`. Every token directly attends to every other token in a single step, regardless of their distance in the sequence.
>
> LSTMs process sequences step by step, passing information through a hidden state. Information from token 1 must survive N recurrent steps to influence token N+1 — and gradients can vanish over those steps. Transformers have a **direct path** between any two tokens — there is no bottleneck. This is why BERT can resolve coreferences ("The trophy didn't fit in the suitcase because **it** was too big") — "it" can directly attend to both "trophy" and "suitcase" simultaneously.
>
> *See:* [[ml/concepts/attention-mechanism]]

---

### Card C2
**Q: What is NDCG@10? Write the formula and explain each component.**

> [!note]- Answer
> **NDCG** = Normalized Discounted Cumulative Gain
>
> ```
> DCG@K = Σᵢ₌₁ᴷ  relᵢ / log₂(i + 1)
>
> NDCG@K = DCG@K / IDCG@K
> ```
> - `relᵢ`: relevance of item at position i (e.g., 0=irrelevant, 1=relevant, 3=highly relevant)
> - `log₂(i+1)`: position discount — item at rank 1 gets full credit; rank 2 gets half; etc.
> - **DCG**: sum of discounted relevance scores
> - **IDCG**: DCG of the ideal (perfect) ranking → used to normalize to [0, 1]
> - **NDCG = 1**: perfect ranking; **NDCG = 0**: all relevant items at the bottom
>
> Used for: search ranking, recommendation systems. Measures whether relevant items appear near the top.
>
> *See:* [[ml/topics/model-evaluation]]
