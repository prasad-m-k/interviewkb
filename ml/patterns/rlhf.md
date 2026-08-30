# RLHF (Reinforcement Learning from Human Feedback)

**Topic:** [[ml/topics/deep-learning]]
**Related concepts:** [[ml/concepts/reinforcement-learning]], [[ml/concepts/llm-fundamentals]]

---

## What it solves

LLMs pretrained on next-token prediction can generate fluent text but are not aligned to human values: they hallucinate, are sycophantic, produce harmful content, and don't reliably follow instructions. RLHF closes the gap between "statistically plausible" and "genuinely helpful and safe."

---

## The Three-Phase Pipeline

### Phase 1: Supervised Fine-Tuning (SFT)

Fine-tune the pretrained LLM on a curated dataset of (prompt, ideal_response) pairs written or selected by human labelers.

- Input: base pretrained LLM
- Training data: 10K–100K human-written demonstrations
- Output: SFT model (aligned starting point for RL)

The SFT model already follows instructions well. RLHF refines *preferences* that are hard to demonstrate explicitly (e.g., "be less sycophantic", "be more concise").

### Phase 2: Reward Model (RM) Training

Collect human preference comparisons: for a given prompt, show two model outputs (A, B); ask a rater which is better. Train a scalar reward model to predict human preference.

```
Reward Model Input:  [prompt + response]
Reward Model Output: scalar score (higher = preferred)
```

**Training objective (Bradley-Terry preference model):**
```
L_RM = -log(sigmoid(r_θ(prompt, preferred) - r_θ(prompt, rejected)))
```

**Data:** Google/OpenAI collect hundreds of thousands to millions of preference pairs.

**Pitfall — reward model overfitting:** if the RM sees the same distribution of responses it will be trained on, it may overfit. Diverse policy rollouts during Phase 3 help, but RM quality is a hard ceiling on alignment quality.

### Phase 3: RL Fine-Tuning with PPO

Use the reward model as the reward signal to optimize the SFT model via PPO:

```
For each prompt:
  1. Sample response from current policy π_θ
  2. Score response with reward model: r = RM(prompt, response)
  3. Apply KL penalty: r_adjusted = r - β × KL(π_θ(response) || π_SFT(response))
  4. Update policy with PPO using r_adjusted as reward
```

**Why the KL penalty:** without it, the policy quickly learns to produce degenerate responses that fool the reward model ("reward hacking"). KL penalty keeps the policy close to the SFT model, preserving general language ability.

**Hyperparameters that matter:**
- β (KL coefficient): too low → reward hacking; too high → policy doesn't improve
- PPO clip ratio ε: typically 0.1–0.2
- Number of RL steps: diminishing returns; over-optimization degrades diversity

---

## Template (PyTorch-style pseudocode)

```python
# Phase 1: SFT (standard cross-entropy fine-tuning)
sft_model = pretrained_model.copy()
for prompt, response in sft_dataset:
    loss = cross_entropy(sft_model(prompt), response)
    loss.backward()

# Phase 2: Reward Model
rm = RewardHead(sft_model.encoder)
for prompt, preferred, rejected in preference_dataset:
    r_pos = rm(prompt + preferred)
    r_neg = rm(prompt + rejected)
    loss = -F.logsigmoid(r_pos - r_neg)
    loss.backward()

# Phase 3: PPO
policy = sft_model.copy()
ref_policy = sft_model.copy()  # frozen reference
for prompt in rl_prompts:
    response = policy.generate(prompt)
    reward = rm(prompt + response)
    kl = kl_divergence(policy(prompt), ref_policy(prompt))
    adjusted_reward = reward - beta * kl
    ppo_update(policy, prompt, response, adjusted_reward)
```

---

## Signal phrases (when to apply this pattern)

- "How do you align a pretrained LLM to human preferences?"
- "We trained an LLM but it gives harmful/sycophantic responses"
- "How does ChatGPT/Gemini training work?"
- "What is RLHF and how does it differ from fine-tuning?"

---

## Complexity

- Phase 1 (SFT): same cost as standard fine-tuning
- Phase 2 (RM): same as SFT but smaller dataset; RM head is compact
- Phase 3 (PPO): most expensive — requires 4 model forward passes per step (policy, reference, reward model, value head). Memory: policy + reference + reward model + value head = 4× model size minimum.

---

## DPO: Simpler Alternative

Direct Preference Optimization (DPO) reformulates RLHF as a supervised learning problem over preference pairs — no explicit reward model, no PPO:

```
L_DPO = -log σ(β × log(π_θ(y_w|x)/π_ref(y_w|x)) - β × log(π_θ(y_l|x)/π_ref(y_l|x)))
```

**Advantages over RLHF:**
- No separate RM training or PPO
- More stable; no reward hacking risk
- Same or better empirical performance on many tasks

**Limitation:** the implicit reward may not generalize; cannot do online learning (generate new responses during training).

---

## Problems using this pattern
- Any LLM alignment task
- [[ml/scenarios/llm-service-design]]
- [[ml/companies/google-ml]]

---

## Sources
- [[ml/concepts/reinforcement-learning]]
- [[ml/concepts/llm-fundamentals]]
