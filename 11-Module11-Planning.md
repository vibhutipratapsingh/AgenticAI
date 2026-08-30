# Module 11 — Planning

### Difficulty
Intermediate

### Learning Objectives
- Understand goal decomposition and task planning.
- Understand dynamic planning and replanning when new information appears.

### Prerequisites
Modules 1–10.

---

## Lesson 11.1 — Goal Decomposition and Task Planning

### Concept Explanation

**Goal decomposition** is breaking a large, ambiguous goal into smaller, concrete, actionable subtasks. **Planning** is the process of an agent producing this breakdown (and an order to tackle it) before or during execution.

### Simple Analogy

> Handing an agent a big goal without decomposition is like telling an intern "organize the company retreat" with no further breakdown. A good planner turns that into a checklist: pick dates, set budget, choose venue, book travel, plan activities, communicate with attendees — each one concrete enough to actually start on.

### Example

**Goal:** "Plan a trip to Japan."

```text
Agent Plan:
1. Determine travel dates
2. Check budget
3. Find flights
4. Find hotels
5. Create itinerary
6. Estimate total cost
```

### Visual Diagram

```text
                        GOAL
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
     Subtask 1       Subtask 2       Subtask 3
          │               │               │
     (concrete,      (concrete,      (concrete,
      actionable)      actionable)     actionable)
```

---

## Lesson 11.2 — Dynamic Planning and Replanning

### Concept Explanation

Real-world execution rarely goes exactly as planned. **Dynamic planning** means the agent doesn't commit to a rigid plan upfront — it revisits and adjusts the plan as new information (tool results, failures, changed constraints) comes in. **Replanning** is explicitly regenerating part or all of the plan when reality diverges from what was expected.

### Practical Example

```text
Initial Plan:
1. Determine travel dates → Oct 10-17
2. Check budget → ₹1,50,000
3. Find flights → search_flights(Delhi→Tokyo, Oct 10)
   Observation: No direct flights available on Oct 10; cheapest option is Oct 12.

REPLAN TRIGGERED:
   New Plan:
   1. Determine travel dates → Oct 12-19 (shifted due to flight availability)
   2. Re-check budget impact of shifted dates
   3. Continue: Find flights → search_flights(Delhi→Tokyo, Oct 12)
   4. Find hotels for adjusted dates
   5. Create itinerary
   6. Estimate total cost
```

### Visual Diagram — Planning Loop

```text
Goal
 ↓
Generate Initial Plan
 ↓
Execute Next Step
 ↓
Observe Result
 ↓
Does result contradict/invalidate the current plan?
 ├─ No  → continue to next planned step
 └─ Yes → Replan (regenerate affected portion of the plan)
 ↓
Repeat until goal is met
```

### Key Takeaways
- Planning turns a vague goal into a concrete, ordered sequence of actionable subtasks.
- Plans should be treated as living documents, not fixed scripts — dynamic replanning is what lets agents handle the real world's unpredictability.
- Replanning should be targeted (adjust the affected steps) rather than always restarting from scratch, for efficiency.

### Common Mistakes
- Generating an overly detailed plan upfront and rigidly executing it even when early steps reveal the plan is now wrong — wastes effort and produces bad outcomes.
- Replanning on every single minor observation, causing thrashing and never making forward progress — reserve replanning for genuinely plan-invalidating information.
- Failing to persist the plan as state, making it hard for the agent (or the developer debugging it) to see what was decided and why (ties to Module 16 — State Management).

### Exercise
Take the goal "Launch a small online store for handmade candles." Write an initial 5–7 step plan.

### Challenge
Introduce one realistic complication partway through your plan from the Exercise (e.g., "the chosen payment provider doesn't support your country"). Show how the plan should be revised — which steps change, which stay the same.

### Knowledge Check
1. What is goal decomposition, and why is it necessary before execution?
2. What triggers a replan, and why shouldn't every minor observation trigger one?
3. Why is treating a plan as a "living document" better than treating it as a fixed script?

Continue to **[12-Module12-Reasoning-Patterns.md](12-Module12-Reasoning-Patterns.md)** to learn concrete reasoning loop patterns (ReAct, plan-and-execute, reflection) that implement this kind of planning in practice.
