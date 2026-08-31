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

Every program, at its core, needs somewhere to keep track of things while it's running — the current step number, a user's question, the result of a calculation. A **variable** is that "somewhere": a named label you attach to a value, so you (and the computer) can refer back to that value later by name instead of having to retype it or remember it by its raw content.

Think about why this even matters. Without variables, you'd have to write `10 < 10` as a literal, unchanging comparison every time — there'd be no way to say "keep comparing against however many steps we've done *so far*." A variable like `step` lets that number change as the program runs, while every piece of code that reads `step` automatically sees the current value. This is the entire foundation of how programs "remember" anything: a variable is a labeled, mutable slot in the computer's memory.

Python is what's called a **dynamically typed** language, which means you never have to declare in advance "this variable will only ever hold text" the way some other languages require. You just write `agent_name = "ResearchBot"` and Python figures out, from the value itself, that this is a string. This is convenient for beginners, but it comes with a trade-off worth knowing up front: Python won't stop you from later writing `agent_name = 42`, silently changing what kind of thing `agent_name` refers to. That flexibility is powerful, but it's also a common source of subtle bugs in larger programs — a variable you expected to always be text turns out, three functions later, to have been quietly reassigned to a number.

Python's built-in data types you'll see constantly in this course, and what each is actually *for*:

- **`str` (string)** — text, always wrapped in quotes (`"hello"` or `'hello'` — both work identically in Python). Used for names, messages, prompts, API keys, anything language-shaped.
- **`int`** — a whole number with no decimal point (`10`, `-3`, `0`). Used for counts: how many steps taken, how many tokens, how many retries.
- **`float`** — a number that *can* have a decimal point (`0.3`, `29.5`). Used for anything continuous or fractional: a temperature setting (Module 2.4), a similarity score (Module 9), a price.
- **`bool`** — exactly two possible values, `True` or `False` (capitalized, no quotes — these are not the strings `"True"`/`"False"`). Used for on/off flags: `is_active`, `is_done`, `has_error`.
- **`list`** — an ordered, changeable collection of values, written with square brackets: `["search", "calculator"]`. Order matters and duplicates are allowed. Used for sequences: a list of available tools, a list of steps taken so far, a list of retrieved document chunks (Module 9).
- **`dict`** (dictionary) — a collection of `key: value` pairs, written with curly braces: `{"name": "ResearchBot"}`. Unlike a list, you look things up by a meaningful key (like `"name"`) rather than by position (like "the third item"). This is, by far, the most important data type in this entire course — more on why in a moment.

### A Common Question: "Why does the dictionary matter so much more than the other types?"

Because a dictionary is exactly the same shape as **JSON** (Lesson 0.6), and JSON is exactly the format that LLM APIs, tool definitions, and agent state all use to talk to each other. When Module 7 describes a "tool schema" as `{"name": "get_weather", "description": "...", "input_schema": {...}}`, that is a Python dictionary, full stop — nested dictionaries inside dictionaries, exactly the way you're about to see below. Once dictionaries feel natural to you, roughly 80% of the code in this course will already look familiar before you've even reached the module that introduces the concept.

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

*Walkthrough, line by line:* the first four lines each create one variable and immediately show, in the comment, which of the five core types it is — notice that Python decided the type purely from how the value was written (quotes → `str`, no decimal → `int`, decimal point → `float`, `True`/`False` keyword → `bool`). `tools_available` is a `list` built from square brackets containing three strings, in a specific order — if you asked for `tools_available[0]`, you'd always get `"search"`, because lists remember position. `agent_config` is a `dict` built from curly braces; each line inside it is one `key: value` pair, separated by commas, and the whole thing can span multiple lines for readability (the trailing comma after `temperature,` is optional but common style). The last line, `agent_config["name"]`, is how you **look up** a value in a dictionary: you put the key you want inside square brackets right after the dictionary's name. This returns `"ResearchBot"` because that's the value stored under the key `"name"`.

**What happens with a key that doesn't exist?** If you instead wrote `agent_config["email"]`, Python would raise a `KeyError` and your program would crash (unless you catch it) — dictionaries do not silently return `None` or an empty value for a missing key the way you might expect. This is a genuinely common beginner trip-up, and it's exactly why Lesson 0.2's example below uses `.get(...)` instead of square brackets when the key might not exist.

*Explanation:* `agent_config` is a dictionary — notice this is exactly the shape you'll see for tool schemas, LLM API requests, and agent state throughout the course (e.g., Module 6, 7, 16). Getting comfortable reading and building dictionaries is the single most useful Python skill for this course.

### Common Mistakes
- **Mixing up `=` and `==`.** A single `=` *assigns* a value ("make `x` equal to 5, right now"); a double `==` *asks a question* ("is `x` currently equal to 5?", returning `True` or `False`). Writing `if step = 5:` instead of `if step == 5:` is actually a syntax error in Python (Python is stricter about this than some languages, and will refuse to run rather than silently doing the wrong thing) — but the *concept* of confusing "set" with "compare" trips up beginners in every language, so it's worth internalizing the distinction early.
- **Assuming dictionary keys are forgiving.** `config["Name"]` and `config["name"]` are two completely different keys as far as Python is concerned — capitalization, spelling, and even invisible whitespace all matter. If a lookup ever mysteriously raises a `KeyError` even though "the key is obviously there," print out `list(config.keys())` and eyeball it character by character; a typo'd key is the single most common cause.
- **Reassigning a variable to a different type partway through a program.** Because Python doesn't stop you from writing `count = 5` and then, fifty lines later, `count = "five"`, it's easy to end up with a variable whose type silently changes mid-program, causing an error far away from the actual mistake. A good habit: pick one type per variable name and stick to it for that variable's whole lifetime.

---

## Lesson 0.2 — Functions

### Concept Explanation

Imagine you needed to check the weather for five different cities in your program. Without any way to package up "the steps to check weather," you'd have to copy-paste the same block of code five times, once per city — and if you ever needed to fix a bug in that logic, you'd have to remember to fix it in all five copies. A **function** solves exactly this problem: it's a named, reusable block of code that you define once and can then *call* as many times as you want, with different input each time.

Mechanically, a function has three parts. The **definition** (`def get_weather(city: str) -> dict:`) is where you write the recipe once. The **parameters** (here, just `city`) are placeholders for whatever input will be supplied each time the function is called — think of them as blank spaces in a template that get filled in on each use. The **return value** is the output the function hands back to whoever called it, via the `return` keyword — this is how a function's result gets out into the rest of your program, rather than just vanishing once the function finishes.

This concept is the single most important idea to internalize before Module 7 (Tool Calling), because **every "tool" an AI agent uses is, underneath, just a Python function.** When Module 7 says "the agent calls the `get_weather` tool," what's actually happening in code is exactly what you're about to see below: a normal Python function gets invoked with some arguments, and its return value is handed back to the agent's reasoning loop as an "observation." There is no special magic tool-calling mechanism at the code level — it's functions, dictionaries, and the loops from Lesson 0.3, combined.

### A Common Question: "What's the difference between a parameter and an argument?"

People use these two words almost interchangeably, but there's a small, useful distinction: the **parameter** is the placeholder name in the function's definition (`city` in `def get_weather(city: str)`); the **argument** is the actual value you plug in when you call it (`"Pune"` in `get_weather("Pune")`). You'll see both terms in error messages and documentation, so it's worth knowing which is which even though the difference rarely changes how you write code day to day.

### Practical Example

```python
def get_weather(city: str) -> dict:
    """Returns weather info for a given city."""
    fake_data = {"Pune": {"temp_c": 29, "condition": "Sunny"}}
    return fake_data.get(city, {"error": f"No data for {city}"})

result = get_weather("Pune")
print(result)   # → {"temp_c": 29, "condition": "Sunny"}
```

*Walkthrough, line by line:* `def get_weather(city: str) -> dict:` starts the function definition — `def` is the keyword that says "I'm about to define a function," `get_weather` is the name we're choosing to call it by, `(city: str)` declares it takes one input called `city` that's expected to be a string, and `-> dict` is a hint that this function will hand back a dictionary. The triple-quoted string right after the `def` line is a **docstring** — a description of what the function does, purely for humans reading the code (Python ignores it functionally). `fake_data = {...}` builds a small dictionary standing in for what would, in a real project, be an actual weather API call. The `return` line does two things at once: it looks up `city` inside `fake_data` using `.get(...)` (explained next), and it immediately hands that result back to wherever the function was called from — execution of the function stops the instant `return` runs. Finally, `result = get_weather("Pune")` *calls* the function with the argument `"Pune"`, and stores whatever comes back into a new variable called `result`.

**Why `.get(city, {"error": ...})` instead of `fake_data[city]`?** This is deliberate, and it directly connects back to the `KeyError` warning from Lesson 0.1. `fake_data[city]` would crash the whole program with a `KeyError` if someone asked for the weather in, say, `"Delhi"` (a city not in our tiny fake dataset). `.get(key, default)` instead says: "look up this key, and if it's not there, calmly hand back this fallback value instead of crashing." This one-line difference is the earliest, simplest example in this course of the **defensive programming** habit that Module 17 (Reliability) builds an entire chapter around: assume your inputs might not be what you expect, and decide in advance what should happen instead of a crash.

**What if you call the function with the wrong type entirely** — say, `get_weather(42)` instead of a string? Because Python's type hints (`city: str`) are not enforced at runtime, this wouldn't immediately error. It would only fail once the code inside the function tried to do something that doesn't work on a number the way it works on a string (in this case, `.get(42)` would just fail to find a match and fall through to the `{"error": ...}` branch, since `42` isn't a key in `fake_data`). Type hints in Python are documentation for humans and tools, not a runtime safety net — real safety comes from validation code you write yourself, which is exactly the theme of Module 17.

*Explanation:* `city: str` and `-> dict` are **type hints** — they document what type of input the function expects and what it returns, without enforcing it at runtime. You'll see this style throughout the course's code examples (Modules 6, 7, 16) because it makes tool schemas and agent code much easier to read.

### Common Mistakes
- **Forgetting `return`.** If you write a function that computes something but never says `return`, Python doesn't complain — it just silently hands back `None` (Python's "nothing" value) to whoever called it. This is one of the most common sources of a confusing bug: your code runs without any error message at all, but three lines later something crashes with `TypeError: 'NoneType' object is not subscriptable`, because you tried to use `result["field"]` on a `None` value. When you see that specific error, the very first thing to check is whether the function that produced `result` actually has a `return` statement on the path that ran.
- **Not handling the case where input doesn't match what's expected.** The `.get(city, {"error": ...})` pattern shown above is one way to handle this; another is to check explicitly with an `if` statement (Lesson 0.3) before proceeding. The underlying principle, which recurs throughout this entire course: a function that only works when its input is "nice" will eventually be called with input that isn't nice, and deciding what happens then is part of writing the function, not an afterthought.
- **Confusing "defining" a function with "calling" it.** Writing `def get_weather(city): ...` does not run any code — it just teaches Python the recipe for later. Nothing actually happens until you write `get_weather("Pune")` somewhere else in your program. Beginners sometimes write a function, run the file, and are confused that "nothing happened" — the fix is realizing the function was only ever called at the bottom, or never called at all.

---

## Lesson 0.3 — Control Flow: `if`, `for`, `while`

### Concept Explanation

So far, every example has been code that runs top to bottom, once, in a straight line. Real programs almost never work that way — they need to make decisions ("if the tool call succeeded, do this; otherwise, do that") and repeat work ("check every item in this list" or "keep trying until this succeeds"). **Control flow** is the general name for the small set of keywords that let a program branch and repeat instead of just running straight through. It's genuinely one of the most important ideas in all of programming, because it's the mechanism that turns a fixed script into something that can react to different situations — which, not coincidentally, is also the mechanism at the very heart of what makes an AI agent "agentic" (Module 4).

**`if` / `elif` / `else`** lets your code choose between different actions depending on a condition. `if condition:` runs its indented block only when `condition` evaluates to `True`; you can chain as many `elif` ("else if") branches as you need for additional conditions, and an `else` at the end catches anything not matched by the earlier branches. Only ever *one* of these branches actually runs per pass through the code — Python checks them in order, top to bottom, and stops at the first one that matches.

**`for`** repeats a block of code once for every item in a collection — a list, the characters in a string, a range of numbers. Crucially, a `for` loop knows in advance (or can figure out) exactly how many times it will run, because it's tied to the size of whatever it's looping over. If you loop `for entry in history:` and `history` is empty, the loop body simply never runs — not an error, just zero iterations, which is a detail that trips people up when they expect at least one pass.

**`while`** repeats a block of code for as long as a condition stays `True`, checked fresh before every single pass. Unlike `for`, a `while` loop has no built-in sense of "how many times" — it keeps going until the condition it's watching becomes `False`, however long that takes. This makes `while` the right tool whenever you don't know the number of repetitions in advance, which describes an AI agent's execution loop perfectly: you don't know ahead of time whether the agent will solve the user's goal in 2 steps or 9 steps, so you can't use a `for` loop tied to a fixed count — you need a `while` loop that keeps going until either the goal is met or a safety limit is hit.

This is worth sitting with, because it's the exact mechanical bridge into Module 6: **the "agent loop" described conceptually in Module 4 and Module 6 is, in real code, nothing more exotic than a `while` loop with an `if`/`elif` inside it deciding what to do on each pass.** Everything Module 4 draws as a flowchart with arrows looping back to "Observe" is the same shape as the `while step < max_steps:` loop below, checking its condition again at the top of every iteration.

### A Common Question: "Why not just use a `for` loop up to `max_steps` instead of a `while` loop?"

You could — and in fact, `for step in range(max_steps):` would behave almost identically to the `while` loop below, since both would stop after at most `max_steps` iterations. The reason this course (and most real agent code) reaches for `while` is a matter of emphasis: a `while` loop's condition foregrounds the thing you actually care about — "keep going while we haven't finished and haven't hit the limit" — whereas a `for` loop centered on `range(max_steps)` foregrounds the count, with the "have we finished?" check buried inside the loop body as a separate `if`. Both are valid; this course consistently uses `while` because it more directly mirrors the "keep looping until done" mental model from Module 4's agent loop diagram.

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

*Walkthrough, line by line:* `step = 0` creates a counter, starting at zero, that will track how many loop iterations have happened. `while step < max_steps:` is the loop's condition, re-checked at the very top of every single pass — as soon as `step` reaches `max_steps` (10, by default, from the function's parameter), the condition becomes `False` and the loop stops, control falling through to the `return "Stopped..."` line below it. Inside the loop, `decision = llm.decide_next_step(goal)` stands in for asking the LLM "what should I do next?" (a real version of this calls an actual LLM API, as shown in Module 6.2 and Module 7.3) — the result is a dictionary, exactly like the ones from Lesson 0.1. The `if`/`elif`/`else` chain then branches on `decision["action"]`: if it's `"finish"`, the function immediately `return`s the final answer and the loop never gets to `step += 1` — the function simply ends right there. If it's `"tool_call"`, the code looks up the actual tool function from the `tools` dictionary by name (`tools[decision["tool_name"]]`) and calls it, unpacking the tool's expected arguments with `**decision["tool_input"]` (the double-star spreads a dictionary's key-value pairs out as named arguments — so `**{"city": "Pune"}` becomes the same as writing `city="Pune"` directly in the call). Any other action value falls into the `else` branch, producing a generic error observation instead of crashing. Finally, `step += 1` (shorthand for `step = step + 1`) increments the counter, and the loop goes back to re-check its condition.

**What happens if `decision` is missing the `"action"` key entirely** — say, the LLM returned a malformed dictionary? Then `decision["action"]` would raise a `KeyError` right there in the `if` line, and the whole function would crash rather than gracefully falling into the `else` branch — because the crash happens while trying to even *read* `decision["action"]`, before the comparison against `"finish"` or `"tool_call"` ever gets a chance to run. This is exactly the kind of gap Module 17's validation guidance is written to close: a more defensive version of this code would use `decision.get("action")` instead of `decision["action"]`, the same `.get(...)` pattern from Lesson 0.2, so a malformed decision falls into the `else` branch instead of crashing the entire agent run.

*Explanation:* this is a simplified version of the exact loop you'll build for real in Module 6.2 and Module 7.3 — `while step < max_steps` is the code-level version of the "max step limit" reliability guardrail covered in Module 17.

### Common Mistakes
- **Writing a `while True:` loop with no exit condition.** `while True:` means "the condition is always `True`, forever" — the loop will never stop on its own. This is precisely the "infinite loop" failure mode described in Module 17.1: without something inside the loop that can eventually `return`, `break`, or otherwise stop it, the program (or the agent) simply runs forever, burning time and — in the case of a real agent making LLM calls — burning real money on every wasted iteration. Always give a `while` loop a clear way to stop, and prefer building the stopping condition into the `while` line itself (as `step < max_steps` does) rather than relying on remembering to `break` somewhere deep inside the loop body.
- **Forgetting to update the loop variable.** If you write `while step < max_steps:` but forget the `step += 1` line at the bottom, `step` never changes, the condition is always `True`, and you've accidentally created the same infinite loop as above — just less obviously, since the code *looks* like it should terminate. Always double-check that whatever your `while` condition depends on is actually being changed somewhere inside the loop body.
- **Using `if`/`elif`/`else` when you meant several independent `if` statements.** Because only one branch in an `if`/`elif`/`else` chain ever runs, chaining conditions together with `elif` when you actually wanted to check them all independently (e.g., "log this if the result is an error, AND increment a counter if it's the third failure") is a subtle logic bug. If two conditions are genuinely unrelated and both might need to fire, use two separate `if` statements, not one `if`/`elif` chain.

---

## Lesson 0.4 — Lists and Dictionaries in Depth

### Concept Explanation

Lesson 0.1 introduced lists and dictionaries individually; this lesson is about the pattern that shows up everywhere once you start combining them: **a list of dictionaries**. Almost every piece of "state" or "memory" an agent keeps track of across this entire course — a log of past messages, a running record of completed steps, a batch of retrieved document chunks (Module 9) — is represented this exact way: an ordered sequence (the list) of structured records (each a dictionary). The list gives you order and the ability to grow the collection over time; each dictionary inside it gives you named, self-describing fields instead of having to remember "the third value in this row is the tool name."

It helps to understand *why* this combination specifically, rather than, say, a dictionary of dictionaries or a list of lists. A list of dictionaries is ideal when you have a sequence of similar-shaped "events" or "records" that need to preserve order (step 1 happened before step 2) but where each record itself has multiple named pieces of information (which tool, what input, what result). Compare that to a plain list of lists, like `[[1, "search", "12 laptops found"], [2, "compare", "3 shortlisted"]]` — this technically stores the same information, but you'd have to remember that "position 0 is always the step number, position 1 is always the tool name," which becomes unreadable and error-prone the moment the structure has more than two or three fields, or if two records don't have exactly the same fields. Dictionaries solve this by naming every field, so `entry["tool"]` is self-explanatory in a way `entry[1]` never is.

### A Common Question: "What is that `[h for h in history if ...]` syntax — is that a new kind of loop?"

That's called a **list comprehension**, and it's not a fundamentally new concept — it's a compact way of writing a `for` loop that builds a new list by filtering or transforming an existing one, all on one line. `[h for h in history if h["tool"] == "search"]` means, in longhand: "go through every item `h` in `history`; for each one, check if `h["tool"] == "search"` is `True`; if it is, include `h` in the new list being built; if not, skip it." You could write this exact same logic as a multi-line `for` loop with an `if` inside it and an empty list you `.append()` to — the list comprehension is just a shorter, more idiomatic way to express "filter this list down to the items matching a condition," which is such a common operation (especially for filtering retrieved memory, Module 8-9, or filtering agent history) that Python gives it its own compact syntax.

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

*Walkthrough, line by line:* `history = []` starts with an empty list — a common pattern for anything you plan to build up incrementally as a program runs, exactly mirroring how an agent's history starts empty before its first step. `.append(...)` adds one new item to the end of a list, in place — each call adds one more dictionary, in order, so `history` grows to have two entries after both `.append()` calls. The `for entry in history:` loop then visits each dictionary in turn, in the same order they were added, and the `print` line uses an **f-string** (a string prefixed with `f`, letting you embed `{expressions}` directly inside the text) to pull out and display specific fields from each dictionary — `entry['step']`, `entry['tool']`, and `entry['result']` are three separate dictionary lookups, one per field, happening inside a single formatted string. The final line is the list comprehension explained above, producing a brand-new list containing only the dictionaries where `"tool"` equals `"search"` — in this example, that would be a list containing just the first entry.

**What happens if `history` is empty when the `for` loop runs?** Nothing prints, and no error occurs — a `for` loop over an empty list simply runs its body zero times, then moves on to whatever code comes after it. This "zero is a perfectly normal number of iterations" behavior is worth remembering, because it means you should never assume a `for` loop implies "at least once" — if your code genuinely needs at least one iteration to have happened, you need to check that explicitly (e.g., `if not history: print("No steps yet.")`).

**What if two dictionaries in the list don't have exactly the same keys** — say, one entry has an extra `"cost"` field the others lack? This is completely legal in Python; a list can hold dictionaries with different shapes, and the `for` loop will still iterate over all of them fine. The danger only shows up the moment your code *assumes* every dictionary has a given key and looks it up with square brackets (`entry["cost"]`) — that line would work for entries that have `"cost"` and raise a `KeyError` on the ones that don't. This is exactly the scenario `.get("cost", 0)` (Lesson 0.1 and 0.2's pattern) is designed to guard against.

*Explanation:* the `for entry in history:` loop and the list comprehension (`[h for h in history if ...]`) are both patterns you'll reuse constantly — from replaying an agent's trace, to filtering retrieved memory, to preparing tool schemas.

### Common Mistakes
- **Forgetting that `.append()` modifies the list in place and returns nothing.** A frequent beginner error is writing `history = history.append(...)`, expecting `.append()` to hand back the updated list. It doesn't — `.append()` always returns `None`, because it changes the original list directly rather than creating a new one. After that line, `history` would be overwritten with `None`, silently destroying all your data. The fix is simply to call `history.append(...)` on its own line, without an assignment.
- **Confusing a list index with a dictionary key.** `history[0]` gets the *first item* in the list (by position); `entry["step"]` gets the value under the *key* `"step"` in a dictionary. These use the same square-bracket syntax but mean completely different things depending on whether the thing on the left is a list or a dictionary — mixing them up (e.g., trying `history["step"]` on a list) raises a `TypeError`.
- **Mutating a list while iterating over it.** Adding or removing items from a list *while* a `for` loop is actively iterating over that same list (e.g., calling `history.append(...)` inside `for entry in history:`) produces confusing, sometimes skipped, sometimes duplicated behavior, because the loop is tracking its position by index into a collection that's changing size underneath it. If you need to build a new, modified list based on an existing one, build a separate list instead (exactly what the list comprehension example does — it never modifies `history` itself).

---

## Lesson 0.5 — Classes (Just Enough to Read Agent Code)

### Concept Explanation

Every example so far has kept data (variables, dictionaries) and behavior (functions) as separate things sitting side by side. A **class** is a way to bundle both together into a single reusable blueprint: it defines both what data an object of this kind carries around, and what actions (methods) it knows how to perform on that data. Once you have a class, you can create as many independent **objects** ("instances") from it as you like, each with its own private copy of the data, all sharing the same behavior defined once in the class.

Why does this matter for agent code specifically? Because an agent's state — as Module 16 covers in depth — needs to bundle several related pieces of information together (a task ID, a list of completed steps, a status flag) *and* needs a few operations that always go together with that data (adding a new step, checking if it's done, saving itself to a database). You could do this with a plain dictionary and a handful of separate functions that each take the dictionary as an argument — and in fact, plenty of real code does exactly that — but a class lets you write `state.add_step(...)` instead of `add_step(state, ...)`, which reads more naturally and keeps the data and the operations on it visually grouped together in one place. It also means each agent run can have its own independent `AgentState` object, with no risk of accidentally mixing up which dictionary belongs to which agent run — a mistake that becomes surprisingly easy once you have many agents or many concurrent tasks running (Module 20's discussion of concurrency).

The two words you must be comfortable reading, even if you never write a class from scratch yourself: `__init__` and `self`. `__init__` (two underscores on each side — Python calls these "dunder," short for "double underscore," methods) is a special method that runs automatically, exactly once, the moment you create a new object from the class — it's where you set up that object's starting data. `self` is the name (by convention, always `self`, though technically you could call it anything) that every method inside a class uses to refer back to "this particular object" — it's how `add_step` knows *which* agent's `completed_steps` list to append to, when there might be several `AgentState` objects existing at once.

### A Common Question: "If a dictionary can hold the same data, why not just always use a dictionary instead of a class?"

You often could, and for simple, short-lived data, a dictionary is genuinely simpler and this course uses dictionaries for exactly that reason in most modules. The tipping point toward a class is when you find yourself writing the same handful of operations on that dictionary's shape over and over across your codebase (validating it, updating it, saving it) — at that point, a class lets you define those operations exactly once, attached to the data they operate on, instead of scattered across separate functions that all need to agree on the dictionary's exact shape. Module 16 reaches for a class specifically because agent state has several such recurring operations (`save`, `load`, `add_step`), and bundling them together reduces the chance that one part of the codebase updates the state's shape without updating the functions that operate on it.

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

*Walkthrough, line by line:* `class AgentState:` begins the blueprint, giving it the name `AgentState`. `def __init__(self, task_id: str):` defines what happens the moment someone writes `AgentState(task_id="task-001")` — Python automatically calls `__init__`, passing in the new (still-empty) object as `self` and whatever arguments you supplied (`task_id="task-001"`) as the rest. Inside `__init__`, each `self.something = ...` line attaches a new piece of data directly onto this object: `self.task_id = task_id` stores the id you passed in, `self.completed_steps = []` gives this object its own empty list (crucially, a *new* empty list for every object created — not one shared list accidentally reused across every `AgentState` you ever make), and `self.status = "in_progress"` sets a starting status. `def add_step(self, step: dict):` defines a method — a function that lives inside the class and automatically receives `self` as its first argument whenever you call it on an actual object. Calling `state.add_step({"tool": "search", "result": "done"})` is Python's shorthand for "call `add_step`, automatically passing in `state` as `self`, and the dictionary as `step`" — inside the method, `self.completed_steps.append(step)` then appends that dictionary onto *this specific* object's list.

The two `print` lines at the bottom read `state.status` and `len(state.completed_steps)` using **dot notation** — the same `.` you've already seen with `.append()` and `.get()`, but here accessing data (`status`, `completed_steps`) that lives on this particular object, rather than calling a built-in method. This is the key mental model for reading any class-based code in Module 16: `object.attribute` reads a piece of data belonging to that object, and `object.method(...)` calls a function that operates on that object's data.

**What happens if you create a second `AgentState` object** — say, `state2 = AgentState(task_id="task-002")` — and call `state2.add_step(...)`? Because `__init__` gives every new object its own fresh `self.completed_steps = []`, `state2`'s list starts empty and independent of `state`'s list — adding a step to `state2` has zero effect on `state`. This independence is exactly the benefit classes provide over, say, one shared global dictionary that every part of your program reads and writes: each `AgentState` object is a self-contained, independent bubble of data, which matters enormously once your system is running many agents at once (Module 20).

*Explanation:* `__init__` runs once when you create the object (`AgentState(task_id="task-001")`) and sets up its starting data. `self` refers to "this particular object" — `self.completed_steps` is a list that belongs to this one state object, not shared with other agent runs. You don't need to master classes deeply for this course — just be able to read code shaped like this, since it appears in Module 16's state management examples.

### Common Mistakes
- **Forgetting `self` as the first parameter of a method.** Every method defined inside a class (other than a few special cases beyond this course's scope) must list `self` as its first parameter, even though you never explicitly pass it yourself — Python supplies it automatically based on which object you called the method on. Leaving it off (`def add_step(step: dict):` instead of `def add_step(self, step: dict):`) causes a confusing `TypeError` about too many or too few arguments the moment you try to call the method.
- **Confusing the class itself with an object made from it.** `AgentState` (no parentheses) refers to the blueprint/class itself; `AgentState(task_id="task-001")` (with parentheses and arguments) creates and returns one actual object built from that blueprint. You can create as many independent objects from one class as you like — `state`, `state2`, `state3` could all exist simultaneously, each with its own separate data, from the exact same `AgentState` class definition.
- **Accidentally sharing mutable default data across objects.** A classic, subtle Python bug (rare enough that you likely won't hit it in this course's examples, but worth knowing) happens when a default argument value is itself a mutable object like a list, defined once at class-definition time rather than fresh per object — this is why `self.completed_steps = []` is written inside `__init__` (creating a brand-new list every time an object is made) rather than, say, as a shared default elsewhere. The takeaway for this course: always initialize list/dict attributes fresh inside `__init__`, exactly as shown above.

---

## Lesson 0.6 — JSON: The Universal Format for LLM Data

### Concept Explanation

Here's a real problem that JSON was invented to solve: your Python program has data in memory as dictionaries, lists, strings, and numbers — but the moment you need to send that data somewhere else (over the internet to an LLM provider's server, into a text file, to a different program written in a completely different language like JavaScript or Java), you can't just send "a Python dictionary," because Python's internal memory representation is specific to Python and meaningless to anything else. You need a format that's just plain text — something any programming language, any server, any file can read and write — but that can still represent the same nested structure (objects containing lists containing more objects) that a dictionary can. **JSON** (JavaScript Object Notation) is that format: a simple, universally agreed-upon way of writing structured data as plain text, using curly braces for objects, square brackets for lists, and quotes for strings — which, not coincidentally, looks almost identical to Python dictionary/list syntax, because Python's syntax was itself influenced by exactly this kind of structured-data thinking.

This is precisely why JSON is the backbone of everything from Module 2 onward. When your code sends a request to an LLM provider's API, it has to travel over the internet as plain text — so your Python dictionary gets converted to a JSON string before it's sent. When the LLM's response comes back, it arrives as plain text too — so your code has to convert that JSON string back into a Python dictionary before you can work with it. Tool schemas (Module 7) are described in JSON specifically so that the LLM — which only ever deals in text, remember (Module 2) — can read a tool's definition and produce a matching, correctly-shaped JSON tool call as part of its generated text output. Structured outputs (Module 2.5) work by asking the LLM to generate text that happens to be valid JSON, so your code can reliably parse it afterward rather than trying to extract information from unpredictable free-form prose.

The `json` module in Python's standard library (meaning: it comes built in, no installation needed) provides exactly two functions you'll use constantly, and it helps to think of them as a matched pair, always converting in opposite directions:

- **`json.dumps(python_object)`** — "dump to string." Takes a Python dictionary or list and converts it *into* a JSON-formatted string. Use this whenever you're about to *send* data somewhere as text (over a network request, into a file).
- **`json.loads(json_string)`** — "load from string." Takes a JSON-formatted string and converts it *into* a Python dictionary or list you can actually work with (look up keys, loop over items). Use this whenever you've just *received* text that's supposed to be JSON (an API response body, an LLM's generated output).

### A Common Question: "Isn't a JSON string basically the same as a Python dictionary already? Why do I need to explicitly convert it?"

They *look* almost identical when printed, which is exactly what makes this mistake so easy to make — but they are fundamentally different things to Python. A JSON string is just one long piece of text (a `str`), even if every character in it happens to look like `{"key": "value"}`. Until you run `json.loads(...)` on it, Python has no idea that text has any internal structure at all — it's just a sequence of characters, and trying `my_json_string["key"]` would fail (or worse, silently do something unexpected, since string indexing with `["key"]` isn't even valid Python and would raise a `TypeError`). Only after `json.loads` converts that text into an *actual* Python dictionary object can you use dictionary operations like `["key"]` or `.get(...)` on it. This distinction — "text that looks like data" versus "data" — is one of the most common sources of confusion for beginners working with any API, not just LLM APIs, and it's worth testing yourself on with the edge case below.

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

*Walkthrough, line by line:* `import json` loads Python's built-in JSON module so its functions become available. `tool_call = {...}` builds an ordinary Python dictionary — notice it's even a *nested* dictionary (the `"input"` key's value is itself a dictionary), which JSON handles just as naturally as a flat one. `json.dumps(tool_call)` walks through that dictionary and produces a single string that represents the same structure in JSON's text format — notice the output uses double quotes around every key and string value (JSON requires double quotes specifically; Python is more relaxed and allows single or double, but `json.dumps` always outputs double quotes, matching the JSON standard). The second block starts from the other direction: `llm_response_text` is already a string — written here by hand to simulate what an LLM's generated output text would look like — and `json.loads(llm_response_text)` converts that string into an actual Python dictionary, which is then stored in `parsed`. Only *after* that conversion does `parsed["date"]` work as a normal dictionary lookup, returning the string `"2026-09-01"`.

**What happens if the text isn't valid JSON at all** — say, the LLM's response text is `"Sure, here's the date: 2026-09-01"` (a perfectly normal sentence, but not JSON)? `json.loads(...)` would raise a `json.JSONDecodeError`, and your program would crash right there unless you'd wrapped the call in a `try`/`except` block. This is not a hypothetical edge case — LLMs occasionally produce output that's almost-but-not-quite valid JSON (a missing closing brace, a stray comment), especially under a poorly written prompt (Module 3) or at a high temperature setting (Module 2.4). A more defensive version of the parsing line above would look like:

```python
try:
    parsed = json.loads(llm_response_text)
except json.JSONDecodeError:
    parsed = {"error": "Model did not return valid JSON"}
```

This `try`/`except` pattern — "attempt something risky, and have a planned fallback if it fails instead of crashing" — is the exact code-level mechanism behind Module 17's guidance on validating tool and model outputs before trusting them, and you'll see this same shape recur constantly from here on.

*Explanation:* `json.dumps` converts a Python object *into* a JSON string; `json.loads` converts a JSON string *into* a Python object. Nearly every "structured output" example in Module 2.5, Module 3, and Module 7 depends on this loads/dumps pair — an LLM's JSON output arrives as text, and `json.loads` is what turns it back into a usable Python dictionary your code can act on.

### Common Mistakes
- **Forgetting that an LLM's JSON output is still just text until you call `json.loads` on it.** Trying to use `response["field"]` directly on the raw text (before parsing) will fail — either with a `TypeError` (since string indexing doesn't work that way) or, in some setups, silently return unexpected characters if you use string-slicing syntax by mistake. The rule to memorize: if it came from an LLM, an HTTP response body, or a file, and you want to treat it as structured data, it needs to pass through `json.loads` first.
- **Not handling malformed JSON.** As shown above, a model can occasionally produce invalid JSON, especially in edge cases or under time/token pressure. Production code always wraps `json.loads` in a `try`/`except` (this connects directly to Module 17's validation guidance) rather than assuming every model response will be perfectly formed.
- **Confusing Python's `None`/`True`/`False` with JSON's `null`/`true`/`false`.** Python and JSON use different spellings for these special values (Python capitalizes `None`, `True`, `False`; JSON lowercases `null`, `true`, `false`). The `json` module handles this translation automatically in both directions, so you rarely need to think about it — but if you ever hand-write a JSON string yourself (rather than using `json.dumps`) and accidentally write `True` instead of `true`, you've produced invalid JSON that `json.loads` (or any other JSON parser) will reject.

---

## Lesson 0.7 — Calling a Web API

### Concept Explanation

Almost nothing interesting an AI agent does happens entirely inside your own computer's memory. Getting an LLM's response, searching the live web, checking a real weather service, looking up a database record over a network — every one of these, underneath, is your program sending a message to some other computer (a **server**) somewhere on the internet, and waiting for that server to send a message back. **HTTP** (HyperText Transfer Protocol) is the standard set of rules that governs how those messages are structured and exchanged, and it's the same protocol your web browser uses every time it loads a page — an "API call" is just your Python code doing programmatically what a browser does when you type a URL and hit enter.

An HTTP request has a few essential parts worth understanding, because you'll see all of them in the example below. The **URL** (or "endpoint") is the address of the specific server and resource you're talking to. The **method** describes the kind of action you're requesting — `GET` typically means "give me some data" (like loading a webpage), while `POST` typically means "here's some data, please process it" (like submitting a form, or — in this course's most common case — sending a prompt to an LLM and asking it to generate a response). **Headers** carry metadata about the request that isn't the main content itself — most importantly for this course, **authentication**: proving to the server who you are and that you're allowed to make this request, usually via an API key. The **body** (for `POST` requests) carries the actual data/payload you're sending — in this course, almost always a JSON object (tying directly back to Lesson 0.6).

When the server responds, you get back a **status code** (a number summarizing what happened — `200` means success, codes starting with `4` typically mean something was wrong with your request, codes starting with `5` typically mean something went wrong on the server's end) and a **response body**, which — for essentially every API this course touches — arrives as a JSON string that you then need to `json.loads` (or, more conveniently, call `.json()` directly on the response object, which does the parsing for you) before you can work with it as a Python dictionary.

Python's `requests` library (not part of the standard library — it needs to be installed via `pip`, Lesson 0.8) is the de facto standard way to make these HTTP calls, because it wraps all of the above into a small handful of simple function calls, hiding a lot of low-level networking detail you don't need to think about for this course.

### A Common Question: "Why does an LLM call need to be a network request at all — isn't the 'brain' just running locally?"

For the vast majority of this course's examples (and the vast majority of real-world agent systems), no — the LLM itself runs on the provider's own servers (Anthropic's, OpenAI's, or similar), not on your machine. Your code's job is simply to package up your prompt as a JSON request body, send it over the network to that provider's API endpoint, and wait for their servers to run the actual model and send back a JSON response containing the generated text. This is why every single LLM interaction in this course, no matter how it's later wrapped in agent logic, ultimately bottoms out in exactly the kind of `requests.post(...)` call shown below. (It is possible to run some smaller models entirely on your own machine — this is called "local inference" — but that's outside this course's scope; assume every LLM call here is a network request to a hosted API.)

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

*Walkthrough, line by line:* `import requests` loads the library (after it's been installed, Lesson 0.8). `requests.post(...)` sends an HTTP `POST` request to the given URL — notice this single function call bundles together everything described above: the URL as the first argument, `headers={...}` carrying the `Authorization` header (the `"Bearer YOUR_API_KEY"` format is a common convention many APIs use — "Bearer" simply meaning "the bearer of this token is authorized"), and `json={...}` telling `requests` to take this Python dictionary, convert it to a JSON string for you automatically (you don't even need to call `json.dumps` yourself — `requests` does it internally when you use the `json=` parameter specifically), and send it as the request body with the correct `Content-Type` header set automatically as well. The whole call blocks (pauses your program) until the server responds, at which point everything about that response — status code, headers, body — is packaged into the `response` object. `response.json()` is a convenience method that reads the response body text and runs `json.loads` on it for you in one step, handing back a ready-to-use Python dictionary, which is stored in `data`. Finally, `data["content"]` reads a field out of that dictionary — exactly the same dictionary-lookup operation from Lesson 0.1, just applied to data that arrived over the network instead of being written by hand.

**What happens if the request fails** — wrong API key, no internet connection, the server is down? The `requests.post(...)` call itself will still typically complete and return a `response` object (rather than crashing outright) as long as *some* response came back from the server, but `response.status_code` would be something other than `200` (commonly `401` for a bad API key, or `429` for a rate limit — you've made too many requests too quickly), and `response.json()` might raise an error if the error response body isn't valid JSON, or might successfully parse into a dictionary describing the error rather than the answer you wanted. This is exactly why the Common Mistakes section below insists on checking `response.status_code` before assuming `data["content"]` will actually be there — code that skips this check works fine during testing (when everything succeeds) and then crashes confusingly in production the first time a request legitimately fails, which happens far more often than beginners expect once real usage volume kicks in.

*Explanation:* `headers` carries authentication (Module 21.4 — never hardcode a real key like this in shared code; use environment variables instead); `json=...` tells `requests` to send your dict as a JSON request body; `.json()` parses the JSON response straight into a Python dict, combining Lessons 0.6 and 0.7 into the exact pattern every LLM API call in this course follows.

### Common Mistakes
- **Hardcoding API keys directly in source code.** Writing your actual, real API key as a literal string (as the placeholder `"Bearer YOUR_API_KEY"` above does, deliberately, for illustration) means that key ends up saved in your code — and if that code is ever committed to a shared or public repository (Module 21.4 covers this in depth), anyone who can see the code can see and misuse your key, potentially running up large bills or accessing data on your behalf. The standard fix is to store secrets as **environment variables** (values set outside your code, in your operating system or a `.env` file) and read them with `os.environ.get("API_KEY")`, so the actual secret value never appears anywhere in your source code or version-control history.
- **Not checking `response.status_code` before assuming the call succeeded.** A failed request (rate limit, network error, invalid input, expired API key) still returns a `response` object — it does not automatically raise a Python exception the way, say, a `KeyError` does. Code that jumps straight to `response.json()["content"]` without first checking `response.status_code == 200` (or using `response.raise_for_status()`, a `requests` convenience method that raises an exception automatically on error status codes) will produce a confusing crash on a `KeyError` or similar deep inside your logic, far from the actual root cause, whenever a request legitimately fails.
- **Forgetting that network calls can be slow or can time out.** Unlike calling a Python function that runs entirely in memory, a network request depends on another computer, an internet connection, and everything in between — any of which can be slow or fail entirely. `requests` calls can hang indefinitely by default if a server never responds; passing a `timeout=` argument (e.g., `requests.post(..., timeout=30)`) ensures your program gives up and raises an error after a reasonable wait instead of freezing forever. This becomes directly relevant in Module 17's discussion of timeouts as a reliability guardrail.

---

## Lesson 0.8 — `pip` and Virtual Environments

### Concept Explanation

Everything you've written so far in this module uses only Python's built-in capabilities (or, in the case of `requests`, a library assumed to already be installed) — but almost nothing interesting in the rest of this course works that way. Talking to an LLM provider typically means using their official SDK (software development kit — a pre-built library that wraps the raw HTTP calls from Lesson 0.7 into friendlier function calls); working with a vector database (Module 9) means installing a client library for whichever database you choose; even reading a `.env` file of secrets (mentioned in Lesson 0.7) typically uses a small helper library. None of these come built into Python — they're written by other people or companies and published for anyone to download and use. **`pip`** is Python's built-in package manager: the tool that downloads a library's code from the internet (from a central repository called PyPI, the Python Package Index) and installs it onto your machine so your own code can `import` it.

This raises an obvious question, though: what happens when two different projects on your machine need *different, incompatible versions* of the same library? Project A might need version 1.0 of some library, while Project B — started a year later — needs version 3.0, and those two versions might behave differently enough that code written for one breaks under the other. If `pip install` always installed libraries globally (system-wide, shared across every Python project on your machine), you could never satisfy both projects at once — installing what Project B needs would break Project A. A **virtual environment** solves this by creating an isolated, self-contained Python installation just for one project: when you activate it, `pip install` and `import` only see the libraries installed *inside that environment*, completely separate from your system's global Python and from any other project's virtual environment. This means Project A and Project B can each have their own environment, each with whatever exact library versions they individually need, with zero conflict between them.

It's worth being precise about what "activating" an environment actually does, since it's easy to treat it as a magic incantation: activating a virtual environment temporarily changes your terminal session so that the `python` and `pip` commands you type point at the isolated environment's copies of those tools, instead of your system's global ones. This effect only lasts for that terminal session (or until you explicitly deactivate it) — opening a brand new terminal window starts fresh, pointed back at the global Python, until you activate the environment again there too. This is precisely why the Common Mistakes section below calls out forgetting to reactivate in a new terminal as a frequent, confusing trap: the commands look identical, and the state that changed (which Python interpreter you're actually running) is invisible unless you know to check.

### A Common Question: "Do I need a separate virtual environment for every single project in this course?"

In practice, one virtual environment per *project folder* (not per module, and not one giant environment for everything) is the standard convention, and it's what the Hands-On Projects in Module 23 assume — each project directory gets its own `venv`, created and activated before you `pip install` anything for that specific project. This keeps each project's exact dependency versions recorded and reproducible independently (via `requirements.txt`, shown below), which matters increasingly as the projects grow — the simple Personal Task Assistant (Project 1) needs far fewer libraries than the Multi-Agent Content Company (Project 5), and there's no benefit to forcing them to share one dependency set.

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

*Walkthrough, line by line:* `python -m venv venv` runs Python's built-in `venv` module (the `-m` flag tells Python "run this installed module as a program") to create a new virtual environment; the second `venv` is just the folder name being created — you could name it anything, but `venv` is the near-universal convention. This creates a new folder on disk containing a self-contained copy of the Python interpreter and an empty space for libraries. The two `activate` lines are alternatives for different operating systems — you only ever run *one* of them, matching whatever terminal/shell you're using — and both do the same conceptual thing described above: reroute the current terminal session's `python`/`pip` commands to this environment's isolated copies. Once activated, `pip install requests python-dotenv chromadb` downloads and installs three separate libraries in one command (space-separated) — `requests` for HTTP calls (Lesson 0.7), `python-dotenv` for reading secrets from a `.env` file (referenced in Lesson 0.7's Common Mistakes), and `chromadb` as an example vector database client (Module 9) — all landing inside this project's isolated environment, invisible to any other project on your machine. Finally, `pip freeze > requirements.txt` lists every currently installed library and its *exact* version number, writing that list into a file called `requirements.txt` — this file is what lets someone else (or you, on a different machine, or after reinstalling) recreate the identical environment later by running `pip install -r requirements.txt`, rather than having to remember and reinstall everything by hand.

**What happens if you skip creating a virtual environment and just run `pip install` directly?** It will very likely still work — `pip` will install the library into your system's global Python installation, and your code will run fine, right up until the day you have a second project needing an incompatible version of the same library, at which point you'll hit exactly the conflict this lesson describes, often with a confusing error message that doesn't obviously point back to "two projects want different versions of the same thing." Using a virtual environment from the very first project, even a tiny one, is one of those habits that costs almost nothing up front and prevents a genuinely confusing class of bug later.

*Explanation:* activating a virtual environment before running `pip install` keeps this project's libraries separate from every other Python project on your machine — this matters once you're juggling multiple course projects (Module 23) with potentially different library versions.

### Common Mistakes
- **Installing packages globally (without activating a virtual environment first).** This works at first — the library installs, `import` succeeds, everything seems fine — but eventually causes version conflicts between unrelated projects on the same machine, exactly as described above. The symptom, when it eventually appears, is often a mysterious `ImportError` or a function behaving differently than documentation suggests, because a different project installed a newer or older version of the same library globally in the meantime.
- **Forgetting to activate the virtual environment in a new terminal session.** Every time you close and reopen your terminal (or open a new tab/window), you start back at the global Python — the activation from an earlier session does not persist. Running `pip install` in a fresh, unactivated terminal will silently install into the wrong place (your global Python, not the project's `venv`), and you may not notice until `import` mysteriously fails later inside the actual project, because the library was installed somewhere your project's environment doesn't look.
- **Committing the `venv` folder itself to version control.** The virtual environment folder can contain thousands of files and is entirely reproducible from `requirements.txt` on any machine — committing it to Git (Module 20, 21) bloats your repository for no benefit and can even leak machine-specific file paths. The standard practice is to add `venv/` to a `.gitignore` file and commit only `requirements.txt`, letting anyone recreate the exact same environment with `pip install -r requirements.txt` instead of downloading your literal folder.

---

## Key Takeaways
- This course's code examples rely on a small, consistent slice of Python: variables/dicts/lists, functions, `if`/`for`/`while`, simple classes, JSON, HTTP requests, and basic dependency management.
- Dictionaries and JSON are the connective tissue of everything from here on — tool schemas, LLM responses, agent state, and memory records are all, at heart, nested dictionaries, and the `json.dumps`/`json.loads` pair is what moves data between "Python object" and "portable text" whenever it crosses a network boundary.
- The `while` loop pattern in Lesson 0.3 *is* the agent loop from Module 6 — once you're comfortable with it, Module 6 will feel like a direct continuation, not a new concept. Likewise, the `.get(key, default)` defensive-lookup pattern introduced in Lesson 0.1-0.2 is the earliest, simplest form of the validation and error-handling discipline that Module 17 builds into a full chapter.
- Every "edge case" called out in this module's walkthroughs — a missing dictionary key, an empty list, malformed JSON, a failed network request — is not a hypothetical footnote. Each one maps directly onto a real reliability concern this course returns to later (Module 17's hallucination/tool-failure/validation guidance), so getting comfortable reasoning about "what if this input isn't what I expect?" now pays off directly later.

### Exercise
Write a Python function `summarize_tasks(tasks: list[dict]) -> str` that takes a list of task dictionaries (each with `"description"` and `"done"` keys) and returns a string like `"2 of 5 tasks done."` Test it with a small hand-written list. Then test it again with an *empty* list — what should it print, and does your function handle that case sensibly without crashing?

### Challenge
Write a small script that: (1) defines a Python dict representing a fake "tool schema" (name, description, input schema, matching the shape from Module 7.2), (2) converts it to a JSON string with `json.dumps`, (3) parses it back with `json.loads`, and (4) prints the tool's `"description"` field from the round-tripped result. As a follow-up, deliberately corrupt the JSON string (delete a closing brace) and wrap your `json.loads` call in a `try`/`except` so the script prints a friendly error message instead of crashing.

### Knowledge Check
1. Why does almost every tool schema, LLM response, and agent state object in this course end up looking like a Python dictionary?
2. What's the difference between `json.dumps` and `json.loads`, and when would you use each? What specifically goes wrong if you try to use dictionary lookup syntax on a JSON string you haven't run `json.loads` on yet?
3. Why does a `while` loop need an explicit exit condition, and what agent failure mode (from later in the course) happens when it doesn't?
4. In the `run_agent` example from Lesson 0.3, what would happen if `decision` were looked up with `decision["action"]` instead of `decision.get("action")`, and the LLM returned a dictionary missing that key entirely?

Continue to **[00-Course-Overview.md](00-Course-Overview.md)** to begin the main course.
