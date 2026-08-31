# Part 14 — Capstone Project

# AI Business Automation Agent Platform

This capstone synthesizes every module into one production-style system. It should:

1. Receive business goals from a user (natural language).
2. Break goals into tasks (planning, Module 11).
3. Select appropriate tools for each task (Module 7).
4. Execute tasks (agent loop, Module 6).
5. Store memory across sessions (Module 8–9).
6. Monitor progress (Module 20).
7. Request human approval when necessary (Module 18).
8. Generate reports (Module 19 informs what "success" means to measure).

Why build this particular system as the capstone, rather than a bigger version of any single earlier project? Every prior project in Module 23 deliberately isolated one or two new skills so you could learn them cleanly. Real production agent systems never get that luxury — a genuine business automation platform needs planning *and* tool execution *and* memory *and* human oversight *and* observability, all operating together, all the time, under real constraints (cost, latency, safety) that don't show up when you're building a solo weekend project. This capstone is the first place in the course where you're asked to make the trade-offs those constraints force on you simultaneously — e.g., a memory design that's convenient for the Reporting Agent but expensive for the Research Agent to write to, or a risk-tier boundary that's safe but slows down otherwise-routine work. Treat every design decision below not as "the one correct answer" but as one reasonable resolution of a trade-off — you should be able to articulate what you'd change if the priorities were different (e.g., a system that must never lose data vs. one that must respond in milliseconds).

---

## System Architecture

```mermaid
flowchart TD
    U([User: Web UI]) --> API["Backend API<br/>(auth, rate limits)"]
    API --> Planner["Task Planner Service<br/>(decomposes goal → task list)"]
    Planner --> Orch["Agent Orchestrator<br/>(Supervisor pattern, Module 15.1)"]

    Orch --> Res["Research Agent<br/>(gathers info)"]
    Orch --> Exec["Execution Agent<br/>(calls external tools/APIs)"]
    Orch --> Appr["Approval Gateway<br/>(Module 18: pauses on high-risk actions)"]
    Orch --> Rep["Reporting Agent<br/>(summarizes progress/results)"]

    Res --> Store[("Shared State & Memory Store<br/>Postgres + Vector DB")]
    Exec --> Store
    Appr --> Store
    Rep --> Store

    Store --> Mon["Monitoring & Logging<br/>(structured logs, dashboards, alerts)"]

    style U fill:#e0e7ff,stroke:#4338ca
    style Orch fill:#fce7f3,stroke:#be185d
    style Appr fill:#fee2e2,stroke:#dc2626
    style Store fill:#fef3c7,stroke:#d97706
```

**How to read this graph:** trace a goal from top to bottom — it's decomposed into tasks *before* it ever reaches the multi-agent layer, which is why the Task Planner sits above the Orchestrator rather than being one of the four specialist agents. Notice the red "Approval Gateway" box is a peer of the other three agents, not a filter every agent must pass through individually — this matches Module 18's design: only actions the Execution Agent classifies as medium-risk-or-higher get routed through it, everything else flows straight to the shared store. Every agent — including the gateway — writes to the same yellow Shared State & Memory Store, which is what lets the Reporting Agent produce a coherent final report without needing to talk to the other three agents directly.

---

## Database Design (Core Tables)

```text
users(id, email, role, created_at)

goals(id, user_id, description, status, created_at)

tasks(id, goal_id, description, status, priority, assigned_agent, created_at, completed_at)

task_steps(id, task_id, step_number, plan_summary, tool_used, tool_input,
           tool_output, status, checkpoint_data, created_at)

approvals(id, task_step_id, risk_level, proposed_action, status
          ["pending","approved","rejected"], reviewer_id, reviewed_at)

memory_facts(id, user_id, fact_text, embedding_vector, source, created_at)

agent_logs(id, task_step_id, level, message, cost_tokens, latency_ms, created_at)
```

*Design notes:* `task_steps.checkpoint_data` supports recovery (Module 16.3). `approvals` implements the risk-tiered human-in-the-loop gate (Module 18.2). `memory_facts` with an `embedding_vector` column implements long-term semantic memory (Module 8–9). `agent_logs` captures per-step cost/latency for evaluation and cost tracking (Module 19, 22).

It's worth walking through what would actually break if any one of these tables were missing, since that's a more concrete way to understand *why* the schema is shaped this way than just reading column names. Drop `task_steps` entirely and try to keep only a final `tasks.status` field, and you lose Module 16's checkpointing guarantee outright — if the process crashes mid-task, there is no record of which of the task's steps already completed, so recovery has no choice but to restart the whole task from scratch, potentially re-sending an email or re-charging a payment that the first attempt had already completed. Drop the `approvals` table and try to just check risk level inline in application code instead, and you lose the audit trail Module 18 depends on — six months from now, when someone asks "who approved this $50,000 payment and when," there is no durable record to answer that question, only whatever happened to be in a log file that may have already rotated out. Drop `memory_facts` and try to keep long-term memory only in `goals`/`tasks` rows, and the Research Agent loses the ability to retrieve semantically related facts across *different* goals — Module 9's whole point, semantic similarity search across a large corpus of stored knowledge, requires a dedicated store built for that access pattern, not a generic goal-tracking table. And drop `agent_logs`, and Module 19's evaluation metrics (task success rate, cost, latency) have no raw data to compute themselves from — you'd be flying blind on exactly the numbers that tell you whether the platform is actually working.

Two column-level choices are worth calling out specifically. First, `approvals.status` is constrained to exactly three values (`"pending"`, `"approved"`, `"rejected"`) rather than being a free-text field — this is a deliberate application of Module 17's validation guidance at the database layer: a typo'd status string (`"aproved"`) would silently break every piece of code that checks `status == "approved"`, and constraining the column at the schema level catches that class of bug before it ever reaches production. Second, `task_steps.tool_input` and `tool_output` are stored as their own columns rather than folded into `plan_summary` as one blob of text — this separation is what lets the Reporting Agent (or a human debugging a failure) distinguish "what the agent *decided* to do" from "what actually *happened* when it did it," which matters enormously when a tool call fails: the plan might have been perfectly reasonable, and the failure lived entirely in `tool_output`, or vice versa.

---

## Agent Architecture

| Agent | Role | Tools | Risk Tier of Its Actions |
|---|---|---|---|
| Task Planner | Decomposes a goal into an ordered task list | None (pure LLM reasoning) | Low |
| Research Agent | Gathers information needed for a task | Search, document retrieval | Low |
| Execution Agent | Performs the actual business action | Email, CRM update, payment API, file operations | Medium–Critical (varies per tool) |
| Approval Gateway | Enforces human-in-the-loop policy | None (pure control logic) | N/A (a control component, not a reasoning agent) |
| Reporting Agent | Summarizes progress and results for the user | None (pure LLM reasoning over logged data) | Low |

Look closely at the "Risk Tier" column and notice it's not uniform across agents — three of the five rows are a single fixed tier, but the Execution Agent's row says "varies per tool." This is deliberate and mirrors a subtlety from Module 18.2 that's easy to miss: risk isn't a property of an *agent*, it's a property of an *action*. The Research Agent is always Low risk because everything it does — searching, reading documents — is read-only and reversible; there's no version of "search the web" that needs human approval. The Execution Agent, by contrast, is the one agent whose whole job is to take real external actions, and those actions span the entire risk spectrum from a routine internal Slack message to an irreversible payment — so the *agent* can't be assigned one risk tier, only its individual tool calls can (which is exactly why the Approval Gateway classifies at the tool level, not the agent level, in the Tool Architecture below). Also notice the Task Planner and Reporting Agent are both "pure LLM reasoning" with no tools at all — this is intentional scope-narrowing, the same instinct from Module 21's least-privilege principle: an agent that never needs to take external action should never be *given* the ability to, even if it would technically work, because every tool you don't hand out is one less thing that can be misused if that agent's reasoning goes wrong (via a bug, or a prompt injection from a document the Research Agent fed it).

---

## Tool Architecture

```text
Tool Registry
├── low_risk/
│   ├── search(query)
│   ├── read_file(path)
│   └── query_db(sql)      [read-only credentials]
├── medium_risk/
│   ├── send_internal_message(channel, text)
│   └── update_crm_record(id, fields)
├── high_risk/
│   ├── send_external_email(to, subject, body)
│   └── update_production_config(key, value)
└── critical_risk/
    ├── issue_payment(amount, recipient)
    └── delete_customer_data(user_id)
```

Each tool is tagged with a risk tier (Module 18.2) enforced centrally by the Approval Gateway — individual agents cannot bypass this classification.

It's worth understanding *why* the boundaries fall exactly where they do, because "risk tier" can otherwise feel like an arbitrary label rather than a reasoned engineering decision. The `low_risk` tier contains only read operations (`search`, `read_file`) plus one interesting case: `query_db(sql)` is explicitly annotated `[read-only credentials]` — meaning the *database connection itself* is configured so that even a maliciously or accidentally crafted `DROP TABLE` statement would fail at the database layer, regardless of what the agent's reasoning decided to do. This is defense-in-depth (Module 21): the risk-tier label is one layer of protection, and the underlying credential's actual permissions are a second, independent layer that holds even if the first one is somehow bypassed. The `medium_risk` tier covers actions that are real but bounded and reversible — sending an internal Slack message can be deleted or corrected with an apology; updating a CRM record can be corrected with another update — so these execute with logging but without a mandatory pause, trading a small amount of risk for meaningfully faster throughput on routine work. The jump to `high_risk` happens specifically at the boundary of "external-facing or affecting systems outside this platform's own database" — `send_external_email` and `update_production_config` both have consequences that reach *other* systems or *other people* who have no way to see or undo the platform's internal state. And `critical_risk` is reserved for actions that are both external-facing *and* effectively irreversible — you cannot "un-send" a payment or "un-delete" a customer's data the way you can send a follow-up correction email. Notice this closely mirrors the four-tier table in Module 18.2, but here it's expressed as a concrete, enforceable registry structure rather than an abstract table — this is what "risk tiering" actually looks like once it's implemented rather than just described.

---

## API Design (Core Endpoints)

```text
POST   /goals                 → submit a new business goal
GET    /goals/{id}            → check status and progress
GET    /goals/{id}/tasks      → list decomposed tasks and their status
GET    /approvals/pending     → list actions awaiting human approval
POST   /approvals/{id}/decide → approve/reject/edit a pending action
GET    /goals/{id}/report     → get the final generated report
GET    /metrics               → task success rate, cost, latency (Module 19)
```

Long-running goals use the async pattern from Module 20.4: `POST /goals` returns immediately with a `goal_id` and `status: "processing"`; the client polls `GET /goals/{id}` or receives a webhook when complete.

---

## Folder Structure

```text
agentic-platform/
├── backend/
│   ├── api/                 # FastAPI routes (auth, goals, approvals, metrics)
│   ├── orchestrator/        # supervisor logic, agent loop, replanning
│   ├── agents/
│   │   ├── planner.py
│   │   ├── researcher.py
│   │   ├── executor.py
│   │   └── reporter.py
│   ├── tools/
│   │   ├── registry.py      # risk-tier tagging + dispatch
│   │   └── implementations/
│   ├── memory/
│   │   ├── vector_store.py
│   │   └── fact_store.py
│   ├── approval/
│   │   └── gateway.py
│   ├── db/
│   │   └── models.py
│   └── monitoring/
│       └── logging.py
├── frontend/                 # goal submission UI, approval review UI, dashboards
├── tests/
│   ├── unit/
│   ├── eval/                 # agent evaluation test datasets (Module 19)
│   └── security/             # prompt injection / permission tests (Module 21)
└── infra/
    ├── docker/
    └── deployment configs
```

---

## Development Roadmap

```text
Phase 1: Core agent loop + single Execution Agent with 2-3 low-risk tools
Phase 2: Add Task Planner + multi-step task decomposition
Phase 3: Add Approval Gateway with risk tiering (Module 18)
Phase 4: Add memory (short-term + long-term via vector store, Module 8-9)
Phase 5: Add Reporting Agent + metrics dashboard (Module 19-20)
Phase 6: Add security hardening (prompt injection tests, least-privilege tools,
         Module 21) + cost optimization pass (Module 22)
Phase 7: Load testing, monitoring/alerting, production deployment (Module 20)
```

The reasoning behind this specific ordering is at least as important as the list itself, because it's a direct application of the same "build the simplest working version first, then layer in the concerns that make it production-grade" philosophy that runs through the whole course. Phase 1 deliberately starts with only low-risk tools and no planning at all — this gets you a genuinely working agent loop (Module 6) as fast as possible, which gives you something real to test every subsequent phase against, rather than trying to design the Task Planner, the Approval Gateway, and memory all at once with nothing running yet to validate any of it. Phase 2 (Task Planner) comes before Phase 3 (Approval Gateway) for a specific reason: until tasks are actually being decomposed into multiple steps, there's nothing risky enough happening yet to meaningfully test approval gating against — building the gateway first would mean testing it against a toy scenario instead of real multi-step task execution. Phase 3 has to precede Phase 4's high-risk tool additions (implied by memory unlocking more sophisticated, farther-reaching tasks) for a safety reason, not just a convenience one: you never want a system capable of taking consequential actions running even briefly without its approval gate already in place. Phase 5 (Reporting + metrics) comes after the core loop, planning, approvals, and memory all exist because there's nothing meaningful to report or measure before then — Module 19's evaluation metrics need real task executions to compute themselves from. And security hardening and cost optimization are deliberately placed at Phase 6, not Phase 1, not because they're unimportant, but because you cannot meaningfully security-test or cost-optimize a system whose final shape isn't built yet — Phase 7's load testing and production deployment are the last step for the same reason you wouldn't stress-test a building's foundation before finishing its walls.

---

## Testing Strategy

| Test Type | What It Covers | Module Reference |
|---|---|---|
| Unit tests | Individual tool functions, planner output schema validation | 7, 17 |
| Agent evaluation suite | Task success rate, tool accuracy across representative + edge cases | 19 |
| Security tests | Prompt injection attempts, permission boundary checks | 21 |
| Reliability tests | Simulated tool failures, timeout handling, loop detection | 17 |
| Load tests | Concurrent goal submissions, queue behavior under load | 20 |
| Human-in-the-loop tests | Verify high/critical-risk actions are always gated, never bypassed | 18 |

---

## Deployment Strategy

- **Local**: Docker Compose running API, orchestrator, Postgres, and vector DB together for development.
- **Staging**: Containerized deployment on a cloud provider, using queued/async task processing (Module 20.4), with monitoring dashboards active.
- **Production**: Scaled containers behind the backend API, autoscaling workers processing the task queue, secrets managed via a secrets manager (never in code or logs, Module 21.4), full logging/monitoring/alerting active before go-live.

---

This capstone is intentionally large — treat it as a multi-week project, building phase by phase per the roadmap above, applying the reliability, security, evaluation, and cost lessons from Modules 17, 19, 21, and 22 at every phase rather than bolting them on at the end.

That last clause — "rather than bolting them on at the end" — is worth taking seriously rather than reading as boilerplate advice. It's tempting, once Phase 1 through 5 are working and the platform genuinely does something impressive, to treat security tests and load tests as a final checklist item before "shipping." The Testing Strategy table above is deliberately organized by *what* each test type catches, not by when to run it, precisely so you build the habit of writing a security test or a reliability test the same week you build the feature it protects, not months later once the surface area to secure has grown far beyond what you can hold in your head. A prompt-injection test written against the Research Agent the day you build it is cheap; a prompt-injection audit of a five-agent platform with a payment tool, written after the fact, is a much larger and much scarier undertaking — and it's exactly the undertaking you'd be facing if security got deferred to "Phase 6" in practice rather than merely in the roadmap's labeling.

Continue to **[25-Learning-Roadmap.md](25-Learning-Roadmap.md)**.
