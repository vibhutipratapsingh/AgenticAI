# Module 22 — Cost Optimization

### Difficulty
Advanced

### Learning Objectives
- Understand token costs and how model choice affects them.
- Learn caching, smaller models, tool optimization, context optimization, and batch operations.

### Prerequisites
Modules 1–21.

---

## Lesson 22.1 — Token Costs and Model Selection

### Concept Explanation

LLM providers charge per token, usually with separate (and different) rates for input tokens vs. output tokens (Module 2.1). Larger, more capable models cost more per token than smaller models. Every extra tool call, every long document stuffed into context, and every unnecessary reasoning round in a multi-agent system directly adds to cost.

### Simple Analogy

> Using the largest, most capable model for every single step of an agent's work is like hiring your most senior (and most expensive) engineer to also handle basic data entry. Match the "seniority" (capability, and cost) of the model to the actual difficulty of each step.

### Practical Example — Model Tiering

```text
Task: Classify a support ticket into one of 5 categories
   → Use a smaller/cheaper, faster model (simple classification task)

Task: Draft a nuanced response to an angry enterprise customer
   → Use the larger, more capable model (requires nuance, tone control)

Task: Decide which tool to call next in a simple agent loop
   → Use a smaller model with clear tool descriptions and few-shot examples
```

---

## Lesson 22.2 — Caching

### Concept Explanation

**Caching** avoids paying for the same computation twice. Two common forms in LLM systems:
- **Prompt/context caching**: providers can cache large, repeated portions of a prompt (like a long system prompt or document) so subsequent calls reusing that same prefix are cheaper and faster.
- **Response caching**: if the same (or very similar) query is likely to recur, cache the final answer and serve it directly instead of re-calling the LLM.

### Visual Diagram

```text
Call 1: [Long System Prompt + Doc] + "Question A" → LLM (full cost)
                     ↓ (system prompt + doc cached)
Call 2: [Cached System Prompt + Doc] + "Question B" → LLM (much cheaper — only
                                                         "Question B" is new)
```

---

## Lesson 22.3 — Tool and Context Optimization

### Concept Explanation

- **Tool optimization**: minimize unnecessary tool calls (e.g., don't re-fetch data the agent already has in its state); batch related tool calls where possible instead of calling one at a time.
- **Context optimization**: apply the same principles from Module 2.2 and Module 10 (RAG) — retrieve only what's relevant instead of dumping entire documents; summarize/compress older conversation history instead of keeping it verbatim forever.

### Practical Example

```python
# BAD: re-fetches the same weather data on every reasoning step
for step in agent_steps:
    weather = get_weather("Pune")  # wasteful repeated tool call
    ...

# GOOD: fetch once, reuse from state
if "weather" not in state:
    state["weather"] = get_weather("Pune")
weather = state["weather"]
```

---

## Lesson 22.4 — Batch Operations

### Concept Explanation

**Batch operations** process many items together in fewer, larger calls instead of one call per item — reducing overhead and often qualifying for cheaper batch-processing pricing offered by some providers for non-time-sensitive workloads.

### Practical Example

```text
BAD (100 separate calls):
for email in 100_emails:
    classify(email)   # 100 separate LLM calls

GOOD (batched):
classify_batch(100_emails)   # fewer calls, or one batch job, processing all
                              # 100 together — cheaper and often faster overall
```

### Key Takeaways
- Cost scales with token volume, model tier, and number of LLM calls — every one of these is a lever you can pull.
- Match model capability to task difficulty instead of defaulting to the most powerful (and expensive) model everywhere.
- Caching, context optimization, and batching are complementary — apply all three where applicable rather than picking just one.

### Common Mistakes
- Defaulting to the largest available model for every single agent step "to be safe," multiplying cost with little quality benefit for simple sub-tasks.
- Re-sending large static context (system prompts, reference documents) on every call without leveraging prompt caching where the provider supports it.
- Not monitoring cost per task type in production (Module 19–20) — cost problems often go unnoticed until the bill arrives.

### Exercise
For a 3-agent content pipeline (Research → Write → Edit from Module 14), assign an appropriate model tier (small/medium/large) to each agent's task and justify your choice.

### Challenge
Design a cost-monitoring dashboard (list the fields/breakdowns) that would let a team see cost per agent, per model tier, and per task type — enough detail to identify exactly where money is being spent inefficiently.

### Knowledge Check
1. Why does model tiering (matching model size to task difficulty) reduce cost without necessarily reducing quality?
2. What's the difference between prompt caching and response caching?
3. Give one concrete example of a wasteful, avoidable repeated tool call.
