# Google AI/ML Interview Prep (2026)

**Related:** [[ml/index]], [[dsa/companies/google]], [[ml/topics/ml-system-design]]

---

## Interview Process

Google AI/ML roles (ML Engineer, Research Engineer, Applied Scientist, AI Engineer) follow a consistent structure:

- **Recruiter Screen** (30 min): Background, motivation, basic ML concepts.
- **Technical Phone Screen** (45-60 min): Either ML coding or a conceptual deep-dive.
- **On-site / Virtual Loop** (4-5 rounds):
  - **ML Coding (1-2 rounds):** Implement algorithms from scratch — no libraries.
  - **ML System Design (1 round):** Design a large-scale ML system end-to-end.
  - **Coding / Algorithms (1 round):** Standard DSA, often graphs or DP. See [[dsa/companies/google]].
  - **Googleyness & Leadership (1 round):** Behavioral; structured around impact and ambiguity.

For Research Engineer roles, expect an additional **ML Theory / Research Discussion** round where they probe deep into architectures, optimization theory, or a recent paper.

---

## What Google Weights Heavily

### 1. First-Principles Thinking
Google expects you to derive, not just recall. "How does backpropagation work?" is not satisfied by "chain rule" — they want you to write it out for a 2-layer network. See [[ml/concepts/backpropagation]].

### 2. Scale as a Constraint
Every design decision must acknowledge Google-scale constraints: billions of users, petabytes of data, millisecond latency budgets. "Assume we have enough RAM" is not an acceptable answer.

### 3. Responsible AI Awareness
Google has strong Responsible AI principles. Expect questions about fairness, bias detection, privacy, and the tradeoffs between model accuracy and fairness metrics. This surfaces in both system design and behavioral rounds.

### 4. Production Rigor
They probe the gap between a model that works offline and a system that works in production: training-serving skew, feedback loops, data freshness, and gradual rollout.

### 5. Familiarity with Google's Stack
You are not required to know internal tools, but knowing the concepts behind them signals depth:
- **TensorFlow / JAX** over PyTorch (though PyTorch fluency is fine)
- **TPU architecture** — why training on TPUs favors XLA-compiled, fixed-shape graphs
- **Vertex AI** — managed ML pipeline; conceptually equivalent to Kubeflow / SageMaker
- **BigQuery ML** — SQL-based model training on warehouse data

---

## Round 1: ML Coding

### What They Ask
Implement an ML algorithm from scratch in Python. No sklearn, no PyTorch. They care about:
- Correctness of the math
- Clean vectorized code (NumPy)
- Edge case handling (empty input, zero division, numerical stability)

### Canonical Problems

| Problem | Key Skill |
|---|---|
| Implement gradient descent (batch, SGD, mini-batch) | Optimization fundamentals |
| Implement backpropagation for a 2-layer MLP | Chain rule, weight update |
| Implement softmax + cross-entropy loss | Numerical stability (`log-sum-exp` trick) |
| Implement k-means clustering | Iterative convergence, centroid update |
| Implement logistic regression with L2 | Gradient derivation, regularization term |
| Implement attention (scaled dot-product) | Matrix ops, masking |
| Implement precision, recall, F1, AUC | Threshold sweep, sorted scoring |

### Numerical Stability Pattern
```python
# Naive softmax: exp(x) overflows for large x
def softmax_naive(x):
    return np.exp(x) / np.sum(np.exp(x))

# Stable softmax: subtract max before exp
def softmax_stable(x):
    x = x - np.max(x)
    return np.exp(x) / np.sum(np.exp(x))
```

### Backprop Skeleton (2-layer MLP)
```python
# Forward pass
z1 = X @ W1 + b1      # (N, H)
a1 = np.tanh(z1)      # (N, H)
z2 = a1 @ W2 + b2     # (N, C)
probs = softmax(z2)    # (N, C)
loss = -np.mean(np.log(probs[range(N), y]))

# Backward pass
dz2 = probs.copy()
dz2[range(N), y] -= 1
dz2 /= N
dW2 = a1.T @ dz2      # (H, C)
db2 = dz2.sum(0)
da1 = dz2 @ W2.T      # (N, H)
dz1 = da1 * (1 - a1**2)  # tanh derivative
dW1 = X.T @ dz1
db1 = dz1.sum(0)
```

---

## Round 2: ML System Design

Use the 7-step framework from [[ml/topics/ml-system-design]]. Google's twist: they push hard on **scale**, **evaluation rigor**, and **failure modes**.

### High-Priority Scenarios (Google-Specific)

| Scenario | Primary Challenge | Key Link |
|---|---|---|
| YouTube Video Recommendation | Retrieval at billions of items, multi-objective ranking | [[ml/scenarios/youtube-recommendations]] |
| Google Ads CTR Prediction | Sub-10ms latency, 100B+ impressions/day, position bias | [[ml/scenarios/google-ads-ctr]] |
| Google Search Ranking | Learning-to-rank, freshness vs. authority, SERP layout | [[ml/topics/ml-system-design]] |
| Gmail Smart Reply | Low-latency generation, personalization without privacy leak | [[ml/concepts/llm-fundamentals]] |
| Content Moderation (YouTube/Search) | Multi-modal, human-in-the-loop, precision vs. recall tradeoff | [[ml/scenarios/content-moderation]] |
| LLM Serving System (Gemini) | KV cache, batching, token streaming, cost optimization | [[ml/scenarios/llm-service-design]] |

### Google's Unique Design Probes
- "How do you evaluate your system before going live?" → Shadow mode, offline metrics, interleaving experiments
- "What happens when your training data is poisoned?" → Outlier detection, label auditing, training data versioning
- "How do you handle the cold start problem?" → Content-based fallback, metadata features, cross-domain transfer
- "What are the feedback loops in this system?" → Self-reinforcing popularity, filter bubbles, mitigation strategies

---

## Round 3: Coding / Algorithms

Same as SRE coding, but with more emphasis on graph algorithms relevant to ML infrastructure:
- Dependency resolution for ML pipelines → [[dsa/concepts/graph]] (Topological Sort)
- Scheduling training jobs → [[dsa/problems/task-scheduler]]
- Optimizing data loading → Two-pointer / sliding window on time-series batches

---

## Round 4: Googleyness

Google's behavioral questions probe **intellectual humility**, **cross-functional impact**, and **operating under ambiguity**.

### Common Questions
- "Tell me about a time you disagreed with your team on a technical direction. How did it resolve?"
- "Describe a project where the requirements changed mid-execution. What did you do?"
- "Tell me about the most impactful ML model you shipped. How did you measure impact?"
- "Describe a failure. What did you learn?"

### STAR Framing Adapted for ML
- **Situation:** set the business context and ML objective
- **Task:** what was your specific contribution (model design, infra, evaluation)?
- **Action:** what was the non-obvious decision you made?
- **Result:** quantifiable impact — model accuracy gain, latency reduction, revenue lift

---

## ML Theory Deep-Dives (Research Engineer Track)

Google Research rounds go far deeper than ML Engineer rounds. Expect:
- Derivation of the attention gradient
- Why LayerNorm is preferred over BatchNorm in Transformers
- Tradeoffs between Adam and AdaGrad (second-moment estimation, memory overhead)
- What happens to the loss landscape when you scale model width vs. depth
- Explaining scaling laws (Chinchilla) and their implications for training budget allocation

### Key References Within This Wiki
- [[ml/concepts/attention-mechanism]] — derivation, multi-head, positional encoding
- [[ml/concepts/transformers]] — architecture, BERT vs GPT, scaling
- [[ml/concepts/batch-normalization]] — LayerNorm comparison
- [[ml/concepts/gradient-descent]] — Adam, AdaGrad, learning rate schedules
- [[ml/concepts/distributed-training]] — data parallelism, model parallelism, pipeline parallelism
- [[ml/concepts/llm-fundamentals]] — pretraining, fine-tuning, RLHF, inference optimization

---

## Napkin Math for ML Interviews (Google Style)

| Quantity | Value |
|---|---|
| L1 cache latency | ~1 ns |
| DRAM access | ~100 ns |
| SSD read | ~100 µs |
| Network RTT (same DC) | ~500 µs |
| FP32 multiply-add on GPU | ~1 TFLOPS (A100: 312 TFLOPS) |
| Transformer inference token/s | ~50-200 tokens/s per A100 (depends on model size) |
| Embedding size (typical) | 768 (BERT-base), 1536 (text-embedding-ada) |
| YouTube videos uploaded/min | ~500 hours |
| Google Search QPS | ~99,000 |

---

## Common Interview Mistakes at Google

- **Jumping to a model before clarifying the business objective.** Always ask: what KPI are we optimizing?
- **Treating offline metrics as ground truth.** A/B testing is the final arbiter; offline AUC is a proxy.
- **Ignoring feedback loops.** Recommendation systems change what users see → changes labels → changes next model. Mention this proactively.
- **Under-specifying evaluation.** Say which offline metrics, which guardrail metrics in A/B, and how long you run the experiment.
- **Forgetting the cold start problem.** Google interviewers always ask about new users and new items.
- **Not discussing responsible AI.** Mention fairness, privacy, and potential misuse even if not asked.

---

## Sources
- [[ml/index]]
- [[mlops/index]]
- [[dsa/companies/google]]
- [[ml/topics/ml-system-design]]
