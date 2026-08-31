# Module 17 — Agent Reliability

### Difficulty
Advanced

### Learning Objectives
- Understand common agent failure modes: hallucination, tool failures, infinite loops, bad planning, wrong tool selection, context overload.
- Understand *why* each failure mode happens mechanically, not just what it looks like.
- Learn concrete mitigations: retries, timeouts, guardrails, validation, fallback models, human approval.

### Prerequisites
Modules 1–16.

---

## Lesson 17.1 — Common Failure Modes

### Why This Module Matters More Than It Might Seem To

Every module up to this point has focused on building capability — giving an agent tools, memory, planning, multi-agent coordination. This module is about the opposite instinct: assuming that everything you built *will* fail sometimes, and designing for that reality up front rather than bolting on fixes after a production incident. This isn't pessimism for its own sake — it's a direct consequence of two things you already know from earlier modules: LLMs are non-deterministic samplers over token probabilities (Module 2.4), not verified-fact machines, and external systems (APIs, networks, databases) are never 100% reliable no matter who built them. Combine those two facts and you get a mathematical certainty: an agent that runs many steps, calling many tools, over many requests, *will* eventually hit some kind of failure. The only real question is whether your system was designed to notice and handle it gracefully, or whether it silently produces a wrong answer, hangs forever, or crashes in a way nobody discovers until a user complains.

| Failure Mode | What It Looks Like | Root Cause |
|---|---|---|
| **Hallucination** | Agent states false information confidently | LLMs generate plausible-sounding text, not verified fact; worse without grounding (RAG) |
| **Tool failures** | API timeout, malformed response, rate limit hit | External systems are unreliable; network issues, quota limits |
| **Infinite loops** | Agent keeps calling the same tool or repeating steps without progress | No termination condition, or model gets "stuck" reasoning in circles |
| **Bad planning** | Agent's plan misses key steps or pursues an ineffective strategy | Vague goal, insufficient context provided to the planner, or planning model limitations |
| **Incorrect tool selection** | Agent calls the wrong tool for the situation | Ambiguous/overlapping tool descriptions, insufficient examples in the prompt |
| **Context overload** | Agent's response degrades as conversation/context grows very long | Context window filled with noise, causing "context rot" — the model loses track of what matters |

### Going Deeper on Each Root Cause

**Hallucination is not a bug that will eventually get "fixed" by a better model — it's a structural consequence of how next-token prediction works, and understanding this precisely is what tells you which mitigations can actually help and which are wishful thinking.** Recall from Module 2 that an LLM generates each token by sampling from a probability distribution over "what token is statistically likely to come next, given everything before it." When you ask a model a factual question it has strong, well-represented training data for ("what's the capital of France?"), the correct answer genuinely is the overwhelmingly most probable continuation, so the model reliably produces it. But when you ask about something rare, something outside its training data, or something that requires precise recall of a specific number or date, the model is still doing exactly the same thing — predicting a *plausible-sounding* continuation — except now "plausible-sounding" and "factually correct" can come apart. The model has no internal mechanism that says "I'm not sure, let me flag this" by default; fluent, confident-sounding text is what the training process optimized it to produce, regardless of whether the specific facts inside that fluent text are accurate. This is why hallucination tends to get *worse*, not better, exactly in the situations where you'd most want the model to admit uncertainty — obscure facts, very specific numbers, and anything genuinely outside its training data all produce the same fluent, confident *tone* as a well-supported fact.

**Tool failures are a reliability problem you inherit, not one you create.** Every external system your agent depends on — a weather API, a payment processor, your own database — has its own uptime, its own rate limits, its own occasional bad day. Your agent's tool-calling code is now a client of all of those systems simultaneously, which means your agent's *effective* reliability is bounded by the reliability of the least reliable tool it depends on, multiplied across however many tool calls one task requires. A tool with 99.9% uptime sounds great in isolation, but a task calling that same tool 20 times has roughly a 2% chance of hitting at least one failure somewhere in that sequence — reliability problems compound across a chain of dependent calls in a way that's easy to underestimate if you only think about one call at a time.

**Infinite loops happen because the agent loop (Module 6) has no built-in sense of "progress" — it only knows how to decide a next action, not whether that action is actually moving it closer to the goal.** If the model's chosen strategy genuinely doesn't work (a search query that returns no useful results, a tool that keeps returning an error the model doesn't know how to interpret), nothing about the loop's basic structure stops it from trying essentially the same thing again, especially if the model's training gives it no better alternative move to make. This is fundamentally different from a traditional software bug (an actual infinite `while True` with no exit) — the loop conceptually *can* terminate, but the model keeps choosing not to terminate it, because from its local, step-by-step perspective, trying again looks like a reasonable next move each time, even though the *sequence* of tries isn't converging on anything.

**Bad planning is an information problem more often than it's a "the model isn't smart enough" problem.** A plan is only as good as what the planner actually knows when it makes the plan (Module 11). If a goal is vague ("improve our marketing"), or the planner is missing key constraints (a budget limit, a deadline, an existing commitment that conflicts with a proposed step), it will produce a plan that's internally coherent and reasonable-*looking* but wrong for the actual situation — not because the model reasoned poorly, but because it was never given the information it would have needed to reason correctly. This distinction matters for how you fix it: the fix for most bad-planning problems is providing better context or asking clarifying questions up front (Module 11), not simply hoping a "smarter" model will somehow infer missing constraints it was never told about.

**Incorrect tool selection is, in the large majority of real cases, a description-writing problem, not a reasoning problem.** Recall from Module 7.2 that a tool's description is the *only* signal the model has for deciding when that tool applies — the model has no access to your tool's actual source code, only the plain-English (or JSON schema) description you wrote. If two tools have descriptions that could both plausibly apply to the same kind of request ("search_web" and "search_internal_docs," both described loosely as "search for information"), the model has no way to reliably distinguish them beyond whatever subtle wording differences you provided, and it will sometimes guess wrong. This is fixable, but the fix lives in how you write and scope your tools, not in trying to make the model "better at intuiting" an ambiguity you created.

**Context overload happens because the transformer architecture underlying LLMs doesn't treat every token in its input with equal, undiluted attention as the input grows — practically speaking, the more the context window fills with tangential or repetitive content, the harder it becomes for the model to reliably locate and weight the specific pieces of information that actually matter for the current decision.** This is why a long agent run that accumulates a huge, unfiltered history of every tool call and every intermediate thought tends to degrade in quality over time, even though nothing was ever technically over the hard token limit (Module 2.2) — the problem isn't running out of room, it's that the useful signal becomes a smaller and smaller fraction of an increasingly noisy whole.

---

## Lesson 17.2 — The General Solution Loop

### Concept Explanation

Given that failures are a certainty rather than an edge case, the practical question becomes: what's the right *order of operations* for responding to one when it happens? A tempting but wrong instinct is to treat every failure identically — either always retrying blindly, or always giving up and escalating immediately. Neither extreme is efficient. The right approach is a graduated response that starts cheap and only escalates to something more expensive or disruptive once cheaper responses have genuinely been tried and failed.

```mermaid
flowchart TD
    P([Problem]) --> D{Detection:<br/>how do we know<br/>something's wrong?}
    D --> F[Fallback:<br/>don't just fail outright]
    F --> R{Retry:<br/>try again,<br/>possibly differently}
    R -- succeeds --> Done([Continue normally])
    R -- fails repeatedly --> Alt[Alternative Strategy:<br/>a genuinely different approach]

    style P fill:#fee2e2,stroke:#dc2626
    style Done fill:#dcfce7,stroke:#16a34a
    style Alt fill:#fef3c7,stroke:#d97706
```

**How to read this graph:** this is a decision funnel, not a straight pipeline — notice "Retry" has two exits. Most failures should resolve at the cheap "Retry" step (a network blip fixed by trying again); only failures that survive repeated retries should escalate all the way to "Alternative Strategy," which is the most expensive, most disruptive branch. A common mistake this graph makes visible: if your code jumps straight from "Problem" to "Alternative Strategy" without ever trying the cheaper Detection → Fallback → Retry path first, you're over-reacting to what might have been a one-off blip. Equally, the reverse mistake is just as costly: if your code keeps looping through "Retry" indefinitely without ever reaching "Alternative Strategy," you've built exactly the infinite-loop failure mode this module is trying to prevent — the funnel only does its job if both the cheap path *and* the escalation path are actually wired up, not just one of them.

### A Common Question

**"How do you decide whether a specific failure belongs on the cheap 'Retry' path versus needing to jump straight to 'Alternative Strategy'?"** The deciding factor is whether the failure is *transient* (caused by something temporary and external — a network blip, a momentarily overloaded server, a rate limit that will reset shortly) or *persistent* (caused by something that won't change no matter how many times you try — malformed input, a tool that fundamentally can't do what was asked, a request that violates the target system's rules). Retrying a transient failure is rational, because the exact same call might simply succeed the second or third time. Retrying a persistent failure is wasted effort — the failure will reproduce identically every single time, because nothing about the situation actually changed between attempts. This distinction is exactly what the `TransientError` exception type in Lesson 17.3's code example is meant to capture: it's a deliberate signal, set by whoever wrote the tool's error-handling code, distinguishing "worth retrying" from "don't bother."

---

## Lesson 17.3 — Concrete Mitigations

### Retries
```python
def call_tool_with_retry(tool_fn, tool_input, max_retries=3):
    for attempt in range(max_retries):
        try:
            return tool_fn(**tool_input)
        except TransientError as e:
            if attempt == max_retries - 1:
                return {"error": f"Failed after {max_retries} attempts: {e}"}
            time.sleep(2 ** attempt)  # exponential backoff
```

*Explanation, line by line:* the `for attempt in range(max_retries)` loop bounds the number of attempts — this itself is a small instance of the "guard against infinite retries" principle from the general solution loop above. The `except TransientError` clause is doing important, deliberate work: by only catching this specific exception type (rather than a bare `except:`), the code ensures that a *non*-transient error (say, a `ValueError` from malformed input) propagates up immediately instead of being silently retried three times for no benefit — this is the code-level enforcement of the "transient vs. persistent" distinction from the question above. `time.sleep(2 ** attempt)` implements **exponential backoff**: on the first retry you wait 1 second (`2**0`), on the second 2 seconds (`2**1`), on the third 4 seconds (`2**2`), and so on. This growing delay matters for a subtle but important reason: if the reason for the failure is that the external service is *overloaded* (too many requests hitting it at once), retrying immediately at full speed makes the overload worse, potentially for every other client of that service too, not just you — spacing out retries with increasing delays gives the struggling service breathing room to recover, which is exactly why virtually every production system that talks to external APIs uses some form of this pattern rather than a fixed, tight retry loop.

*Why it matters:* transient failures (network blips, rate limits) often succeed on retry; exponential backoff avoids hammering a struggling service. Non-transient errors (bad input) should NOT be blindly retried — they'll fail identically every time.

### Timeouts
```python
result = call_tool_with_timeout(tool_fn, tool_input, timeout_seconds=10)
```

**Why a timeout is a distinct concern from a retry:** a retry handles the case where a call *fails* (returns an error). A timeout handles the different, arguably more dangerous case where a call never returns *anything at all* — it just hangs, waiting on a network response or a slow downstream system that may never come back. Without an explicit timeout, your agent's entire execution loop (Module 6) is at the mercy of whatever the slowest possible response from any tool it might call happens to be — a single hung network connection can stall an otherwise-perfectly-functioning agent indefinitely, burning wall-clock time and, in a multi-user system, potentially tying up server resources that other users' requests need. Setting `timeout_seconds=10` here means: "if this call hasn't returned within 10 seconds, treat that as a failure and move on" — converting an open-ended, unbounded wait into a bounded, predictable one that your retry/fallback logic can then handle like any other failure.

*Why it matters:* without a timeout, one hung tool call can stall the entire agent run indefinitely.

### Guardrails

Pre- and post-execution checks that constrain agent behavior:

```text
Input Guardrail:  Reject tool calls with obviously malformed/dangerous input
                  (e.g., a "delete_file" call targeting a system directory)
Output Guardrail: Reject/flag responses containing disallowed content
                  (e.g., PII leakage, policy violations)
```

**What makes a guardrail structurally different from validation (below):** a guardrail is a *policy* check — it's not asking "is this technically well-formed," it's asking "is this allowed at all, regardless of whether it's well-formed." A perfectly well-formed, syntactically valid `delete_file(path="/etc/passwd")` call is exactly the kind of thing a guardrail exists to catch — there's nothing malformed about that tool call's structure, but it's an action you never want an agent to actually execute. Guardrails typically sit as a hard-coded check *between* the model's decision and the actual tool execution (an input guardrail) or between the tool's/model's output and whatever surfaces it to a user (an output guardrail) — they are deliberately simple, deterministic, rule-based checks, precisely because you want them to be 100% reliable and auditable, unlike the probabilistic model they're constraining. This connects directly to Module 21's security guidance: guardrails are one of the primary defenses against a compromised or manipulated agent taking a genuinely harmful action, because they don't rely on the model "deciding" to behave safely — they enforce it externally, regardless of what the model decided.

### Validation
```python
def validate_tool_output(output: dict, expected_schema: dict) -> bool:
    # Check output actually matches the expected structure before trusting it
    ...
```

**Why this is a distinct step from a guardrail:** validation isn't asking "is this allowed" — it's asking "is this the *shape* I expected." An LLM asked to produce structured JSON output (Module 2.5) will, occasionally, produce output that's subtly malformed: a missing field, a string where a number was expected, an extra unexpected key, or outright invalid JSON syntax. If your downstream code blindly does `result["confidence"]` on output that never actually contained a `confidence` key, you get a crash — or worse, if you're using `.get("confidence", "high")` with a lenient default, you get a silent, wrong assumption baked into your agent's behavior with no visible error at all. Validating output against an explicit expected schema *before* trusting it converts an unpredictable, possibly-silent failure into a predictable, catchable one — exactly the same principle behind Module 16's schema-versioning discussion for checkpoints, applied here to tool and LLM outputs instead of persisted state.

*Why it matters:* never assume a tool (especially an LLM-generated structured output) returns exactly the expected shape — validate before using it downstream, since malformed output propagating silently causes hard-to-trace bugs later.

### Fallback Models
```text
Primary model unavailable / rate-limited / fails repeatedly
   ↓
Fallback: retry the same request against a secondary model provider or a
smaller/cheaper model, possibly with a degraded-but-functional response
```

**Why "degraded-but-functional" is the right target, not "identical quality":** the point of a fallback model isn't to pretend nothing went wrong — it's to avoid a total outage when your primary model is unavailable. A smaller or different fallback model may give a noticeably lower-quality response than your primary model would have, but a slightly worse answer delivered to the user is very often preferable to a hard failure and no answer at all, especially for time-sensitive or customer-facing agents. Designing for this gracefully typically means your application code is written against a stable interface (a function that takes a prompt and returns a response) that can be pointed at *either* provider, rather than being tightly coupled to one specific provider's API in a way that makes swapping providers under failure conditions a scramble rather than a planned, tested code path.

### Human Approval (see Module 18 for full treatment)
For high-risk actions (sending money, deleting data, sending external emails), insert a mandatory human approval gate rather than letting the agent act autonomously. This is, in a sense, the ultimate fallback: when the cost of a wrong autonomous decision is high enough, the most reliable mitigation isn't a smarter retry strategy or a better guardrail — it's simply not letting the fully autonomous loop make that particular call unsupervised in the first place.

---

## Lesson 17.4 — Specific Fixes Per Failure Mode

| Failure Mode | Mitigation |
|---|---|
| Hallucination | Ground answers in retrieval (RAG, Module 10); instruct the model to say "I don't know" when uncertain; add a fact-checking/critic step (Module 12, 15) |
| Tool failures | Retries with backoff, timeouts, structured error observations fed back to the agent (Module 7) |
| Infinite loops | Hard max-step limits (Module 6); detect repeated identical actions and force a different strategy or stop |
| Bad planning | Clarify/decompose the goal further (Module 11); add a plan-review step before execution begins |
| Incorrect tool selection | Sharpen tool descriptions; reduce overlapping tools; add few-shot examples of correct tool choice (Module 3, 7) |
| Context overload | Summarize/compress older history; use RAG instead of dumping full documents; trim irrelevant tool results before re-injecting (Module 2, 10) |

### Practical Example — Loop Detection

```python
def detect_stuck_loop(history: list[dict], window=3) -> bool:
    if len(history) < window:
        return False
    recent = history[-window:]
    return all(
        step["tool"] == recent[0]["tool"] and step["input"] == recent[0]["input"]
        for step in recent
    )

# In the agent loop:
if detect_stuck_loop(state["history"]):
    # Force a different approach instead of repeating the same failing action
    inject_message(state, "You've repeated the same action. Try a different approach or ask for help.")
```

*Explanation, line by line:* `recent = history[-window:]` grabs just the last `window` (here, 3) entries from the agent's history — the function deliberately only looks at a small recent slice, not the entire history, because "stuck in a loop *right now*" is about recent, consecutive behavior, not something that happened many steps ago. The `all(...)` check then verifies that every one of those recent steps used the *exact same* tool with the *exact same* input as the first one in the window — if all three match, that's a strong signal the agent is repeating itself rather than making progress. This is exactly a code-level implementation of the "infinite loop" root cause identified in Lesson 17.1: the agent loop itself has no innate sense of "am I progressing," so this function manually supplies that missing sense by checking for a specific, detectable symptom of *not* progressing.

*Explanation, in one sentence:* this checks whether the last N actions were identical — a strong signal the agent is stuck. Instead of letting it loop forever (burning cost with no progress), it's nudged toward a different strategy or an explicit stop.

### Going Further: Detecting Near-Duplicate, Not Just Identical, Repeats

The exact-match check above catches the most obvious case — literally repeating the same tool call — but it misses a subtler and arguably more common variant: an agent that keeps trying *semantically equivalent* actions phrased slightly differently each time, which never trips an exact-match comparison. Consider a research agent that searches `"best laptops under 80000"`, gets a disappointing result, then tries `"top laptops under 80,000 rupees"`, then `"good laptops budget 80000"` — three different strings, so `recent[0]["input"] == step["input"]` is `False` every single time, and `detect_stuck_loop` never fires, even though the agent is functionally stuck asking the same question in slightly different words and getting equally unhelpful results each time.

A second heuristic addresses this by checking *semantic* similarity (Module 9's embeddings, applied here to actions instead of documents) rather than exact string equality:

```python
def detect_semantic_loop(history: list[dict], window=3, similarity_threshold=0.9) -> bool:
    if len(history) < window:
        return False
    recent = history[-window:]
    if not all(step["tool"] == recent[0]["tool"] for step in recent):
        return False  # different tools entirely — not this kind of loop

    inputs_as_text = [str(step["input"]) for step in recent]
    embeddings = [embed_text(text) for text in inputs_as_text]
    similarities = [
        cosine_similarity(embeddings[0], embeddings[i])
        for i in range(1, len(embeddings))
    ]
    return all(sim >= similarity_threshold for sim in similarities)
```

*Explanation:* this reuses the exact embedding and cosine-similarity mechanism from Module 9.2 — instead of comparing search results to a user's question, it compares the agent's own successive *inputs* to each other. If three consecutive search queries, despite being worded differently, all land above a high similarity threshold (0.9 here — very close in meaning), that's evidence the agent is circling the same underlying idea without making genuine progress, even though no two strings were literally identical. In practice, production systems often run both checks together: the cheap exact-match check as a fast first pass, and the more expensive embedding-based check as a fallback for catching the sneakier, reworded-repetition case.

### Key Takeaways
- Reliability problems are not edge cases — they are the default state of any nontrivial agent system and must be designed for from the start, precisely because non-deterministic models and unreliable external systems make failure a statistical certainty over enough steps and enough requests.
- Each failure mode has a distinct root cause and requires a distinct mitigation — there's no single fix-all, and understanding *why* a failure mode happens (Lesson 17.1) is what tells you which mitigation will actually address it versus just paper over the symptom.
- Detection before mitigation: you can't fix what you don't notice (logging, monitoring, and loop detection are prerequisites to any fix) — and detection itself often needs more than one heuristic, since a naive check (exact-match repetition) can miss a real problem hiding in a slightly different form (semantic repetition).

### Common Mistakes
- Adding retries for errors that will deterministically fail again (e.g., malformed input) — wastes calls without fixing anything, and can even make things worse by masking a persistent bug behind a delay that makes it look like the system is "still working."
- Treating hallucination as unfixable — grounding, uncertainty instructions, and verification steps meaningfully reduce (though don't eliminate) it; the mechanistic explanation in Lesson 17.1 is precisely why "wait for a smarter model" is not, by itself, a fix — hallucination is a property of how any next-token-prediction model generates text, not a bug specific to any one model's current capability level.
- Setting max-step limits so low that legitimate complex tasks get cut off, or so high that runaway loops become expensive before being caught — this trade-off has no universal right answer; it should be calibrated against the actual expected step count of your real tasks (measured, not guessed) plus a reasonable safety margin, not picked arbitrarily.
- Relying only on exact-match loop detection and assuming that covers "infinite loops" as a category — as shown above, a stuck agent rewording the same failing action repeatedly will sail right past an exact-match check while being just as unproductive as literal repetition.
- Building guardrails and validation as an afterthought bolted on after a production incident, rather than as a designed-in part of the tool-calling and output-handling code from the start — retrofitting these checks is always more expensive and more likely to miss cases than designing them in alongside the tools and outputs they're meant to constrain.

### Exercise
For an agent that books restaurant reservations via an API, list 3 specific failure scenarios and the mitigation you'd apply to each.

### Challenge
Design a monitoring dashboard (on paper — list the metrics) that would let an on-call engineer quickly spot each of the 6 failure modes covered in this module in a production agent system. For at least one failure mode, specify a metric that would catch it *before* it fully manifests as a user-visible problem, not just after.

### Knowledge Check
1. Why shouldn't every tool failure be retried the same way?
2. What's the difference between a guardrail and validation?
3. Give one concrete technique to reduce hallucination beyond "hope the model doesn't."
4. Why can an agent get stuck in a loop that an exact-match repetition check would fail to detect?
