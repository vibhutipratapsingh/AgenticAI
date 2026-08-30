# Module 16 — Agent State Management

### Difficulty
Advanced

### Learning Objectives
- Distinguish stateless vs. stateful agents.
- Understand session state vs. persistent state.
- Understand checkpointing and recovery.

### Prerequisites
Modules 1–15.

---

## Lesson 16.1 — Stateless vs. Stateful Agents

### Concept Explanation

A **stateless agent** treats each request independently, with no memory of prior interactions unless everything needed is passed in again each time. A **stateful agent** maintains context — its current plan, progress, and history — across multiple steps or sessions.

### Simple Analogy

> A stateless agent is like a customer service rep with amnesia between every call — you must re-explain your entire situation every time. A stateful agent is like a rep who has your case file open and remembers exactly where you left off.

### Visual Diagram

```text
STATELESS:
Request 1 → [fresh context] → Response 1
Request 2 → [fresh context, no memory of Request 1] → Response 2

STATEFUL:
Request 1 → [context + saved state] → Response 1 → [state updated & saved]
Request 2 → [context + PREVIOUS state loaded] → Response 2 → [state updated & saved]
```

---

## Lesson 16.2 — Session State vs. Persistent State

| Type | Scope | Example | Storage |
|---|---|---|---|
| **Session state** | Lasts for one active session/run | Current plan, steps completed so far in this task | In-memory, or a session store (e.g., Redis) |
| **Persistent state** | Survives across sessions, restarts, even server crashes | User preferences, long-running task progress that spans days | Database (SQL/NoSQL), durable storage |

### Practical Example

```python
class AgentState:
    def __init__(self, task_id: str):
        self.task_id = task_id
        self.plan: list[str] = []
        self.completed_steps: list[dict] = []
        self.status: str = "in_progress"

    def save(self, db):
        db.save_state(self.task_id, self.__dict__)   # persistent state

    @classmethod
    def load(cls, task_id: str, db):
        data = db.load_state(task_id)
        state = cls(task_id)
        state.__dict__.update(data)
        return state
```

*Explanation:* `save`/`load` make state persistent by writing it to a database rather than only keeping it in memory — critical for long-running agent tasks (e.g., a research task that takes 20 minutes and multiple tool calls) that must survive a server restart or be resumed later.

---

## Lesson 16.3 — Checkpointing and Recovery

### Concept Explanation

**Checkpointing** means saving the agent's state at meaningful points during execution (after each completed step, for instance), so that if something fails partway through, the agent can **recover** by resuming from the last checkpoint instead of restarting the entire task from scratch.

### Visual Diagram

```text
Step 1 → ✅ Checkpoint saved
Step 2 → ✅ Checkpoint saved
Step 3 → ❌ CRASH (server restart, tool timeout, etc.)
       ↓
Recovery: Load last checkpoint (after Step 2)
       ↓
Resume from Step 3 (not Step 1)
```

### Practical Example

```python
def run_long_task(task_id: str, plan: list[str], db):
    state = AgentState.load(task_id, db) or AgentState(task_id)
    start_index = len(state.completed_steps)

    for i, step in enumerate(plan[start_index:], start=start_index):
        result = execute_step(step)
        state.completed_steps.append({"step": step, "result": result})
        state.save(db)   # checkpoint after every step

    state.status = "done"
    state.save(db)
    return state
```

*Explanation:* `start_index = len(state.completed_steps)` is what enables resuming mid-plan — if the process crashes after step 2, reloading state and recomputing `start_index` correctly skips the already-completed steps instead of redoing them.

### Key Takeaways
- Stateless agents are simpler but can't handle multi-step tasks spanning time without external state.
- Session state handles "this run"; persistent state handles "across runs/restarts."
- Checkpointing at each meaningful step is what makes recovery from failure cheap instead of catastrophic.

### Common Mistakes
- Only saving state at the very end of a long task — a crash halfway through loses all progress.
- Storing state only in memory for tasks that realistically take longer than a single server process's lifetime (risk of loss on deploys/restarts/crashes).
- Not versioning or validating loaded state — a schema change between deployments can break resuming old in-flight tasks.

### Exercise
Design the state schema (as a list of fields) for a multi-step "trip planning" agent that could take several minutes and multiple tool calls to complete, such that it could be safely resumed after a crash at any point.

### Challenge
What should happen if, upon recovery, the agent discovers that the world has changed since the last checkpoint (e.g., a previously found flight is now sold out)? Design the recovery logic to handle this rather than blindly resuming.

### Knowledge Check
1. What's the difference between session state and persistent state?
2. Why is checkpointing after every step better than checkpointing only at the end?
3. What extra check should recovery logic perform beyond just reloading the last state?
