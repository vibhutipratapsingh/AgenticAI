# Module 5 — Agent vs. Workflow vs. Chatbot

### Difficulty
Beginner

### Learning Objectives
- Clearly differentiate chatbots, workflows, and agents on concrete dimensions.
- Learn to choose the right pattern for a given problem instead of defaulting to "build an agent."

### Prerequisites
Module 4.

---

## Lesson 5.1 — The Comparison

### Concept Explanation

Not every LLM-powered feature should be an agent. Many production problems are better solved by a simple workflow, or even just a chatbot. Choosing the simplest architecture that solves the problem reliably is a core engineering skill — agents add power but also add unpredictability, cost, and failure modes.

### Comparison Table

| Feature | Chatbot | Workflow | AI Agent |
|---|---|---|---|
| **Decision making** | None beyond generating a reply | None — steps are pre-defined by a developer | Decides its own next steps dynamically |
| **Tool usage** | Usually none (or one fixed tool call per turn) | Fixed, pre-wired tool calls in a set order | Chooses which tools to use and when |
| **Adaptability** | Low — responds to each message independently | Low — same sequence every run regardless of intermediate results | High — adjusts plan based on observations |
| **Autonomy** | None (waits for next human message each turn) | None (executes a script; a human or trigger starts it) | Can run multiple steps unattended toward a goal |
| **Planning** | No planning; single-turn response | Implicit plan baked in by the developer at design time | Plans dynamically at runtime, can replan |
| **Example use cases** | FAQ bot, casual conversation, writing assistant for one-off replies | "New signup → send welcome email → add to CRM → notify sales" (fixed pipeline, LLM might classify one step) | "Research this market and produce a competitive analysis report" |
| **Failure mode risk** | Low (each turn is isolated) | Low-medium (predictable, easy to test since steps are fixed) | Higher (more autonomy = more ways to go wrong; needs guardrails) |
| **Best when...** | The task is a single conversational exchange | The steps are well-known and don't change per case | The steps required genuinely depend on what's discovered along the way |

### Real-World Examples

- **Chatbot**: A website widget that answers "what are your store hours?" — no action taken, no memory of state beyond the current chat.
- **Workflow**: An onboarding pipeline: "extract data from resume → classify seniority level → route to the right recruiter" — an LLM might power the classification step, but the *sequence* is fixed by a human regardless of the outcome.
- **AI Agent**: A "trip planner" that must decide, based on what it finds (flight prices, availability, weather), whether to search more, adjust dates, or finalize — the actual sequence of steps depends on what it discovers.

### Visual Diagram

```text
CHATBOT:
User → LLM → Response  (repeat per turn, no state changes to the world)

WORKFLOW:
Trigger → Step 1 (fixed) → Step 2 (fixed) → Step 3 (fixed) → Done
              ↑ an LLM MAY power one or more steps, but order never changes

AGENT:
Goal → [Think → Act → Observe] → [Think → Act → Observe] → ... → Done
              ↑ the agent itself decides how many loops, and which actions
```

### Key Takeaways
- A chatbot answers; a workflow executes a fixed recipe; an agent decides the recipe as it goes.
- More autonomy is not automatically "better" — it trades predictability for flexibility.
- Many real business problems are solved perfectly well by workflows with an LLM step or two — reach for a full agent only when the actual path forward genuinely can't be predetermined.

### Common Mistakes
- Building a full autonomous agent for a task that's really just a fixed 3-step pipeline — adds cost, latency, and unpredictability with no benefit.
- Building a rigid workflow for a task where the right next step truly depends on unpredictable intermediate results — the workflow will need constant special-casing and still won't handle novel cases.

### Exercise
For each scenario, classify it as best solved by a chatbot, a workflow, or an agent, and justify in one sentence:
1. Answering "What's your return policy?" on an e-commerce site.
2. Given a spreadsheet of expenses, categorize each row and produce a monthly summary.
3. Given "reduce our AWS bill," investigate current usage, identify waste, and propose specific changes.

### Challenge
Take one process at your job that is currently manual. Decide whether it should be automated as a workflow or an agent, and explain the deciding factor (predictability of steps vs. need for judgment/adaptation).

### Knowledge Check
1. What's the deciding factor between choosing a workflow vs. an agent for a new feature?
2. Why might a team deliberately choose *not* to build a full agent even when it's technically possible?
3. Give an example of an LLM-powered feature that is not an agent.
