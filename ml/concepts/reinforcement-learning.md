# Reinforcement Learning

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/patterns/rlhf]], [[ml/concepts/llm-fundamentals]], [[ml/concepts/gradient-descent]]

---

## What it is

Reinforcement Learning (RL) is a learning paradigm where an **agent** interacts with an **environment** to maximize cumulative **reward**. Unlike supervised learning, there is no labeled dataset — feedback comes as scalar reward signals after taking actions.

---

## Core Concepts

| Concept | Definition |
|---|---|
| **State (s)** | Current situation the agent observes |
| **Action (a)** | Choice the agent makes in state s |
| **Reward (r)** | Scalar feedback after taking action a in state s |
| **Policy (π)** | Mapping from state to action (what to do) |
| **Value function V(s)** | Expected cumulative reward from state s |
| **Q-function Q(s,a)** | Expected cumulative reward from taking action a in state s |
| **Episode** | Sequence from initial state to terminal state |
| **Discount factor (γ)** | Weight on future rewards (0 = myopic, 1 = no discounting) |

**Bellman equation:**
```
Q(s, a) = r + γ × max_{a'} Q(s', a')
```

---

## Policy Gradient Methods

Policy gradient methods directly optimize the policy by gradient ascent on expected reward:

```
∇_θ J(θ) = E[∇_θ log π_θ(a|s) × R]
```

**REINFORCE (Monte Carlo Policy Gradient):**
- Sample full episode; compute return R
- Update: θ ← θ + α × ∇_θ log π_θ(a_t|s_t) × G_t

**Problem:** high variance. Fix: subtract a baseline (advantage function):
```
A(s, a) = Q(s, a) - V(s)
```

**Actor-Critic:**
- **Actor:** learns policy π_θ(a|s)
- **Critic:** learns value function V_φ(s) to compute advantage
- Reduces variance while maintaining low bias

---

## Proximal Policy Optimization (PPO)

PPO is the most widely used RL algorithm in RLHF for LLMs. It is an on-policy, trust-region method that prevents large policy updates that could destabilize training.

**Key idea:** clip the probability ratio between old and new policy:
```
L_CLIP = E[min(r_t(θ) × A_t, clip(r_t(θ), 1-ε, 1+ε) × A_t)]
where r_t(θ) = π_θ(a_t|s_t) / π_θ_old(a_t|s_t)
```

The clip ensures the policy cannot change too much in one update. ε is typically 0.1–0.2.

**Why PPO for RLHF:**
- Stable training with reward model (reward hacking risk if updates are too large)
- Sample efficient relative to REINFORCE
- Parallelizable across many environment rollouts

---

## Q-Learning and DQN

**Q-Learning:** model-free, off-policy algorithm that updates Q values using Bellman:
```
Q(s, a) ← Q(s, a) + α × [r + γ × max_{a'} Q(s', a') - Q(s, a)]
```

**DQN (Deep Q-Network):** use a neural network to approximate Q(s, a). Key tricks:
- **Experience replay:** store past transitions; sample randomly to break correlation
- **Target network:** separate frozen network for computing target values; updated periodically

Limitation: only works for discrete action spaces.

---

## RLHF (RL from Human Feedback)

The dominant technique for aligning LLMs. See [[ml/patterns/rlhf]] for the full pattern.

Three phases:
1. **SFT:** fine-tune pretrained LLM on demonstration data
2. **Reward Modeling:** train a scalar reward model on human preference comparisons
3. **RL (PPO):** optimize LLM with PPO against the reward model + KL penalty

**KL penalty prevents reward hacking:**
```
reward = RM(response) - β × KL(π_θ || π_SFT)
```
β controls the trade-off between reward maximization and staying close to the SFT model.

---

## Multi-Armed Bandit (Google Relevance)

A simplified RL setting with no state transitions: pick an arm (action), observe reward. Foundational for recommendation, ads, and A/B testing.

**Exploration-Exploitation Dilemma:**
- **Exploit:** pick the arm with the highest estimated reward so far
- **Explore:** try other arms to gather information

**Strategies:**
- **ε-greedy:** explore randomly with probability ε; exploit otherwise. Simple but wasteful.
- **UCB (Upper Confidence Bound):** pick arm with highest `estimated_reward + c × sqrt(log(t) / n_i)`. Prioritizes uncertain arms. Theoretically optimal.
- **Thompson Sampling:** maintain a probability distribution over reward for each arm; sample from each and pick the highest. Bayesian; excellent in practice.

**Application at Google:** ads auction — which ad variant to show? News feed — which article to surface? Each impression is a bandit arm.

---

## Common RL Interview Questions (Google)

1. "Explain the exploration-exploitation tradeoff. How does UCB address it?"
2. "What is the policy gradient theorem? Derive the REINFORCE update."
3. "Why is PPO preferred over REINFORCE for RLHF?"
4. "What is reward hacking? How does the KL penalty in RLHF mitigate it?"
5. "How would you apply RL to optimize a recommendation system's long-term engagement?"
6. "What is the credit assignment problem in RL? How does actor-critic help?"
7. "What is the difference between on-policy and off-policy learning?"

---

## RL vs. Supervised Learning Decision Guide

| Scenario | Use RL | Use Supervised |
|---|---|---|
| Labeled data available | No | Yes |
| Environment is sequential (actions affect future state) | Yes | No |
| Reward signal is delayed | Yes | No |
| Optimizing long-term user engagement | Yes | Not directly |
| Aligning LLM to human preferences (RLHF) | Yes | Partly (SFT phase) |

---

## Sources
- [[ml/concepts/gradient-descent]]
- [[ml/patterns/rlhf]]
- [[ml/concepts/llm-fundamentals]]
