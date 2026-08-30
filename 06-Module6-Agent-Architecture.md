# Module 6 — Basic Agent Architecture

### Difficulty
Beginner → Intermediate

### Learning Objectives
- Understand the components of an agent: brain, instructions, tools, state, memory, environment, execution loop.
- Walk through a conceptual agent solving a real goal.

### Prerequisites
Modules 1–5.

---

## Lesson 6.1 — The Components of an Agent

### Concept Explanation

```text
User Goal
    ↓
Agent
 ┌────────────────────────┐
 │      LLM / Brain       │  ← reasons about what to do next
 └────────────────────────┘
    ↓
Decision                      ← "I need to search for X" / "I have enough info to answer"
    ↓
Tool Selection                ← which tool fits this decision
    ↓
Tool Execution                ← the tool actually runs (API call, calculation, file read...)
    ↓
Observation                   ← the tool's result is fed back in
    ↓
Next Decision (loop back to Brain)
```

- **Agent brain**: the LLM itself — reasons over the current state and decides the next action.
- **Instructions**: the system prompt defining the agent's role, rules, and boundaries (Module 3).
- **Tools**: functions/APIs the agent can call to affect or query the outside world (Module 7).
- **State**: the current snapshot of what's happened so far in this run (steps taken, results so far).
- **Memory**: information retained across turns or sessions — short-term (this conversation) or long-term (across sessions) (Module 8).
- **Environment**: the world the agent operates in — a filesystem, an API, a database, a web browser.
- **Execution loop**: the code that repeatedly calls the brain, executes the chosen tool, and feeds results back, until a stopping condition is met.

### Simple Analogy

> Think of a restaurant kitchen: the **head chef** (LLM/brain) decides what to cook next based on orders (goal) and what's already been prepared (state/memory); the **kitchen tools** (knives, oven, tools) let them actually act on ingredients (environment); and the **line** of kitchen staff repeating "check order → cook → plate → check next order" is the execution loop.

---

## Lesson 6.2 — Walking Through a Conceptual Agent

### Concept Explanation

**Goal:** "Find the best laptop under ₹80,000."

The agent should:
1. Understand the request (budget constraint, category = laptop, criteria = "best").
2. Search available information.
3. Compare options.
4. Evaluate specifications against likely priorities (performance, battery, reviews).
5. Select suitable choices.
6. Explain the recommendation.

### Visual Walkthrough (Decision Summaries, Not Hidden Reasoning)

```text
GOAL: Find the best laptop under ₹80,000.

Step 1
  Plan: Understand what "best" likely means without more input — assume general
        use: good performance, reliable brand, strong reviews, since no specific
        use case was given.
  Action: search_tool("best laptops under 80000 rupees 2026 reviews")
  Observation: 12 candidate laptops retrieved with prices and rough specs.

Step 2
  Plan: Filter to laptops actually under budget with recent, credible reviews.
  Action: filter_tool(candidates, max_price=80000, min_review_score=4.0)
  Observation: 5 candidates remain.

Step 3
  Plan: Compare the 5 remaining on CPU, RAM, battery life, and build quality.
  Action: compare_tool(remaining_5)
  Observation: 2 laptops clearly outperform the others on this use case.

Step 4
  Plan: Verify current stock/price hasn't changed since search.
  Action: price_check_tool(top_2)
  Observation: Both still in stock and within budget.

Step 5 — Finish
  Decision: Recommend both, with a clear primary pick and reasoning.
  Final Answer: "Laptop A is the best overall pick for battery life and build
  quality; Laptop B is a strong budget-performance alternative if raw speed
  matters more to you."
```

This is how agent reasoning should be *shown* to users and in documentation: concise plans, explicit tool actions, observations, and a final decision — never raw private chain-of-thought.

### Practical Example (Simplified Python Skeleton)

```python
def run_agent(goal, tools, llm, max_steps=10):
    state = {"goal": goal, "history": []}

    for step in range(max_steps):
        # 1. Brain decides next action based on goal + history so far
        decision = llm.decide_next_step(state)

        if decision["action"] == "finish":
            return decision["final_answer"]

        # 2. Execute chosen tool
        tool = tools[decision["tool_name"]]
        result = tool.run(decision["tool_input"])

        # 3. Record the observation and loop
        state["history"].append({
            "plan": decision["plan_summary"],
            "tool": decision["tool_name"],
            "result": result,
        })

    return "Stopped: reached max steps without finishing."
```

*Explanation:* `llm.decide_next_step` is where the brain reasons over the goal and history to choose an action (a real implementation returns structured JSON — see Module 7). The loop caps at `max_steps` to prevent runaway execution (Module 17 covers this reliability concern in depth). Each iteration appends to `history`, which becomes the agent's working memory for this run (Module 8).

### Key Takeaways
- An agent = Brain (LLM) + Instructions + Tools + State/Memory + Environment + Execution Loop.
- The execution loop is ordinary code — the "intelligence" lives in the brain's decisions, not magic in the loop itself.
- Always show reasoning as concise plans/observations, never exposed raw internal chain-of-thought.

### Common Mistakes
- Forgetting a maximum step limit — an agent without a stopping condition can loop indefinitely, burning cost (Module 17).
- Not logging each step's plan/action/observation — without this, debugging why an agent failed becomes nearly impossible.

### Exercise
Design (on paper) the step-by-step plan/action/observation trace for an agent with the goal: "Book the cheapest available flight from Delhi to Mumbai next Friday." Include at least 4 steps.

### Challenge
Identify one point in your Exercise trace where the agent's plan should change based on what it observes (e.g., "no flights found for Friday, check Saturday instead"). Explain how the execution loop needs to support that kind of replanning.

### Knowledge Check
1. Name the six components of a basic agent architecture.
2. Why does the execution loop need a maximum step count?
3. What's the difference between "state" and "memory" in an agent (hint: think about a single run vs. across runs — this will be sharpened in Module 8).
