# ReAct Pattern (Reason + Act)

**Topic:** [[agents/topics/agent-architectures]]
**Related concepts:** [[agents/concepts/agent-loop]], [[agents/concepts/reasoning-and-planning]], [[agents/concepts/tool-calling]]

---

## What it solves

A model that jumps straight to an action without reasoning first tends to call the wrong tool or misinterpret the goal; a model that only reasons (chain-of-thought) without grounding in real observations tends to hallucinate facts it can't actually verify. ReAct interleaves the two: **Thought → Action → Observation**, repeated, so each reasoning step is grounded in the latest real information from the environment.

---

## Template / Skeleton

```python
def react_agent(goal: str, tools: dict, max_steps: int = 10) -> str:
    history = [f"Goal: {goal}"]

    for step in range(max_steps):
        # 1. THINK — model reasons over goal + history so far
        thought = llm.generate(prompt=build_prompt(history))
        history.append(f"Thought: {thought}")

        # 2. ACT — model decides on a tool call or a final answer
        action = parse_action(thought)
        if action.type == "final_answer":
            return action.content

        # 3. EXECUTE — orchestrator runs the tool, not the model
        try:
            result = tools[action.tool_name](**action.arguments)
            observation = str(result)
        except Exception as e:
            observation = f"Error: {e}"

        # 4. OBSERVE — result appended to history for the next iteration
        history.append(f"Action: {action.tool_name}({action.arguments})")
        history.append(f"Observation: {observation}")

    return "Max steps reached without a final answer."
```

### Prompt scaffold (what the model actually sees)
```
You have access to these tools: search(query), calculator(expr), get_weather(city)

Use this format:
Thought: <reasoning about what to do next>
Action: <tool_name>(<arguments>)
Observation: <will be filled in for you>
... (repeat Thought/Action/Observation as needed)
Thought: I now have enough information to answer.
Action: Final Answer(<answer>)

Goal: What's the weather in the capital of France?
```

---

## Worked Example

```
Thought: I need to know the capital of France first.
Action: search("capital of France")
Observation: "Paris is the capital of France."

Thought: Now I need the weather in Paris.
Action: get_weather("Paris")
Observation: "18°C, cloudy"

Thought: I have both pieces of information needed.
Action: Final Answer("The weather in Paris, the capital of France, is 18°C and cloudy.")
```

---

## Signal Phrases

- "Design an agent that answers questions using external tools/search"
- "The agent needs to reason about which tool to use at each step"
- "We need the agent's decisions to be groundable/inspectable, not a black box"
- "Build a general-purpose tool-using assistant"

---

## Complexity

| Aspect | Notes |
|---|---|
| Latency | One model call per Thought+Action step; scales with number of steps taken |
| Cost | Grows with steps × (accumulated history length) unless context is managed — see [[agents/concepts/context-engineering]] |
| Predictability | Medium — the model chooses the path, so step count and tool sequence vary by input |
| Debuggability | High — the Thought trace makes the model's reasoning inspectable at every step, a major advantage over opaque single-shot generation |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Infinite/repeating loop | Model retries the same failing action | Loop detection, max steps — see [[agents/concepts/agent-loop]] |
| Wrong tool selected | Ambiguous tool descriptions | Sharpen tool descriptions/scope — see [[agents/topics/tool-use]] |
| Reasoning drifts from the goal | Long history dilutes the original goal's salience | Periodically re-state the goal in the prompt, summarize history |
| Hallucinated Thought not grounded in Observation | Model "assumes" a result instead of waiting for the actual tool output | Enforce strict Thought/Action/Observation format parsing; reject malformed steps |

---

## Problems using this pattern
- [[agents/scenarios/research-agent-design]]
- [[agents/scenarios/customer-support-agent-design]]
- General-purpose tool-using assistants

---

## Sources
- [[agents/concepts/agent-loop]]
- [[agents/concepts/reasoning-and-planning]]
- [[agents/concepts/tool-calling]]
