# Google AI/ML Interview — Top 20 Flashcards

**Company:** [[ml/companies/google-ml]]
**Format:** Front (Q) / Back (A)

---

## Core ML Theory

**Q1. Derive the softmax gradient.**
A: For output `p_i = exp(z_i) / sum(exp(z_j))`:
`∂L/∂z_i = p_i - y_i` (when combined with cross-entropy loss). This is cleaner than the raw softmax Jacobian because cross-entropy + softmax simplifies to a linear residual. Memorize this; Google coding rounds ask you to implement it.

---

**Q2. Why does Adam use bias correction in the first few steps?**
A: The first-moment estimate `m_t` and second-moment estimate `v_t` are initialized to zero. Early in training, they are biased toward zero before enough gradient signal accumulates. Bias correction (`m_t / (1 - β1^t)`, `v_t / (1 - β2^t)`) compensates so the learning rate is not artificially inflated in early steps. Without it, first steps are too large and can destabilize training.

---

**Q3. Explain the vanishing gradient problem. When does it occur and how is it fixed?**
A: In deep networks with sigmoid/tanh activations, gradients are repeatedly multiplied by derivatives that saturate near 0 or 1 (sigmoid: max derivative 0.25). After many layers, the gradient signal approaches zero and early layers barely update. Fixes: ReLU activations (derivative = 1 in active region), residual connections (skip connections add the gradient path directly), gradient clipping, BatchNorm/LayerNorm.

---

**Q4. What is the difference between BatchNorm and LayerNorm? Which does Google prefer for Transformers?**
A: BatchNorm normalizes over the batch dimension (mean and variance computed across the batch per feature). LayerNorm normalizes over the feature dimension (mean and variance computed per sample). For Transformers, LayerNorm is preferred because: (1) batch size 1 is valid (e.g., autoregressive decoding), (2) variable-length sequences make batch statistics noisy, (3) LayerNorm works identically at train and inference time. Google's Transformer variants (and most LLMs) use pre-LN (LayerNorm before attention/FFN, not after).

---

**Q5. What are scaling laws? What is the Chinchilla finding?**
A: Scaling laws (Kaplan et al., 2020) show that LLM loss follows power laws in model size (N), dataset size (D), and compute (C). The Chinchilla paper (Hoffmann et al., 2022) showed that the original GPT-3 era models were undertrained: optimal allocation is to scale D proportionally to N (roughly D ≈ 20N tokens). Same compute budget → smaller model trained on more data often beats a larger undertrained model. Google's Gemini models follow Chinchilla-optimal or compute-optimal training.

---

## ML System Design

**Q6. Explain the two-stage retrieval + ranking pipeline. Why is it necessary?**
A: A heavy ranker (e.g., 100-feature neural network) cannot score 800M videos per request in 200ms. Stage 1 (retrieval): a fast approximate model (two-tower + ANN search) narrows the corpus to 100-1000 candidates. Stage 2 (ranking): the heavy model scores only the candidates with rich features. Retrieval = scalability lever; ranking = quality lever. This is used in YouTube, Google Ads, and Google Search.

---

**Q7. What is position bias and how do you correct for it in a CTR model?**
A: Items shown in higher positions receive more clicks regardless of quality. A naive model trained on raw clicks learns position as a proxy for relevance. Two fixes: (1) Add position as a training feature; set it to zero at inference. (2) Inverse Propensity Scoring: weight each training example by `1 / P(shown at position k)`. IPS is unbiased but high-variance; position-as-feature is biased but lower-variance. Google uses position-as-feature with a learned examination model.

---

**Q8. What is training-serving skew? Give an example and the fix.**
A: Training-serving skew occurs when features are computed differently offline (training) vs. online (serving). Example: a recommendation model trains on user's "total watch time in the past 7 days" computed in batch from a warehouse. At serving, the same feature is computed from a streaming system that only has the last 24 hours. The model is served with a different distribution than it was trained on. Fix: use a feature store that serves the same computation logic offline and online (point-in-time correct). See [[mlops/concepts/training-serving-skew]].

---

**Q9. Explain the cold start problem and three approaches to handle it.**
A: Cold start: new users or new items have no interaction history, so collaborative filtering produces no signal. Approaches: (1) Content-based fallback — use item metadata (title, category, description) to find similar items with history. (2) Popularity-based fallback — recommend trending or high-engagement items. (3) Meta-learning / transfer — transfer embeddings from similar items or users in adjacent domains. For new users, an onboarding flow that elicits explicit preferences is the cleanest solution.

---

**Q10. What is a feedback loop in an ML system? Give a YouTube example.**
A: A feedback loop occurs when model predictions influence the data used to train the next model. YouTube example: the recommender surfaces certain videos → those videos get more watches → they become "popular" in the next training set → the model reinforces their dominance → non-recommended content never gets discovered. Mitigations: counterfactual logging (log what would have been shown with a random policy), exploration policy (reserve 5-10% of recommendations for exploration), off-policy evaluation.

---

## LLM / Generative AI

**Q11. What is the KV cache and why does it matter for inference?**
A: During autoregressive generation, attention requires the Key and Value matrices for every previous token. Without caching, computing attention at token t costs O(t²). The KV cache stores precomputed K and V matrices for all previous tokens; each new token only needs O(1) new attention computation plus O(t) memory reads. Trade-off: massive memory consumption — a 70B model with 8K context needs ~35GB for the KV cache alone, often exceeding model weight memory.

---

**Q12. What is the difference between RAG and fine-tuning for domain adaptation?**
A: RAG retrieves relevant documents at inference time; knowledge stays external and is easily updated. Fine-tuning bakes knowledge into model weights via additional training; knowledge is opaque and requires retraining to update. RAG is preferred when: knowledge changes frequently, citations are needed, data is proprietary (never enters weights), or a fine-tuning dataset is unavailable. Fine-tuning is preferred when: latency is critical (no retrieval step), behavior/style needs to change (not just knowledge), or the model needs to generalize beyond available documents.

---

**Q13. Explain RLHF in three sentences.**
A: First, fine-tune the base LLM on human-written demonstrations (SFT). Then, collect human preference comparisons between model outputs and train a reward model to predict human preference. Finally, use PPO to optimize the SFT model against the reward model's scores, with a KL penalty to prevent the policy from drifting too far from the SFT model.

---

**Q14. What is DPO and how does it improve on RLHF?**
A: Direct Preference Optimization (DPO) reformulates RLHF as supervised learning on preference pairs, eliminating the need for a separate reward model and PPO. It directly optimizes the policy by treating the optimal policy as implicitly defined by the reward model's preferences. Advantages: simpler, more stable, same or better empirical performance. Limitation: requires offline preference data; cannot do online learning where the model generates new data during training.

---

**Q15. How does speculative decoding work and what is its speedup?**
A: A small draft model generates K candidate tokens cheaply. The large verifier model checks all K tokens in a single parallel forward pass. Tokens accepted where verifier and draft model agree; regenerate from the first disagreement. Accepted tokens are generated at the cost of ~1 large model forward pass instead of K separate passes. Typical speedup: 2-3× for long generation. Zero quality change — outputs are provably equivalent to sampling from the large model alone.

---

## ML Infrastructure

**Q16. What is the difference between data parallelism and tensor parallelism?**
A: Data parallelism: each device holds a full model copy; dataset is sharded; gradients are aggregated (AllReduce) after each step. Scales to 1000s of GPUs but requires the model to fit in one GPU. Tensor parallelism: each layer's weight matrices are split across devices; both devices participate in every forward pass; requires high-bandwidth communication (NVLink) per layer. Required when model size exceeds one GPU. Modern systems use both together.

---

**Q17. What is ZeRO / FSDP and when would you use it?**
A: ZeRO (Zero Redundancy Optimizer) shards optimizer states, gradients, and model parameters across data-parallel workers, eliminating redundant storage. Stage 3 / PyTorch FSDP shards everything — each GPU only holds a shard of the model, gathering parameters from peers during each forward/backward pass. Use when: model is too large for one GPU but you want data-parallel-style training without the complexity of tensor parallelism. Trade-off: higher communication volume vs. standard data parallelism.

---

**Q18. Why does Google prefer BF16 over FP16 for training on TPUs?**
A: BF16 has the same exponent width as FP32 (8 bits), giving the same dynamic range and avoiding the overflow/underflow issues that require loss scaling in FP16 training. FP16 has more mantissa bits (10 vs. 7) but a smaller exponent range, which requires careful gradient scaling. On TPUs, which are natively optimized for BF16, this eliminates loss scaling complexity entirely and is the default for all LLM training at Google.

---

## Responsible AI

**Q19. How would you detect and measure bias in a content moderation classifier?**
A: Disaggregate evaluation: split the test set by demographic proxies (language variety, protected attribute proxies in the content). Measure false positive rate (wrongly flagging legitimate content) and false negative rate (missing violations) separately per group. Key finding at Google: classifiers trained on web text have higher FPR on African-American Vernacular English and LGBTQ+ speech because of training data imbalance. Mitigations: fairness-aware training (reweighting), balanced evaluation sets, threshold adjustment per demographic to equalize FPR.

---

**Q20. What is the difference between individual fairness and group fairness? Which should you use?**
A: Individual fairness: similar individuals should receive similar predictions (mathematical definition requires a similarity metric between individuals — hard to define). Group fairness: statistical parity across demographic groups — e.g., equal FPR for male and female candidates. Common group metrics: demographic parity (equal positive rate), equalized odds (equal TPR and FPR), equal opportunity (equal TPR only). Tradeoff: Impossibility theorem (Chouldechova 2017) shows demographic parity, equalized odds, and calibration cannot all hold simultaneously when base rates differ across groups. In interviews, name the metric you'd use and acknowledge the tradeoffs explicitly.

---

## Sources
- [[ml/companies/google-ml]]
- [[ml/concepts/llm-fundamentals]]
- [[ml/concepts/distributed-training]]
- [[ml/concepts/reinforcement-learning]]
- [[ml/patterns/rlhf]]
- [[ml/concepts/rag]]
