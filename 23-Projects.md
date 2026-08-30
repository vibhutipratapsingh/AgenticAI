# Part 13 — Hands-On Projects

Five progressively harder projects, each applying concepts from the modules above. Build them in order.

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

1. **Define the data model.**
```python
# task schema
{"id": str, "description": str, "due_date": str | None, "priority": "low"|"medium"|"high", "done": bool}
```

2. **Build a "parse task" LLM call** that turns free text into the structured schema:
```python
def parse_task(user_text: str) -> dict:
    prompt = f"""Extract a task from this text. Respond as JSON matching:
    {{"description": str, "due_date": "YYYY-MM-DD" or null, "priority": "low"|"medium"|"high"}}
    Text: {user_text}"""
    return json.loads(ask_llm(prompt, temperature=0))
```

3. **Persist tasks** to a simple database (SQLite is fine for a personal project).

4. **Build a prioritization function** — sort by due date proximity and priority level; this can be plain code (no LLM needed — a good example of Module 5's "don't over-agent-ify" principle).

5. **Build a Q&A function** that retrieves relevant tasks (filter by date/status) and asks the LLM to answer naturally:
```python
def answer_question(question: str, tasks: list[dict]) -> str:
    context = json.dumps(tasks)
    return ask_llm(f"Given these tasks: {context}\n\nAnswer: {question}")
```

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

1. **Define a search tool** (a real web-search API, or a mocked one for practice).
2. **Implement a ReAct loop** (Module 12.1): the agent alternates between deciding to search and synthesizing what it's found, until it has enough information.
3. **Track sources** alongside collected information, so the final report can cite them.
4. **Generate the final report** — a structured summary with sections and citations, not just a single paragraph.

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

1. **Ingestion pipeline**: chunk uploaded documents (Module 9.3), embed each chunk, store in a vector DB with metadata (source filename, section).
2. **Query pipeline**: embed the user's question, retrieve top-k relevant chunks, inject as context, instruct the model to answer only from context (Module 10.2).
3. **Add citations**: return which document/section each part of the answer came from.
4. **Handle "not found" gracefully**: if no chunk is sufficiently relevant, respond "I don't have that information" rather than guessing.

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

1. **Generate an upfront plan** for the research question (plan-and-execute, Module 12.2).
2. **Execute each planned sub-task**, using the ReAct loop from Project 2 for each one, saving results into persistent state after each step (checkpointing, Module 16.3).
3. **Replan** if a sub-task reveals the original plan needs adjusting (Module 11.2).
4. **Reflect**: before producing the final report, have the agent critique its own draft against the original question and revise gaps.
5. **Produce the final report**, with sources and a summary of the plan actually followed (transparency for the user).

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
