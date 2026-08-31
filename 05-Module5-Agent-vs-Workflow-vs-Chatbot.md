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

Not every LLM-powered feature should be an agent. This might feel like a strange thing for an "Agentic AI" course to emphasize so early, but it's one of the most important professional judgments you'll develop throughout this material: many production problems are better solved by a simple workflow, or even just a chatbot, and reaching for the most powerful, most autonomous pattern by default is a common and expensive mistake. Choosing the simplest architecture that solves the problem reliably is a core engineering skill — agents add power but also add unpredictability, cost, and failure modes, and every one of those costs is real and has to be paid for, in engineering time, in dollars, and in the number of ways the system can go wrong in production.

To understand why this trade-off exists, it helps to think about what you're actually buying when you move from a chatbot, to a workflow, to a full agent. A **chatbot** buys you natural-language understanding for a single exchange — nothing more. A **workflow** buys you a fixed, repeatable, testable sequence of steps, some of which may use an LLM's language understanding, but the sequence itself is decided once by a human at design time and never changes at runtime. An **agent** buys you the ability for the system to *decide its own sequence of steps*, adapting to what it discovers along the way — but this adaptability is exactly what makes an agent harder to test (you can't simply assert "step 2 always follows step 1," because it might not), harder to predict (the same input can, in principle, take a different path to a similar answer, as covered in Module 19's discussion of non-determinism), and more expensive to run (more LLM calls deciding *what* to do, on top of the calls that actually do it). None of this makes agents bad — it makes them a tool suited to a specific kind of problem: one where the actual sequence of right steps *genuinely cannot be known in advance*, because it depends on information only available once execution has started.

This is the single sharpest diagnostic question you should ask before building anything: **can a human, sitting down right now with full knowledge of the business rules, write out the exact fixed sequence of steps that should happen for every case?** If yes — even if that sequence includes an LLM doing something clever at one particular step, like classifying an email or extracting a date — you almost certainly want a workflow, not an agent, because you get all the language-understanding benefit of the LLM with none of the unpredictability cost of letting the system also decide *when* and *whether* to take each step. If no — if the right next step truly depends on unpredictable intermediate results, and no fixed sequence could cover every case — that's the signal an agent's adaptability is actually buying you something a workflow structurally cannot provide.

### A Common Question

**"Isn't a workflow with an LLM step basically an agent already, since it's 'using AI'?"** No — and this question points at exactly the confusion Module 4 addressed head-on: "uses an LLM" and "is agentic" are independent properties, not the same thing. A workflow can use an LLM for one or several of its steps (classifying, summarizing, extracting) while the overall *sequence* of steps remains entirely fixed by the developer — the LLM is doing sophisticated work *within* one step, but it never gets to decide "should I even run step 3, or should I add a new step 2.5 I wasn't asked to consider?" That decision-making-about-the-sequence-itself is specifically what Module 4 identified as the defining trait of an agent, and a workflow, by construction, never has it.

**"If agents are riskier and harder to test, why would anyone choose one?"** Because for a genuine subset of real problems, a fixed sequence simply cannot cover the space of things that might need to happen. Consider "investigate why our AWS bill increased 40% this month" — there is no fixed sequence of steps that would work identically well whether the cause turns out to be a runaway compute instance, a forgotten storage bucket, or a pricing tier change, because which of those it is (and therefore what to investigate next) is only discoverable *during* the investigation. Forcing this into a workflow would mean either building an enormous fixed decision tree trying to anticphonecipate every possible cause in advance (expensive to build, and still incomplete), or accepting that the workflow will only handle the causes its designer happened to think of. An agent's ability to look at what it's found so far and decide what to check next is precisely what a fixed workflow cannot replicate without becoming, in effect, a hand-coded imitation of an agent's decision loop — at which point you've just built a worse, less flexible agent by hand.

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

Read this table row by row as a series of trade-offs, not a scoreboard where "Agent" is trying to win every category — notice that Workflow explicitly beats Agent on "Failure mode risk," and that's a real, important advantage of workflows you should weigh seriously, not a limitation to apologize for. The "Best when..." row is the one worth internalizing above all the others: it's the practical decision rule the rest of this lesson exists to teach you to apply.

### Real-World Examples

- **Chatbot**: A website widget that answers "what are your store hours?" — no action taken, no memory of state beyond the current chat. Notice this chatbot could in principle be asked a follow-up like "and what about weather delays?" — it would try its best, but it has no mechanism to go check a live weather feed or update its own understanding; it can only draw on whatever was in its training data or the immediate prompt.
- **Workflow**: An onboarding pipeline: "extract data from resume → classify seniority level → route to the right recruiter" — an LLM might power the classification step, but the *sequence* is fixed by a human regardless of the outcome. Even if the classification step returns a surprising result (say, the resume is for a completely different role than expected), the workflow still proceeds through exactly the same three steps in exactly the same order — it will route an unusual result to *a* recruiter, just possibly the wrong one, because "should I actually change my plan given this surprising classification?" was never a decision the system was built to make.
- **AI Agent**: A "trip planner" that must decide, based on what it finds (flight prices, availability, weather), whether to search more, adjust dates, or finalize — the actual sequence of steps depends on what it discovers. If flights are unexpectedly expensive on the requested dates, the agent can decide, on its own, to check nearby dates instead — a workflow with a fixed "search flights → book flights" sequence has no equivalent branch to fall into; it would just report the expensive flights, or fail outright if no flights matched a hardcoded price filter.

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

Look closely at the arrow annotations under each pattern, because they're marking the exact place each architecture's flexibility (or lack of it) lives. The Chatbot line has no branching at all — it's a straight, repeatable line with no memory of previous state feeding forward. The Workflow line is also straight, but the annotation flags that an LLM can live *inside* one of the fixed boxes without changing the fact that the boxes themselves, and their order, never move — this is the exact configuration described in the onboarding-pipeline example above. The Agent line is the only one drawn with a repeating bracketed unit and an ellipsis (`...`), because the *number* of loops and *which* actions happen inside each one are not fixed in advance — they're determined at runtime, which is precisely why an agent trace can legitimately look different across two runs of the "same" task, in a way that would be considered a bug in a workflow.

### Key Takeaways
- A chatbot answers; a workflow executes a fixed recipe; an agent decides the recipe as it goes.
- More autonomy is not automatically "better" — it trades predictability for flexibility, and that trade is only worth making when the problem genuinely can't be solved with a predetermined sequence.
- Many real business problems are solved perfectly well by workflows with an LLM step or two — reach for a full agent only when the actual path forward genuinely can't be predetermined.
- The diagnostic test worth applying every time: could a knowledgeable human write out the fixed sequence of steps in advance for every case? If yes, build a workflow (possibly with LLM-powered steps). If genuinely no, that's your signal an agent's runtime decision-making is buying you something real.

### Common Mistakes
- Building a full autonomous agent for a task that's really just a fixed 3-step pipeline — adds cost, latency, and unpredictability with no benefit. The underlying mechanism of this mistake: every extra LLM call an agent makes to *decide* what to do next is a call a workflow simply doesn't need to make at all, because the decision was already made once, by a human, at design time — so for a task with a genuinely fixed sequence, an agent is strictly paying for reasoning it doesn't need.
- Building a rigid workflow for a task where the right next step truly depends on unpredictable intermediate results — the workflow will need constant special-casing and still won't handle novel cases. The underlying mechanism here is the mirror image of the first mistake: every new case the workflow's designer didn't anticipate in advance becomes either a silent misfire (routed through steps that don't actually fit) or a maintenance burden (another `if` branch bolted onto an already-growing decision tree that's slowly reinventing, badly, the adaptive behavior an agent would have provided natively.
- Treating "does it use an LLM" as the deciding question instead of "does it decide its own sequence of steps." This is the single most common confusion this module exists to correct — plenty of workflows use an LLM heavily (for classification, summarization, extraction) without being remotely agentic, and conflating the two leads teams to either over-build (adding agent-style autonomy to something that never needed it) or mis-market a fixed pipeline as having flexibility it structurally does not have.

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
4. Why does an agent's ability to adapt its sequence at runtime make it inherently harder to test than a workflow?
