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

**Goal decomposition** is the process of breaking a large, ambiguous, high-level goal into a set of smaller, concrete, individually actionable subtasks. **Planning** is what an agent does to produce that breakdown — deciding both *what* the subtasks are and *what order* to tackle them in — before or during execution.

To understand why this matters, it helps to see what goes wrong without it. Recall from Module 6 that an agent's "brain" is an LLM making one decision at a time inside a loop: observe, think, decide the next action, act. If you hand that loop a goal like "plan a trip to Japan" with no decomposition step, the LLM has to simultaneously juggle every dimension of the problem — dates, budget, flights, hotels, itinerary, total cost — while also deciding which single next action to take. That's a lot of competing considerations to hold in one reasoning pass, and it tends to produce agents that either freeze (ask the user twelve clarifying questions before doing anything), wander (start booking a flight before dates are even fixed), or silently drop requirements (forget to check budget until the very end, after money has already been "spent" in the plan). Decomposition solves this by front-loading the *structural* thinking — "what are the pieces of this problem?" — into a distinct step, so that the execution loop afterward only ever has to solve one well-defined subtask at a time. This is the same reason a human project manager writes a task list before diving into work: it converts one overwhelming, vague ball of requirements into a sequence of small, checkable, individually low-risk actions.

It's worth being precise about what makes a subtask "concrete enough." A decomposition like "1. Research the trip, 2. Book the trip, 3. Go on the trip" is technically a breakdown, but each item is still too vague to act on directly — "research the trip" doesn't tell the agent (or a human) what tool to call or what question to answer first. A well-formed subtask should be small enough that a single tool call, or a very short sequence of them, could plausibly complete it, and specific enough that "done" is unambiguous. "Find flights from Delhi to Tokyo for the chosen dates" is checkable: either the agent found flight options or it didn't. "Research the trip" is not checkable in the same way — there's no clear finish line. Good planning is largely the skill of finding the right *grain size*: not so coarse that steps are unactionable, not so fine that the plan itself becomes a bigger reasoning burden than the task it's organizing.

### A Common Question

**"Isn't this just the agent loop from Module 6 again, with extra words?"** Not quite — they solve different problems and normally work together. The agent loop (Observe → Think → Plan → Act → Check → Continue) describes how an agent makes *one* decision at a time, moment to moment. Planning, as covered in this module, is about producing the *shape of the whole task* before diving into that step-by-step execution — it's a higher-altitude view. In practice, an agent typically runs one planning step to get an initial task list (Lesson 11.1), and then runs the ordinary agent loop, one iteration per subtask, to actually execute that list (with the possibility of revising the list along the way, which is Lesson 11.2). Think of the agent loop as the *engine* and planning as the *map* the engine consults to decide which road to drive down next.

**"Who actually writes the plan — is this a separate model call?"** In most implementations, yes: a plan is typically produced by a dedicated prompt to the LLM, something like *"Given this goal, break it into an ordered list of concrete, actionable subtasks. Return the list as JSON."* This is a structured-output call in the same style as Module 2.5 and Module 7 — the LLM's job here isn't to execute anything, only to produce the task list as data. That data (the plan) then becomes part of the agent's state (Module 16), consulted and updated as execution proceeds.

### Simple Analogy

> Handing an agent a big goal without decomposition is like telling an intern "organize the company retreat" with no further breakdown. A good planner turns that into a checklist: pick dates, set budget, choose venue, book travel, plan activities, communicate with attendees — each one concrete enough to actually start on. Notice that the checklist doesn't just list random related words — it puts them in a sensible order (you can't book travel before you have dates, and you can't set a realistic budget before you know roughly how many people are attending). A useful plan captures both *what* needs doing and *which things depend on which other things*.

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

Notice the ordering isn't arbitrary. Dates come first because almost everything else depends on them (flight prices and hotel availability both change with dates). Budget comes second, before flights and hotels are actually searched, so the agent has a ceiling to filter against instead of finding great options it can't afford and only realizing that at the end. "Estimate total cost" comes last because it needs the *actual* numbers from flights and hotels, not the earlier budget guess — this is a subtle but important distinction: step 2 ("check budget") is about a constraint the plan should respect, while step 6 ("estimate total cost") is about a result the plan produces. Confusing these two is a common source of bad plans — an agent that treats its budget check as the final answer, instead of as a constraint to plan around, will happily "succeed" at a plan that turns out to be unaffordable.

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

**How to read this diagram:** the single GOAL box at the top splits into multiple Subtask boxes below it — this is the decomposition itself, rendered visually. But don't read the three arrows as meaning the subtasks are independent or can run in any order or all at once; in the Japan example above, they're drawn side-by-side purely for layout, while the actual plan is a strict *sequence* (1 then 2 then 3...). A more complete diagram for a real plan would also need arrows *between* the subtask boxes showing dependencies — which is exactly what Lesson 11.2's Planning Loop diagram adds once we introduce execution order and revision.

---

## Lesson 11.2 — Dynamic Planning and Replanning

### Concept Explanation

Real-world execution rarely goes exactly as planned. A plan is, by necessity, written *before* the agent has gathered all the information execution will reveal — it's a best guess based on what's known at planning time. **Dynamic planning** means the agent doesn't treat that initial guess as gospel: it revisits and adjusts the plan as new information (tool results, failures, changed constraints) comes in during execution. **Replanning** is the concrete mechanism for this — explicitly regenerating part or all of the plan when reality diverges from what the plan assumed.

It helps to think about *why* rigid, unchangeable plans fail so often in agentic systems, specifically. A plan is built on assumptions — "there will be a direct flight on the exact date I want," "the API I need will respond successfully," "the data will be in the format I expect." Every one of those assumptions can turn out to be false once the agent actually acts (Module 6's "Act" and "Check Result" stages), and an agent that mechanically executes step 4 of its plan even after step 3 revealed the underlying premise was wrong will produce nonsensical or actively harmful results — booking hotels for dates that no longer have flights, for instance. This is the planning equivalent of the reliability problems in Module 17: a rigid plan is *brittle* in exactly the way an agent without error handling is brittle. The fix in both cases is the same philosophy — detect that something didn't go as expected, and have an explicit, designed response instead of blindly continuing.

The harder engineering question is not "should the agent replan" but "*when* should it replan, and *how much* of the plan should it throw away." Not every surprising observation should trigger a full replan — that would make the agent thrash (spend all its effort regenerating plans and never actually executing any of them), which is one of the Common Mistakes below. The right threshold is: **does this new information invalidate an assumption a later, not-yet-executed step depends on?** If the agent searches for a specific hotel and finds it's fully booked, but the plan's next step ("find hotels") hasn't committed to that specific hotel yet, there's nothing to replan — the agent just tries a different hotel within the same subtask. But if the agent discovers there are no flights at all on the planned dates, every subsequent step that assumed those dates (hotel search windows, the itinerary, the final cost estimate) is now built on a false premise, and a real replan is warranted. A useful mental test: **"if I kept executing the rest of the plan unchanged, would the result still make sense given what I just learned?"** If no, replan; if yes, just note the observation and continue.

It's also worth being precise about *targeted* vs. *full* replanning, since jumping straight to "regenerate the whole plan from scratch" is wasteful and can even undo good decisions already locked in (a chosen budget, for instance, usually doesn't need to be revisited just because the travel dates shifted by two days). Good replanning logic identifies which downstream steps are actually affected by the new information and regenerates only those, leaving unaffected steps (and any work already completed for them) untouched. In the Japan example below, only steps 1 and 3 onward change — the fact that a budget of ₹1,50,000 was set in step 2 doesn't need to be re-derived from scratch just because the departure date moved by two days; a well-designed replanning step would re-*check* that the budget still holds for the new dates, not blindly throw the old number away and start over.

### A Common Question

**"How does the agent actually decide when to replan, in code — is this a special model call, or a hardcoded rule?"** It's usually a mix. A simple, reliable approach is a hardcoded rule for "obviously plan-breaking" observations — for instance, a tool explicitly returning an error, or a returned value falling outside an expected range (no results found, a required field is null). For subtler cases — "is this new fact actually a problem for the rest of my plan?" — many production agents make a small, targeted LLM call whose only job is to answer that yes/no question given the current plan and the new observation, separate from the call that actually does the replanning. Splitting "should I replan?" from "what should the new plan be?" into two distinct decisions tends to produce more reliable behavior than asking one big prompt to do both at once, for the same reason splitting any complex judgment into smaller sub-judgments usually helps (this echoes the separation-of-concerns idea behind the Critique & Revise pattern in Module 12.4).

**"What stops an agent from replanning forever, chasing every small surprise?"** This is exactly the "thrashing" failure mode flagged in Common Mistakes below, and the fix is the same discipline used everywhere else reliability matters in this course (Module 17): put a hard limit on how many times the agent is allowed to replan for a single goal, and if that limit is hit, stop and either present the best plan so far or escalate to a human (Module 18) rather than looping indefinitely.

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

Walking through why this is a *targeted* replan rather than a full restart: the trigger is the observation in step 3 ("no direct flights on Oct 10"), which directly invalidates the dates chosen in step 1. Everything downstream of dates — hotels, itinerary, cost — is potentially affected too, since hotel prices and availability are also date-dependent. But notice step 2 ("check budget → ₹1,50,000") is *not* regenerated from scratch; it's carried forward and merely re-verified ("re-check budget impact of shifted dates"), because a two-day shift in travel dates doesn't invalidate the underlying premise that ₹1,50,000 is the available budget — it only means that number needs to be re-checked against the *new* costs, not re-derived.

**A messier example, to show replanning isn't always this clean.** Suppose the agent gets one step further before hitting trouble:

```text
Plan after first replan (dates now Oct 12-19):
3. Find flights → search_flights(Delhi→Tokyo, Oct 12)   ✅ found, ₹42,000 round trip
4. Find hotels → search_hotels(Tokyo, Oct 12-19)          ✅ found, ₹8,000/night average
5. Create itinerary → build_itinerary(Tokyo, 7 days)
   Observation: itinerary tool returns a 7-day plan that includes visiting Mount Fuji,
   but a side note flags that early access tickets for that date range are already
   sold out for the specific weekend the itinerary scheduled the visit.

SECOND REPLAN TRIGGERED (partial, mid-itinerary):
   The agent does NOT restart from step 1 — dates and flights are already booked-in-plan
   and shouldn't be casually discarded (re-searching flights costs tool calls and could
   even return a worse price than the one already found). Instead:
   5a. Re-run itinerary generation with a constraint: move the Mount Fuji day to a
       different day within the existing Oct 12-19 window, or substitute an alternative
       activity if no day works.
   6. Estimate total cost (unchanged — flights and hotels already confirmed)
```

This second example illustrates a case where the *right-sized* replan is much smaller than the first one — only sub-step 5 needs regenerating, constrained to stay within dates that are already locked in from earlier steps. An agent that treated every new piece of information as a reason to re-run steps 1 through 6 from scratch would waste several already-good tool calls (and, worse, risk landing on a *different*, possibly worse, flight price the second time around) to fix a problem that only actually lived inside step 5.

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

**How to read this diagram:** the decision diamond ("Does result contradict/invalidate the current plan?") is the entire discipline of this lesson compressed into one branch point — everything above it is ordinary step-by-step execution, and the "Yes" branch is the only place a replan happens. Notice the diagram loops back around regardless of which branch is taken; the difference is only *what* the agent does before looping — quietly move to the next planned step, or first patch the plan and *then* move to the (possibly different) next step. If you're picturing this next to the agent loop diagram from Module 6, this Planning Loop is best understood as sitting one level above it: each pass through "Execute Next Step → Observe Result" here can itself be a full Observe → Think → Plan → Act → Check cycle from Module 6, applied to just that one subtask.

### Key Takeaways
- Planning turns a vague goal into a concrete, ordered sequence of actionable subtasks — the right grain size is small enough to act on directly, but not so fine that managing the plan itself becomes the bottleneck.
- Plans should be treated as living documents, not fixed scripts — dynamic replanning is what lets agents handle the real world's unpredictability, in the same way error handling (Module 17) lets execution survive unpredictable failures.
- Replanning should be targeted (regenerate only the steps whose underlying assumptions were actually invalidated) rather than always restarting from scratch — this preserves already-good work and avoids wasted tool calls or worse outcomes on re-attempted steps.
- A practical test for "should I replan?" is: if I kept executing the rest of the plan unchanged, would the result still make sense given what I just learned? If no, replan; if yes, continue.

### Common Mistakes
- **Generating an overly detailed plan upfront and rigidly executing it even when early steps reveal the plan is now wrong.** This happens when a system treats the plan as a fixed script to be run top-to-bottom rather than as a working hypothesis to be checked against reality — the fix is to always run each step's result through a "does this still make sense?" check, not just execute the next line unconditionally.
- **Replanning on every single minor observation, causing thrashing and never making forward progress.** This is the overcorrection in the opposite direction — usually caused by not having a clear threshold for "does this actually invalidate a downstream assumption," so *any* surprising fact triggers a full regeneration. Reserve replanning for genuinely plan-invalidating information, and use the "would the rest of the plan still make sense unchanged?" test to decide.
- **Failing to persist the plan as state**, making it hard for the agent (or the developer debugging it) to see what was decided and why. Without this, a crash or restart mid-task loses not just progress but the *reasoning* behind the choices already made — ties directly to Module 16's State Management, which is what makes a plan recoverable rather than something that only exists transiently inside one LLM call's context.
- **Regenerating the entire plan from scratch on every replan instead of a targeted patch.** Beyond wasting tool calls (as in the messier Mount Fuji example above), a full regeneration can silently produce a *worse* plan than the one being replaced, since the new plan is generated without the benefit of decisions and confirmed results the old plan had already locked in.

### Exercise
Take the goal "Launch a small online store for handmade candles." Write an initial 5–7 step plan.

### Challenge
Introduce one realistic complication partway through your plan from the Exercise (e.g., "the chosen payment provider doesn't support your country"). Show how the plan should be revised — which steps change, which stay the same, and use the "would the rest of the plan still make sense unchanged?" test explicitly to justify your answer.

### Knowledge Check
1. What is goal decomposition, and why is it necessary before execution?
2. What triggers a replan, and why shouldn't every minor observation trigger one?
3. Why is treating a plan as a "living document" better than treating it as a fixed script?
4. What test can you apply to decide whether a new observation warrants a full replan, a targeted (partial) replan, or no replan at all?

Continue to **[12-Module12-Reasoning-Patterns.md](12-Module12-Reasoning-Patterns.md)** to learn concrete reasoning loop patterns (ReAct, plan-and-execute, reflection) that implement this kind of planning in practice.
