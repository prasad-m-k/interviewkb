# Reflection Pattern (Self-Critique)

**Topic:** [[agents/topics/agent-architectures]]
**Related concepts:** [[agents/concepts/reasoning-and-planning]], [[agents/concepts/agent-evaluation]], [[agents/concepts/multi-agent-orchestration]]

---

## What it solves

A model's first-pass output is often good but not great — it can contain errors it would catch if explicitly asked to review its own work, the same way a human's first draft benefits from a second pass. Reflection formalizes this: generate a draft, critique it (by the same model in a second pass, or a separate critic), and revise — optionally looping until the critique passes or a round limit is hit.

---

## Template / Skeleton

```python
def reflection_agent(task: str, max_rounds: int = 3) -> str:
    draft = generator_llm.generate(prompt=f"Task: {task}")

    for round in range(max_rounds):
        critique = critic_llm.generate(
            prompt=f"""Task: {task}
Draft: {draft}

Critique this draft against the task. List concrete problems, if any.
If it fully satisfies the task, respond with exactly: APPROVED"""
        )

        if critique.strip() == "APPROVED":
            return draft

        draft = generator_llm.generate(
            prompt=f"""Task: {task}
Previous draft: {draft}
Critique: {critique}

Revise the draft to address the critique."""
        )

    return draft  # return best-effort after max_rounds, flagged as unreviewed
```

---

## Worked Example

```
Task: "Write a SQL query to find the top 5 customers by total spend in 2025."

Draft 1:
  SELECT customer_id, SUM(amount) FROM orders
  GROUP BY customer_id ORDER BY SUM(amount) DESC LIMIT 5;

Critique: "This doesn't filter to 2025 — it sums spend across all time,
not just the requested year."

Draft 2:
  SELECT customer_id, SUM(amount) AS total_spend FROM orders
  WHERE order_date >= '2025-01-01' AND order_date < '2026-01-01'
  GROUP BY customer_id ORDER BY total_spend DESC LIMIT 5;

Critique: APPROVED
```

---

## Signal Phrases

- "The agent's output needs a quality/correctness check before it's used"
- "Build a code-review step into the agent's workflow"
- "We need the agent to catch its own mistakes before responding"
- "Design a generator + critic system"

---

## Same-Model vs. Separate-Critic Reflection

| | Same model reflects on itself | Separate critic model/agent |
|---|---|---|
| Cost | Lower (one model, extra calls) | Higher (potentially a second, possibly stronger model) |
| Bias risk | Higher — a model can be blind to its own systematic errors | Lower — an independent critic (especially a stronger or differently-prompted model) catches more |
| Setup complexity | Simple | Requires defining a separate role/prompt (or agent) for the critic |
| When to use | Low-stakes tasks, cost-sensitive settings | High-stakes tasks, or where the generator's failure modes are known to be self-blind |

A separate critic is one instance of the evaluator-optimizer multi-agent topology — see [[agents/concepts/multi-agent-orchestration]].

---

## Complexity

| Aspect | Notes |
|---|---|
| Cost | Roughly 2× to (2 × max_rounds)× a single generation, depending on how many rounds actually run |
| Latency | Sequential rounds add up — not ideal for latency-critical interactive use unless rounds are capped low |
| Quality gain | Largest on tasks with objectively checkable failure modes (code correctness, factual grounding, format compliance); smaller on purely subjective quality |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Critic and generator share the same blind spot | Same model, same training biases | Use a different model or a very different prompt framing for the critic role |
| Infinite revision without convergence | Critique keeps finding new issues each round | Hard round cap; return best-effort with a "needs human review" flag past the cap |
| Critique is vague, not actionable | Poor critic prompt | Require the critique to list concrete, specific problems, not general impressions |
| Over-correction | Revision overreacts to critique, introduces new errors | Ask the reviser to make the minimal change that addresses the critique |

---

## Problems using this pattern
- [[agents/scenarios/coding-agent-design]]
- Content generation with correctness or compliance requirements
- Any task where evaluation criteria are checkable but generation alone is unreliable

---

## Sources
- [[agents/concepts/reasoning-and-planning]]
- [[agents/concepts/agent-evaluation]]
- [[agents/concepts/multi-agent-orchestration]]
