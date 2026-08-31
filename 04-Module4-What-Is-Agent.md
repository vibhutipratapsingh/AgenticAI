# Module 4 — What Is an AI Agent?

### Difficulty
Beginner

### Learning Objectives
- Define what makes a system "agentic."
- Understand the agent loop: Goal → Observe → Think → Plan → Act → Check → Continue/Finish.
- Understand goals, decisions, actions, tools, environment, and feedback loops.

### Prerequisites
Modules 1–3.

---

## Lesson 4.1 — Defining an Agent

### Concept Explanation

An **AI agent** is a system built around an LLM that:
1. Is given a **goal** (not a single fixed instruction for one output).
2. **Decides** what steps are needed to reach that goal.
3. Can use **tools** to take actions in the world (search, calculate, read/write files, call APIs).
4. **Observes** the results of its actions.
5. **Continues or adjusts** its approach until the goal is achieved (or it gives up / asks for help).

The defining trait is the **loop**: think → act → observe → think again, repeated until done — not a single input/output pass.

It's worth slowing down on why this definition is drawn exactly here, because "agent" is one of the most overused and inconsistently defined words in this entire field. Every module up to this point has dealt with a single LLM call: you send a prompt (Module 2-3), the model generates one response, and the interaction is over. That's true no matter how clever the prompt is, how much few-shot context you include, or how carefully you've engineered it — it's fundamentally a *request-response* pattern, the same shape as a vending machine transaction (Module 1), just with a vastly more sophisticated "machine" inside. An agent is the first thing in this course that breaks that request-response shape. Instead of "one prompt in, one response out," an agent's defining structural feature is that the *model's own output feeds back into its own next input* — it is called again and again, each time seeing what happened as a result of its last decision, until it decides (or is forced) to stop.

This has a very concrete practical consequence that's easy to miss: an agent is not "a smarter chatbot" — it is a different *control structure* wrapped around the same underlying LLM technology you learned about in Module 2. The LLM itself hasn't changed at all between a single Q&A call and an agent using that same model in a loop. What's changed is the *code around* the model: instead of calling the model once and returning its answer to the user, the surrounding program repeatedly calls the model, executes whatever action the model requested, and feeds the result of that action back in as new input for the model's *next* call. This distinction — same brain, different control loop — is the single most important idea in this module, and it's why Module 6 will show you that an "agent" is mostly just an ordinary `while` loop (Module 0.3) wrapped around an LLM call plus some tool-execution code.

### A Common Question

**"If I give a single LLM call a really long, detailed prompt describing multiple steps, isn't that basically an agent already?"** No — and the distinction matters enough to spell out precisely. A single call, no matter how elaborate the prompt, still only gets to observe the world *once*, at the moment you constructed that prompt, and it produces its entire response in one shot with no opportunity to see whether its first sub-step actually worked before committing to the rest. If you ask a single call to "search for flights, then based on what you find, book the cheapest one, then email me a confirmation," the model has to *hallucinate* what the search results probably look like, because it has no actual tool execution and no real observation step — it's writing fiction about what a plausible search result and a plausible booking might be, not actually doing either. An agent, by contrast, genuinely pauses after the search step, waits for a real tool to return real data, and only then decides what a "cheapest flight" among the *actual* results is. The loop isn't a stylistic choice — it's what allows the system to react to ground truth instead of guessing at it in advance.

**"Does 'agent' require full autonomy — no human involved at all?"** No, and this is a common misconception this course corrects early and returns to often (especially in Module 18, Human-in-the-Loop). An agent can, and in production systems very often should, pause partway through its loop and wait for a human decision before continuing — that pause is still part of the agent's control flow, not a departure from "being an agent." What makes something agentic is the goal-driven, self-directed loop of deciding, acting, and observing — not the total absence of any human checkpoint. A fully autonomous system with zero human oversight and a well-designed agent with strategic approval gates are both agents; they just sit at different points on a risk-management spectrum you'll study in depth in Module 18.

### Simple Analogy

> A chatbot is like someone answering questions while sitting at a desk — one question in, one answer out, then they wait for the next question. If you ask them to "figure out the best restaurant nearby and make a reservation," they might describe *how* they'd do that, but they can't actually walk outside, check which restaurants are open, or place a phone call — they're confined to the desk, confined to language, confined to one exchange at a time.
>
> An AI agent is like an employee who receives a goal ("organize the Q3 marketing campaign"), figures out what tasks that involves, uses the tools available to them (email, spreadsheets, the internet), checks their own work, and keeps going — reporting back only once the goal is actually accomplished (or they hit a blocker worth escalating). Crucially, this employee doesn't wait at their desk for the next instruction after every micro-step; they decide for themselves what the next step should be, based on what they've learned so far, and only come back to you when the whole thing is done or when something requires your input. That self-directed continuation, across many steps, without needing a human to explicitly trigger each one, is exactly the behavior the agent loop below is modeling in code.

### The Agent Loop

```mermaid
flowchart TD
    Goal([Goal]) --> Observe[Observe:<br/>What do I know?<br/>What just happened?]
    Observe --> Think[Think:<br/>What does this mean?<br/>What's missing?]
    Think --> Plan[Plan:<br/>What's the next step?]
    Plan --> Act[Act:<br/>Use a tool / take an action]
    Act --> Check{Check Result:<br/>Did it work? Closer to goal?}
    Check -- Not done yet --> Observe
    Check -- Goal met --> Finish([Finish:<br/>Report result])

    style Goal fill:#e0e7ff,stroke:#4338ca
    style Finish fill:#dcfce7,stroke:#16a34a
    style Check fill:#fef3c7,stroke:#d97706
```

**How to read this graph:** this is a loop, not a straight line — notice the arrow from the "Check Result" diamond back up to "Observe." That loop-back is the single most important feature of this diagram: it's what lets an agent try again, gather more information, or take a different action when its first attempt doesn't fully solve the goal. A regular program (or a chatbot) would just stop after one pass through the top-to-bottom boxes; an agent keeps circling through Observe → Think → Plan → Act → Check until the diamond finally routes it out to "Finish." The yellow diamond is the agent's one and only exit decision point — everything before it can repeat indefinitely (up to a safety limit, covered in Module 6 and Module 17), but this is the only place the agent decides "am I done or not."

One more thing worth noticing about this diagram: the *shape* of the loop never changes, no matter what task the agent is solving. Whether the goal is "find the best laptop" or "plan a trip to Japan" or "draft and send a status email," the exact same five boxes execute in the exact same order, over and over. What varies from task to task isn't the control structure — it's the *content* of each box: which tool gets called in "Act," what counts as "the goal being met" in "Check Result." This is exactly why the agent loop is worth learning as its own abstract concept, separate from any one example — once you internalize this shape, you'll recognize it underneath every agent you build or read about for the rest of this course, including the far more elaborate reasoning patterns in Module 12 and the multi-agent systems in Module 14-15, which are, at their core, this same loop composed and nested in more sophisticated ways.

### Explaining Each Stage

- **Goal**: the outcome the agent is working toward, defined by the user (e.g., "find the best laptop under ₹80,000"), not a single Q&A turn. The goal is set once, at the start, and stays fixed as the reference point the agent measures its own progress against for the entire run — it's the one thing in the loop that (usually) doesn't change between iterations.
- **Observe**: gather current state — what has been done so far, what information is available, what the last tool result was. On the very first pass through the loop, "observe" mostly just means noticing the goal itself and the fact that nothing has happened yet; on later passes, it means specifically looking at the most recent tool result and reviewing the history of everything done before it.
- **Think**: the LLM reasons about what the observation means relative to the goal (internally — not necessarily shown to the user in raw form). This is where the model asks itself, in effect, "given what I now know, am I closer to done, and what's still missing?" — the connective reasoning step between raw observation and a concrete next plan.
- **Plan**: decide the next concrete step (which tool to use, what sub-task to tackle). This is a narrower decision than "Think" — thinking is open-ended reasoning about the situation, while planning narrows that down to one specific, executable next move.
- **Act**: execute that step — usually a tool call. This is the only stage in the loop that actually reaches outside the model itself and touches the real world (an API, a file, a calculation) — everything else is the model reasoning about text.
- **Check Result**: evaluate whether the action succeeded and moved the agent closer to the goal. This stage matters more than it might first appear — a tool call can technically "succeed" (return data with no error) while still not actually helping toward the goal (e.g., a search that returned irrelevant results), and a well-designed Check step needs to distinguish "did the tool run without crashing" from "did this actually help."
- **Continue/Finish**: loop back if the goal isn't met yet, or produce the final answer/deliverable if it is. This is the diamond in the diagram above — the one moment where the loop's shape genuinely branches rather than proceeding straight through.

### Real-World Example

**Goal:** "Find the best laptop under ₹80,000."

```text
Plan: Search for laptops under ₹80,000.
Action: search_tool("best laptops under 80000 INR 2026")
Observation: Found 12 candidate laptops with specs and prices.

Plan: Narrow down by comparing specs (RAM, processor, reviews).
Action: compare_tool(candidates)
Observation: 3 laptops stand out on specs and reviews.

Plan: Check current price and availability.
Action: price_check_tool(these 3 laptops)
Observation: One is out of stock; two remain in budget.

Decision: Recommend the two remaining, explain trade-offs.
Finish: Present recommendation with reasoning.
```

Trace this against the six-stage diagram above and you'll see each named stage happening, even though the trace doesn't label them explicitly: each "Plan" line is the Think+Plan stages compressed into one summary sentence; each "Action" line is Act; each "Observation" line is the result that gets fed into the *next* iteration's Observe stage. Notice, too, that the plan genuinely changes shape at each step based on what was actually observed — the second plan ("narrow down by comparing specs") only makes sense *because* the first search returned 12 candidates rather than, say, zero or one. If the first search had come back empty, the correct next plan would have been something entirely different (broaden the search terms, or ask the user to relax the budget), which is exactly the kind of adaptive behavior a single, non-looping LLM call structurally cannot produce, because it never gets to see real search results before committing to its full answer.

Note: this shows **concise decision summaries and observations**, not hidden internal chain-of-thought — this is the standard, safe way to demonstrate agent reasoning. This is a deliberate documentation and product-design choice you'll see maintained throughout this entire course: an agent's raw internal reasoning (the literal token-by-token "thinking" the model does) is not something you should show to end users or bake into your logs verbatim — instead, you engineer the agent to produce short, human-readable summaries of its plan and findings at each step, the way the trace above does. This keeps the agent's behavior auditable and understandable without exposing an unfiltered, potentially confusing or sensitive stream of raw model reasoning.

### What Happens Without a Stopping Condition?

The trace above happens to finish cleanly after four steps, but it's worth asking: what if it hadn't? What if `search_tool` kept returning slightly different, unhelpful results, and the model kept deciding "let me search again with different terms" indefinitely? This is not a hypothetical edge case — it's one of the most common real failure modes in agent systems, and it's directly connected to the "Check" diamond in the loop diagram above. If "Check Result" never confidently routes to "Finish," and there's no independent safeguard, the loop can genuinely run forever, calling tools, burning API cost, and never producing an answer. This course introduces the fix for this — a hard maximum number of loop iterations, checked by the surrounding code rather than trusted to the model's own judgment — properly in Module 6.2's code example, and treats it as a first-class reliability concern in Module 17.1. For now, the important takeaway is structural: the loop diagram's "Continue" arrow is powerful specifically because it can repeat indefinitely, which means something *outside* the model itself must always be able to force a stop.

### Visual Diagram — What Makes It "Agentic"

```text
                     Is there a GOAL (not just one fixed instruction)?
                                    │
                     ┌──────────────┴──────────────┐
                    No                             Yes
                     │                              │
              Not agentic                  Does it DECIDE its own steps?
      (e.g., a fixed "summarize"                    │
             button)                  ┌──────────────┴──────────────┐
                                      No                             Yes
                                       │                              │
                                Still just a               Does it use TOOLS and
                                workflow/LLM call            OBSERVE results in a loop?
                                                                       │
                                                        ┌──────────────┴──────────────┐
                                                       No                             Yes
                                                        │                              │
                                                 A single-shot                    ✅ This is
                                                 LLM response                     an AI Agent
```

Treat this as a diagnostic flowchart you can literally run any real system through when someone tells you "we built an AI agent." Walk a "smart" auto-reply email feature through it: is there a goal, or a single fixed instruction ("draft a reply")? A single fixed instruction — so it exits at the first "No," and by this course's definition, it is not an agent, no matter how good the LLM behind it is. Now walk an automated research assistant through it: goal ("investigate X and report back") — yes. Decides its own steps (chooses what to search, in what order, based on what it finds) — yes. Uses tools and observes results in a loop — yes. It reaches "✅ This is an AI Agent" only after passing all three gates, and this module's whole point is that all three gates matter — a system that passes two out of three (say, it has a goal and uses tools, but always executes the exact same fixed tool sequence regardless of what it observes) is a **workflow**, not an agent, which is precisely the distinction Module 5 develops in full.

### Key Takeaways
- An agent has a goal, not just an instruction.
- An agent decides its own steps rather than following a hardcoded sequence.
- Tools + observation + iteration is what separates an agent from a single LLM call.
- The loop continues until the goal is met or the agent determines it needs to stop/escalate.
- Structurally, an agent is the *same* underlying LLM as a single Q&A call — what's different is the surrounding control loop that repeatedly re-invokes the model with fresh observations, rather than calling it exactly once.

### Common Mistakes
- Calling anything that uses an LLM "an agent" — a single Q&A call is not agentic; the defining feature is the goal-driven loop. This mistake matters beyond pedantry: if you build a single-call feature but market/design it as though it has agentic capabilities (adaptivity, multi-step follow-through), users and stakeholders will expect behavior the system structurally cannot provide, leading to confusing failures that look like "the AI is broken" when really the architecture was mismatched to the task from the start.
- Believing agents must run forever without stopping conditions — a well-designed agent has clear termination conditions (goal met, max steps reached, or needs human input). The root cause of this mistake is usually treating the "Continue" arrow in the loop diagram as automatically safe just because it's part of the intended design — but an arrow that *can* loop is not the same as an arrow that *always terminates*, and only explicit, code-enforced limits guarantee the latter (Module 6, Module 17).
- Assuming more loop iterations always means better answers. In practice, an agent that's genuinely stuck (repeating a failing action, or unable to make progress toward an ambiguous goal) doesn't improve with more time in the loop — it just accumulates cost and, sometimes, drifts further from a sensible answer as increasingly irrelevant history piles up in its context window (Module 2.2's "context rot," revisited in Module 17). Recognizing when to stop and escalate is as important a skill to design for as recognizing when to keep going.

### Exercise
Take a task you do at work or home (e.g., "plan weekly groceries"). Write it as a goal, then sketch 4–5 steps an agent might take to accomplish it, in the Plan → Act → Observe format.

### Challenge
Describe a task that *looks* like it needs an agent but could actually be solved with a simple fixed workflow (no autonomous decision-making needed). Explain why the simpler approach is sufficient — this judgment call is a skill you'll need throughout the course.

### Knowledge Check
1. What is the single defining difference between a chatbot and an agent?
2. Name the six stages of the agent loop in order.
3. Why is "checking the result" a critical step, not an optional one?
4. Why can a single, very detailed LLM prompt never fully substitute for a real agent loop, even for a multi-step task?
