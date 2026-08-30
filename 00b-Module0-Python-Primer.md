# Module 0 — Python Primer for Agentic AI

### Difficulty
Beginner

### Learning Objectives
- Learn the specific slice of Python used throughout this course: variables, data types, functions, control flow, lists/dictionaries, and classes.
- Learn to work with JSON (the format almost every LLM response and tool call in this course uses).
- Learn to call a web API and manage dependencies with `pip` and virtual environments.

### Prerequisites
None — this is the starting point referenced by the roadmap's "Python Basics" step.

> **Note:** This is not a full Python course. It covers exactly the subset of Python used in this course's code examples, so that Modules 1 onward make sense even if you've never written Python before. If you already know Python, skim this module for the JSON/API sections and move on to **[00-Course-Overview.md](00-Course-Overview.md)**.

---

## Lesson 0.1 — Variables, Data Types, and Basic Operations

### Concept Explanation

A **variable** is a named box that holds a value. Python's core data types you'll see constantly in this course: `str` (text), `int`/`float` (numbers), `bool` (True/False), `list` (ordered collection), `dict` (key-value pairs — this is what JSON maps onto directly).

### Practical Example

```python
agent_name = "ResearchBot"        # str
max_steps = 10                    # int
temperature = 0.3                 # float
is_active = True                  # bool

tools_available = ["search", "calculator", "get_weather"]   # list

agent_config = {                  # dict
    "name": agent_name,
    "max_steps": max_steps,
    "temperature": temperature,
}

print(agent_config["name"])       # → "ResearchBot"
```

*Explanation:* `agent_config` is a dictionary — notice this is exactly the shape you'll see for tool schemas, LLM API requests, and agent state throughout the course (e.g., Module 6, 7, 16). Getting comfortable reading and building dictionaries is the single most useful Python skill for this course.

### Common Mistakes
- Mixing up `=` (assignment: "set this variable to this value") with `==` (comparison: "is this equal to that?").
- Forgetting that dictionary keys are case-sensitive strings — `config["Name"]` and `config["name"]` are different keys.

---

## Lesson 0.2 — Functions

### Concept Explanation

A **function** is a reusable, named block of code that takes input (parameters) and can return output. Every "tool" in Module 7 is just a Python function with a description attached.

### Practical Example

```python
def get_weather(city: str) -> dict:
    """Returns weather info for a given city."""
    fake_data = {"Pune": {"temp_c": 29, "condition": "Sunny"}}
    return fake_data.get(city, {"error": f"No data for {city}"})

result = get_weather("Pune")
print(result)   # → {"temp_c": 29, "condition": "Sunny"}
```

*Explanation:* `city: str` and `-> dict` are **type hints** — they document what type of input the function expects and what it returns, without enforcing it at runtime. You'll see this style throughout the course's code examples (Modules 6, 7, 16) because it makes tool schemas and agent code much easier to read.

### Common Mistakes
- Forgetting `return` — a function without `return` gives back `None`, a common source of confusing bugs.
- Not handling the case where input doesn't match what's expected (see `.get(city, {"error": ...})` above, which avoids crashing on an unknown city).

---

## Lesson 0.3 — Control Flow: `if`, `for`, `while`

### Concept Explanation

- `if`/`elif`/`else`: run different code depending on a condition.
- `for`: repeat code once per item in a collection (a list, a range of numbers, etc.).
- `while`: repeat code as long as a condition stays true — this is exactly how the **agent loop** (Module 6) is implemented in code.

### Practical Example

```python
def run_agent(goal, tools, llm, max_steps=10):
    step = 0
    while step < max_steps:                     # the agent loop, Module 6
        decision = llm.decide_next_step(goal)

        if decision["action"] == "finish":
            return decision["final_answer"]
        elif decision["action"] == "tool_call":
            tool = tools[decision["tool_name"]]
            result = tool(**decision["tool_input"])
        else:
            result = {"error": "unknown action"}

        step += 1

    return "Stopped: reached max steps."
```

*Explanation:* this is a simplified version of the exact loop you'll build for real in Module 6.2 and Module 7.3 — `while step < max_steps` is the code-level version of the "max step limit" reliability guardrail covered in Module 17.

### Common Mistakes
- Writing a `while True:` loop with no exit condition — this is precisely the "infinite loop" failure mode described in Module 17.1. Always give a `while` loop a clear way to stop.

---

## Lesson 0.4 — Lists and Dictionaries in Depth

### Concept Explanation

Almost every piece of "state" or "memory" in this course (Module 8, 16) is a list of dictionaries — a list of past messages, a list of completed steps, a list of retrieved document chunks.

### Practical Example

```python
history = []   # this list will grow as the agent takes steps

history.append({"step": 1, "tool": "search", "result": "12 laptops found"})
history.append({"step": 2, "tool": "compare", "result": "3 laptops shortlisted"})

for entry in history:
    print(f"Step {entry['step']}: used {entry['tool']} → {entry['result']}")

# Filtering (a simplified version of what memory retrieval does, Module 8-9)
search_steps = [h for h in history if h["tool"] == "search"]
```

*Explanation:* the `for entry in history:` loop and the list comprehension (`[h for h in history if ...]`) are both patterns you'll reuse constantly — from replaying an agent's trace, to filtering retrieved memory, to preparing tool schemas.

---

## Lesson 0.5 — Classes (Just Enough to Read Agent Code)

### Concept Explanation

A **class** bundles data and functions together into one reusable object. This course uses simple classes for things like agent state (Module 16).

### Practical Example

```python
class AgentState:
    def __init__(self, task_id: str):
        self.task_id = task_id
        self.completed_steps = []
        self.status = "in_progress"

    def add_step(self, step: dict):
        self.completed_steps.append(step)

state = AgentState(task_id="task-001")
state.add_step({"tool": "search", "result": "done"})
print(state.status)              # → "in_progress"
print(len(state.completed_steps))  # → 1
```

*Explanation:* `__init__` runs once when you create the object (`AgentState(task_id="task-001")`) and sets up its starting data. `self` refers to "this particular object" — `self.completed_steps` is a list that belongs to this one state object, not shared with other agent runs. You don't need to master classes deeply for this course — just be able to read code shaped like this, since it appears in Module 16's state management examples.

---

## Lesson 0.6 — JSON: The Universal Format for LLM Data

### Concept Explanation

**JSON** (JavaScript Object Notation) is a text format for structured data that looks almost identical to a Python dictionary. It's the format LLM APIs use for requests/responses, the format tool schemas are written in (Module 7), and the format structured outputs are returned in (Module 2.5).

### Practical Example

```python
import json

# Python dict → JSON string (e.g., to send in an API request)
tool_call = {"tool": "get_weather", "input": {"city": "Pune"}}
json_string = json.dumps(tool_call)
print(json_string)   # → '{"tool": "get_weather", "input": {"city": "Pune"}}'

# JSON string → Python dict (e.g., parsing an LLM's structured response)
llm_response_text = '{"date": "2026-09-01", "confidence": "high"}'
parsed = json.loads(llm_response_text)
print(parsed["date"])   # → "2026-09-01"
```

*Explanation:* `json.dumps` converts a Python object *into* a JSON string; `json.loads` converts a JSON string *into* a Python object. Nearly every "structured output" example in Module 2.5, Module 3, and Module 7 depends on this loads/dumps pair — an LLM's JSON output arrives as text, and `json.loads` is what turns it back into a usable Python dictionary your code can act on.

### Common Mistakes
- Forgetting that an LLM's JSON output is still just text until you call `json.loads` on it — trying to use `response["field"]` directly on the raw string will fail.
- Not handling malformed JSON — a model can occasionally produce invalid JSON; production code wraps `json.loads` in a `try/except` (this connects directly to Module 17's validation guidance).

---

## Lesson 0.7 — Calling a Web API

### Concept Explanation

Nearly every tool in Module 7, every LLM call in Module 2 onward, and every RAG retrieval in Module 10 is, underneath, an HTTP request to some API. The `requests` library is the standard way to make these calls in Python.

### Practical Example

```python
import requests

response = requests.post(
    "https://api.example-llm-provider.com/v1/messages",
    headers={"Authorization": "Bearer YOUR_API_KEY"},
    json={
        "model": "some-model-name",
        "messages": [{"role": "user", "content": "Hello!"}],
    },
)

data = response.json()          # parses the JSON response body automatically
print(data["content"])
```

*Explanation:* `headers` carries authentication (Module 21.4 — never hardcode a real key like this in shared code; use environment variables instead); `json=...` tells `requests` to send your dict as a JSON request body; `.json()` parses the JSON response straight into a Python dict, combining Lessons 0.6 and 0.7 into the exact pattern every LLM API call in this course follows.

### Common Mistakes
- Hardcoding API keys directly in source code (see Module 21.4) — use `os.environ.get("API_KEY")` and a `.env` file instead.
- Not checking `response.status_code` before assuming the call succeeded — a failed request (rate limit, network error) still returns a `response` object, just with an error status.

---

## Lesson 0.8 — `pip` and Virtual Environments

### Concept Explanation

- **`pip`** is Python's package installer — it downloads and installs libraries (like `requests`, an LLM provider's SDK, or a vector database client).
- A **virtual environment** is an isolated Python installation for one project, so its dependencies don't conflict with other projects on your machine.

### Practical Example

```bash
# Create a virtual environment named "venv" for this project
python -m venv venv

# Activate it (macOS/Linux)
source venv/bin/activate
# Activate it (Windows PowerShell)
venv\Scripts\Activate.ps1

# Install the libraries this course's examples use
pip install requests python-dotenv chromadb

# Save your project's exact dependencies for others to reproduce
pip freeze > requirements.txt
```

*Explanation:* activating a virtual environment before running `pip install` keeps this project's libraries separate from every other Python project on your machine — this matters once you're juggling multiple course projects (Module 23) with potentially different library versions.

### Common Mistakes
- Installing packages globally (without activating a virtual environment first) — works at first, but eventually causes version conflicts between unrelated projects.
- Forgetting to activate the virtual environment in a new terminal session — `pip install` will silently install into the wrong place.

---

## Key Takeaways
- This course's code examples rely on a small, consistent slice of Python: variables/dicts/lists, functions, `if`/`for`/`while`, simple classes, JSON, HTTP requests, and basic dependency management.
- Dictionaries and JSON are the connective tissue of everything from here on — tool schemas, LLM responses, agent state, and memory records are all, at heart, nested dictionaries.
- The `while` loop pattern in Lesson 0.3 *is* the agent loop from Module 6 — once you're comfortable with it, Module 6 will feel like a direct continuation, not a new concept.

### Exercise
Write a Python function `summarize_tasks(tasks: list[dict]) -> str` that takes a list of task dictionaries (each with `"description"` and `"done"` keys) and returns a string like `"2 of 5 tasks done."` Test it with a small hand-written list.

### Challenge
Write a small script that: (1) defines a Python dict representing a fake "tool schema" (name, description, input schema, matching the shape from Module 7.2), (2) converts it to a JSON string with `json.dumps`, (3) parses it back with `json.loads`, and (4) prints the tool's `"description"` field from the round-tripped result.

### Knowledge Check
1. Why does almost every tool schema, LLM response, and agent state object in this course end up looking like a Python dictionary?
2. What's the difference between `json.dumps` and `json.loads`, and when would you use each?
3. Why does a `while` loop need an explicit exit condition, and what agent failure mode (from later in the course) happens when it doesn't?

Continue to **[00-Course-Overview.md](00-Course-Overview.md)** to begin the main course.
