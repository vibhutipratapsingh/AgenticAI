# Part 13 — Hands-On Projects

Five progressively harder projects, each applying concepts from the modules above. Build them in order.

Why this order, specifically? Each project is deliberately scoped so that its "new" difficulty comes from exactly one or two fresh concepts, while everything else reuses a pattern you already built. Project 1 has no tool calling and no multi-step loop at all — it's pure structured-output plus persistence, so you can get comfortable with the LLM-as-a-function idea in isolation. Project 2 adds the agent loop and tool calling on top of that. Project 3 swaps tool calling out for retrieval, so you can focus entirely on embeddings and RAG without also juggling a reasoning loop. Project 4 then recombines loop + tools + retrieval + planning + reflection into one system — which is only tractable because you've already debugged each piece separately in isolation. Project 5 is the first time you split one agent's responsibilities across several communicating agents. If you skip ahead to Project 4 or 5 without building 1–3 first, you'll be debugging three unfamiliar mechanisms simultaneously, which is exactly the situation this staged order is designed to prevent.

---

## Project 1 (Beginner) — Personal Task Assistant

### Features
- Accept tasks in natural language ("remind me to email the client tomorrow").
- Store tasks persistently.
- Prioritize tasks (e.g., by urgency/deadline).
- Answer questions about current tasks ("what's due today?").

### Modules Used
Module 2 (structured output), Module 3 (prompting), Module 6 (basic agent architecture), Module 8 (memory basics), Module 16 (persistent state).

### Implementation Steps

**1. Define the data model.**
```python
# task schema
{"id": str, "description": str, "due_date": str | None, "priority": "low"|"medium"|"high", "done": bool}
```

Why this step comes first, before any LLM call: every other piece of this project — parsing, storage, prioritization, and Q&A — needs to agree on exactly one shape for "what a task is." If you start writing the parsing prompt before you've pinned down the schema, you'll end up improvising fields as you go, and by the time you write the prioritization function you'll discover half your stored tasks are missing a field the new code assumes exists. Notice `due_date: str | None` — it's explicitly optional, because a huge fraction of real task descriptions ("remind me to call mom") have no date at all, and treating that as an error instead of a valid, common case is one of the most frequent beginner mistakes in this kind of project (more on this below).

**2. Build a "parse task" LLM call** that turns free text into the structured schema:
```python
def parse_task(user_text: str) -> dict:
    prompt = f"""Extract a task from this text. Respond as JSON matching:
    {{"description": str, "due_date": "YYYY-MM-DD" or null, "priority": "low"|"medium"|"high"}}
    Text: {user_text}"""
    return json.loads(ask_llm(prompt, temperature=0))
```

Walking through why this is written the way it is: `temperature=0` (Module 2.4) is deliberate, not incidental — this call's whole job is turning messy human phrasing into one exact, parseable shape, and you want the *least* creative, most consistent behavior the model can give you, not variety. The prompt explicitly states the JSON schema inline, including the literal string `"YYYY-MM-DD" or null` — spelling out the null case in the prompt itself is what stops the model from inventing a placeholder date (like today's date, or a made-up one) when the user didn't actually give one. `json.loads(...)` (Module 0.6) is what turns the model's raw text response into an actual Python dict your code can index into — skip this and you'd just have a string that looks like JSON but that Python can't use as a dict. In production you'd wrap this `json.loads` call in a `try/except`, because an LLM occasionally emits text that isn't valid JSON (a stray sentence before the braces, a trailing comma) — Module 17's validation guidance applies directly here even in a "beginner" project.

**3. Persist tasks** to a simple database (SQLite is fine for a personal project).

This is the step that turns "a task the program currently has open" into "a task that still exists tomorrow." Without this step, every task would live only in a Python variable in memory — the instant the program exits (or crashes), everything is gone. This is the smallest possible instance of the session-state-vs-persistent-state distinction from Module 16.2: your in-memory `parse_task` output is session state; writing it into SQLite is what promotes it to persistent state. SQLite specifically is the right choice here (not Postgres, not a cloud database) because a personal task assistant has exactly one user and doesn't need concurrent-write support or a network round-trip — reaching for a heavier database here would be the Module 5 mistake of over-engineering a simple problem.

**4. Build a prioritization function** — sort by due date proximity and priority level; this can be plain code (no LLM needed — a good example of Module 5's "don't over-agent-ify" principle).

It's worth pausing on *why* this specific step should NOT call an LLM, since it's tempting to think "everything AI-adjacent should go through the model." Sorting by due date and priority is a fully deterministic operation — given the same list of tasks, there is exactly one correct sorted order, and a plain Python `sorted()` call computes it instantly, for free, with zero chance of error. Routing this through an LLM would add latency, cost, and — critically — the possibility that the model sorts inconsistently between two runs on the identical input, which a human user would rightly experience as a bug. This is precisely the workflow-vs-agent judgment call from Module 5: reach for the LLM only for the steps that genuinely require language understanding (parsing free text, answering open-ended questions), and use plain code for anything with one unambiguous right answer.

**5. Build a Q&A function** that retrieves relevant tasks (filter by date/status) and asks the LLM to answer naturally:
```python
def answer_question(question: str, tasks: list[dict]) -> str:
    context = json.dumps(tasks)
    return ask_llm(f"Given these tasks: {context}\n\nAnswer: {question}")
```

Notice this is a miniature RAG pattern (Module 10), even though it's far simpler than a real vector-search pipeline: you're retrieving relevant data (here, just "all tasks," since the dataset is tiny) and injecting it as context before asking the model to answer, rather than letting the model try to answer from nothing. `json.dumps(tasks)` (Module 0.6) turns your Python list of dicts back into a JSON string, because the prompt is plain text and can't contain a live Python object — this is the mirror image of the `json.loads` used in Step 2. For a personal task list with, say, under a few hundred tasks, dumping everything into context like this is perfectly fine; the moment your dataset grows large enough that "all tasks" no longer fits comfortably in the context window (Module 2.2), you'd need to filter to just the relevant subset first (e.g., "tasks due this week") before handing them to the model — which is the seed of the retrieval-then-generate idea that Project 3 builds out properly with real embeddings.

### Handling a Realistic Complication: Ambiguous Due Dates

Real users write things like "remind me to email the client soon" or "sometime next week" — phrasing with no crisp date at all. If `parse_task`'s prompt only ever asks for a hard `YYYY-MM-DD` or `null`, the model is forced into one of two bad behaviors: either it guesses a specific date that wasn't actually said (a small hallucination, Module 17.1), or it returns `null` and silently discards the "soon"/"next week" urgency signal entirely. A better design adds a third field, `date_confidence: "exact"|"approximate"|"none"`, and when confidence is `"approximate"`, has the assistant surface that ambiguity back to the user ("I'll treat this as due sometime next week — let me know an exact day if you have one") rather than pretending certainty it doesn't have. This is a small, concrete instance of the "instruct the model to say it doesn't know rather than guess" principle that reappears at much higher stakes in Module 10 (RAG) and Module 17 (hallucination mitigation).

### Stretch Goals
- Add natural-language task editing ("push the client email to Friday").
- Add daily summary generation.

---

## Project 2 (Beginner/Intermediate) — Research Agent

### Capabilities

```text
User Question
      ↓
Research Agent
      ↓
Search Tool
      ↓
Collect Information
      ↓
Summarize
      ↓
Final Report
```

### Modules Used
Module 6 (agent architecture), Module 7 (tool calling), Module 11–12 (planning, ReAct pattern).

### Implementation Steps

**1. Define a search tool** (a real web-search API, or a mocked one for practice).

Start with the tool, not the agent loop around it, because a tool with a badly-written description (Module 7.2) will sabotage every downstream step no matter how well the loop is built — if the model can't tell from the tool's description when and how to call it correctly, it'll either never use it or misuse it. Building and manually testing the tool function in isolation first (call it directly with a few example queries and eyeball the output) means that when something goes wrong later in the full agent, you can rule out "the tool itself is broken" and focus your debugging on the loop logic instead.

**2. Implement a ReAct loop** (Module 12.1): the agent alternates between deciding to search and synthesizing what it's found, until it has enough information.

This is the step where the project stops being "one LLM call" and becomes genuinely agentic in the Module 4 sense — there's now a goal (the user's question) that the code doesn't know in advance how many search calls it will take to satisfy, and the loop itself decides that dynamically, one step at a time, exactly as described in Module 12.1's ReAct pattern.

**3. Track sources** alongside collected information, so the final report can cite them.

This step is easy to skip and easy to regret skipping. If you only store the *text* the search tool returned and discard *where* it came from, the final report has no way to attribute its claims — which means a user (or you, debugging later) has no way to check whether a given sentence is well-supported or a hallucinated addition the synthesis step introduced. Tracking `{"query": ..., "result": ...}` pairs (as the code below does) rather than just a flat list of result text is what makes citations possible at all in Step 4.

**4. Generate the final report** — a structured summary with sections and citations, not just a single paragraph.

```python
def research_agent(question: str, max_steps=5) -> dict:
    findings = []
    for step in range(max_steps):
        decision = llm_decide_next_action(question, findings)
        if decision["action"] == "finish":
            break
        result = search_tool(decision["query"])
        findings.append({"query": decision["query"], "result": result})

    report = ask_llm(f"Write a report answering '{question}' using these findings:\n{findings}")
    return {"report": report, "sources": [f["query"] for f in findings]}
```

Walking through this line by line: `findings = []` is the agent's working memory for this run (Module 8) — everything the agent has learned so far in this specific execution, which starts empty and grows with every loop iteration. The `for step in range(max_steps)` loop, rather than a bare `while True`, is the max-step guardrail from Module 6 and Module 17 baked directly into the code — without it, a model that never decides `"finish"` (because the question is unanswerable, or because of a bug in the decision prompt) would search forever, burning cost with no way to stop. Inside the loop, `llm_decide_next_action(question, findings)` is the "Brain" from Module 6.1 — it looks at the goal and everything gathered so far, and returns a structured decision about what to do next; note it's given `findings`, not just `question`, precisely because a good next search query almost always depends on what earlier searches already turned up (e.g., don't search for the same thing twice). The `if decision["action"] == "finish": break` line is the agent's own exit condition — the loop can end early, before `max_steps` is reached, whenever the model itself judges it has enough information, which is what makes this efficient rather than always doing exactly `max_steps` searches regardless of need. Finally, notice the report-generation call happens *after* the loop entirely, as one separate LLM call over the *complete* findings list — this separation (gather everything first, synthesize once at the end) tends to produce a more coherent report than trying to write incrementally after every single search result.

### Stretch Goals
- Add a reflection step (Module 12.3) that checks whether the report actually answers the original question before finalizing.

---

## Project 3 (Intermediate) — RAG Knowledge Assistant

### Capabilities
- Upload documents.
- Process (chunk) and embed documents.
- Store embeddings in a vector database.
- Search knowledge semantically.
- Answer questions, grounded in the uploaded documents.

### Modules Used
Module 9 (embeddings/vector DB), Module 10 (RAG).

### Implementation Steps

**1. Ingestion pipeline**: chunk uploaded documents (Module 9.3), embed each chunk, store in a vector DB with metadata (source filename, section).

This step happens entirely separately from, and before, any user ever asks a question — it's the "ahead of time" half of the two-pipeline RAG diagram from Module 10.2. Skipping straight to building the query pipeline without first getting ingestion right is a common ordering mistake: if your chunks are badly sized, or your metadata is missing, every single query later will silently retrieve mediocre context, and it's much harder to diagnose "why are the answers bad?" after the fact than it is to sanity-check your ingestion output directly (e.g., print out 5 stored chunks and read them) before building anything on top of it.

**2. Query pipeline**: embed the user's question, retrieve top-k relevant chunks, inject as context, instruct the model to answer only from context (Module 10.2).

The phrase "answer only from context" is doing real work here, not just decoration — without that explicit instruction in the prompt, the model is free to blend in whatever it happens to remember from training alongside what was actually retrieved, and the user (and you) would have no way to tell which parts of the answer are actually grounded in your uploaded documents versus quietly drawn from the model's general knowledge. This is the single most important sentence in the whole project for keeping the assistant trustworthy.

**3. Add citations**: return which document/section each part of the answer came from.

**4. Handle "not found" gracefully**: if no chunk is sufficiently relevant, respond "I don't have that information" rather than guessing.

Steps 3 and 4 are two sides of the same coin: a RAG system's entire value proposition over a raw LLM is that its claims are *checkable* against real source material, and both steps exist to preserve that checkability. Step 3 lets a user verify a *present* answer; Step 4 stops the assistant from confidently fabricating an answer when the *right* answer is "I don't know" — which, without an explicit instruction and a genuine relevance check, an LLM will very rarely say on its own, since generating *some* plausible-sounding text is usually easier for the model than recognizing and admitting the gap.

### Handling a Realistic Complication: No Chunk Is Relevant Enough

Every top-k similarity search (Module 9.2) returns *something* — even when none of your documents actually address the question, the search will still hand back whichever stored chunks happen to be least-dissimilar, because "find the k nearest vectors" has no built-in concept of "good enough." If your code blindly injects those weak matches into the prompt and asks the model to answer, you'll get a confidently-worded answer built on marginally-related material — arguably worse than no answer at all, because it *looks* grounded but isn't really. The fix is to check the actual similarity scores (Module 9.2's bar-chart example) against a minimum threshold before injecting anything: if the top result's cosine similarity is, say, below 0.5, treat that as "nothing relevant found" and trigger the graceful "I don't have that information" response from Step 4 instead of proceeding as if a good match existed. Picking that threshold number is itself a judgment call you'll tune empirically against your own documents and query patterns — there's no universal "correct" cutoff.

### Stretch Goals
- Add hybrid search and reranking (Module 10.3) for better precision.
- Add metadata filtering (e.g., only search documents uploaded by the current user).

---

## Project 4 (Advanced) — Autonomous Research System

### Capabilities
- Planning (Module 11).
- Multi-step research with tool calling (Module 7, Project 2).
- Memory across the whole task (Module 8, 16).
- Reflection before finalizing (Module 12.3).
- Final report generation with structured sections.

### Implementation Steps

**1. Generate an upfront plan** for the research question (plan-and-execute, Module 12.2).

This is a deliberate departure from Project 2's pure ReAct loop, and the reason is scale: Project 2's questions were simple enough that deciding "one step at a time" never wasted much effort, but a genuinely open-ended research question (Module 11.1's "plan a trip to Japan"-style decomposition, applied to research instead) benefits from seeing the whole shape of the problem before diving in, so the system can allocate its limited step budget sensibly across sub-questions rather than wandering into whichever thread the very first search happens to surface.

**2. Execute each planned sub-task**, using the ReAct loop from Project 2 for each one, saving results into persistent state after each step (checkpointing, Module 16.3).

Notice this step explicitly reuses Project 2's ReAct loop rather than inventing a new execution mechanism — this is the payoff of having built that loop once, carefully, already: here it's just called once per planned sub-task instead of once for the whole question. The "saving results into persistent state after each step" clause is not optional polish; a genuinely long-running autonomous research task (potentially many minutes and dozens of tool calls) is exactly the scenario Module 16.3 describes where a crash partway through, without checkpointing, means losing all completed work and starting over from sub-task one.

**3. Replan** if a sub-task reveals the original plan needs adjusting (Module 11.2).

**4. Reflect**: before producing the final report, have the agent critique its own draft against the original question and revise gaps.

Steps 3 and 4 add the two "self-correction" capabilities that distinguish this project from a more rigid pipeline: replanning (Module 11.2) handles the case where the *plan itself* turns out to be wrong mid-execution (a sub-task reveals information that invalidates a later planned step), while reflection (Module 12.3) handles the case where the plan was fine but the *final synthesis* has gaps or errors that only become visible once you compare the draft against the original question as a whole. These operate at different points in the process and catch different classes of mistakes — building only one of them leaves a real gap in the system's ability to self-correct.

**5. Produce the final report**, with sources and a summary of the plan actually followed (transparency for the user).

Including "the plan actually followed" (not just the final answer) matters because replanning in Step 3 means the executed plan may differ from the original — showing the user what actually happened, not just a polished final answer, is what lets them sanity-check the system's process, catch a bad replanning decision, and build calibrated trust in the tool over time, rather than treating its output as an unquestionable black box.

### Handling a Realistic Complication: A Sub-Task's Findings Contradict Each Other

Real research frequently turns up genuinely conflicting information — two sources disagreeing on a statistic, or a fact that turns out to have changed over time. A naive reflection step might just pick one source and present it confidently, silently discarding the disagreement (a subtle form of hallucination — not inventing new information, but overstating certainty about existing information). A better design has the reflection step explicitly check for this: if two retained findings conflict, the final report should surface the disagreement directly ("Source A reports X; Source B reports Y — this may reflect a recent change or differing methodology") rather than resolving it invisibly. This connects directly to the Debate pattern from Module 15, Pattern 5 — for a high-stakes version of this system, you could route genuinely conflicting findings through an actual debate-and-judge step rather than handling it inline in the reflection prompt.

### Stretch Goals
- Add a hard step/cost budget with graceful early termination and a partial report if the budget runs out (Module 17, 22).
- Add human-in-the-loop plan approval before execution begins (Module 18).

---

## Project 5 (Advanced) — Multi-Agent Content Company

### Agents
- Research Agent
- Content Writer
- SEO Specialist
- Editor
- Quality Reviewer

### Modules Used
Module 14–15 (multi-agent architecture and patterns).

### Architecture Diagram

```text
                    Supervisor Agent
                          │
     ┌─────────┬──────────┼───────────┬────────────┐
     ▼         ▼          ▼           ▼            ▼
 Research   Writer      SEO         Editor      Quality
  Agent     Agent      Specialist   Agent       Reviewer
     │         │          │           │            │
     └─────────┴──────────┴───────────┴────────────┘
                          │
                          ▼
                   (loop back to Writer if Quality
                    Reviewer rejects — Critic pattern,
                    Module 15, Pattern 6)
                          │
                          ▼
                    Final Article
```

### Implementation Notes
- Use a **Pipeline pattern** (Module 15, Pattern 4) for the main Research → Write → SEO → Edit flow.
- Add a **Critic pattern** (Module 15, Pattern 6) loop between Editor and Quality Reviewer, with a max of 3 revision rounds before escalating to a human (Module 18).
- Use **shared structured state** (not raw text dumps) between agents — e.g., a `sources` list, a `seo_keywords` list, a `flagged_issues` list — so each agent gets exactly what it needs (Module 14.3).

Why this project combines two different coordination patterns (Pipeline *and* Critic) instead of picking just one: the main flow — research feeds writing, writing feeds SEO, SEO feeds editing — genuinely is linear and well-understood, which is exactly Module 15 Pattern 4's sweet spot, so forcing it through a more complex pattern would add coordination overhead for no benefit. But the Editor-to-Quality-Reviewer relationship is fundamentally different: it's not "hand off and move on," it's "produce, critique, revise, possibly repeat" — which is precisely what Pattern 6 (Critic) is built for. Recognizing that a single system can need *different* coordination patterns for different parts of its flow, rather than forcing everything through one pattern, is itself one of the most useful design instincts this course tries to build (see also the Router-plus-Debate hybrid suggested as a Challenge in Module 15).

### What Happens When the Quality Reviewer Rejects a Draft

Walking through the loop-back in the diagram concretely: the Quality Reviewer doesn't rewrite the article itself — its only output is structured feedback, something like `{"approved": false, "issues": ["claim in paragraph 2 has no supporting source", "tone is too informal for the target audience"]}`. That structured feedback (not a vague "make it better") is what gets routed back to the Writer Agent, along with the current draft, so the Writer's next attempt addresses concrete, specific problems rather than guessing at what changed. Each round increments a counter; when the counter hits the 3-round cap mentioned in the Implementation Notes, the system stops looping automatically and instead routes the draft, plus the full history of the Quality Reviewer's objections across all three rounds, to a human for a final call (Module 18) — this is the same "detect → fallback → alternative strategy" shape from Module 17.2's general reliability loop, just instantiated for a content-quality failure instead of a tool failure.

### Why "Shared Structured State" Instead of Passing Full Text Between Agents

It's tempting to just have each agent pass its entire raw output forward as one big blob of text — simpler to code at first. The problem surfaces once you look at what the SEO Specialist actually needs versus what the Research Agent produced: the SEO agent doesn't need the Research Agent's full notes and source list, it needs the *current draft text* plus a short list of *target keywords* the Research Agent identified along the way. If every agent receives everything every previous agent ever produced, two things go wrong as the pipeline grows: context windows fill up with irrelevant material (Module 2.2's context rot, now happening across agent hops instead of within one conversation), and it becomes much harder to debug which agent actually used which piece of information when something goes wrong. Keeping distinct, named, structured fields (`sources`, `seo_keywords`, `flagged_issues`) — each populated by the agent responsible for it and consumed only by the agents that actually need it — keeps every hop in the pipeline both cheaper and easier to trace.

---

## Project Roadmap Table

| Project | Difficulty | Skills Learned |
|---|---|---|
| Personal Task Assistant | Beginner | Structured outputs, basic persistence, simple prompting |
| Research Agent | Beginner/Intermediate | Tool calling, ReAct loop, source tracking |
| RAG Knowledge Assistant | Intermediate | Embeddings, vector search, grounded generation |
| Autonomous Research System | Advanced | Planning, replanning, reflection, checkpointing |
| Multi-Agent Content Company | Advanced | Multi-agent orchestration, critic/pipeline patterns, shared state |

Continue to **[24-Capstone.md](24-Capstone.md)**.
