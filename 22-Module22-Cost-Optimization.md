# Module 22 — Cost Optimization

### Difficulty
Advanced

### Learning Objectives
- Understand token costs and how model choice affects them.
- Learn caching, smaller models, tool optimization, context optimization, and batch operations.

### Prerequisites
Modules 1–21.

---

## Lesson 22.1 — Token Costs and Model Selection

### Concept Explanation

Recall from Module 2.1 that everything an LLM processes — your system prompt, the conversation history, retrieved documents, the model's own generated response — is measured in tokens, and from Module 0.7 that every call to an LLM provider is an ordinary HTTP request under the hood. Providers meter and bill exactly the same way a utility company meters electricity: they count how much of their resource (in this case, tokens processed) you actually consumed, and charge accordingly, usually with **separate, different rates for input tokens versus output tokens** (output tokens are typically priced higher, because generating each one requires a full pass of the model's computation, whereas many input tokens can be processed together in parallel). This billing mechanic has a direct, mechanical consequence for agent systems specifically: an agent, by its very nature (Module 6), makes *multiple* LLM calls per task — one for each iteration of the think-act-observe loop — and every one of those calls, every tool description included in every call, every piece of conversation history re-sent, and every token of the model's response, adds to a running total that a single-shot chatbot query would never accumulate.

Providers also offer models at different capability tiers — commonly described informally as "small," "medium," and "large," though exact naming varies by provider — and larger, more capable models cost meaningfully more per token than smaller ones, often by a factor of 10x or more between the smallest and largest models a provider offers. This isn't arbitrary pricing; it roughly reflects the actual computational cost of running a larger model (more parameters, per Module 2.4, generally means more computation per token generated), which the provider passes through to you. The practical consequence: choosing to run every single step of a multi-step agent task through the largest available model is not just "the safe choice" — it's a direct multiplier on your total cost, and as the "Model Tiering" example below shows, that expense often buys you nothing, because many of an agent's individual steps don't actually require the largest model's extra capability to get right.

### A Common Question

**"If a bigger model is always at least as capable as a smaller one, why not just always use it and skip the tiering decision entirely?"** Because "at least as capable" doesn't mean "worth the extra cost for this specific step." Consider the tool-selection decision inside an agent's loop (Module 7): given a small, well-scoped set of clearly-described tools, this is often a task a smaller, cheaper model handles just as reliably as a large one — the decision space is narrow and the signal (the tool descriptions) is clear, so the extra reasoning capacity of a larger model isn't actually being exercised. Running that same narrow decision through the most expensive available model is like hiring a surgeon to take a patient's temperature — the surgeon *could* do it perfectly well, but you're paying surgeon rates for a task that never required surgeon-level skill. The skill this lesson teaches is recognizing, task by task, which steps genuinely benefit from extra capability (nuanced writing, complex multi-factor reasoning) and which don't (simple classification, well-scoped tool selection, straightforward data extraction) — Module 22's overall goal is spending your capability budget where it actually buys you something.

**"How do I know in advance whether a smaller model is 'good enough' for a given step, rather than guessing?"** This connects directly back to Module 19's evaluation framework — don't guess, measure. Run the same test dataset (Module 19.3) through both a smaller and a larger model for the specific step you're considering downgrading, and compare task success rate and tool accuracy side by side. If the smaller model's accuracy on that specific, narrow task is statistically indistinguishable from the larger model's, you've empirically confirmed the downgrade is safe and you keep the cost savings; if it's meaningfully worse, you've learned that step genuinely needs the larger model's extra capability, and now you know that with evidence rather than intuition.

### Simple Analogy

> Using the largest, most capable model for every single step of an agent's work is like hiring your most senior (and most expensive) engineer to also handle basic data entry. Match the "seniority" (capability, and cost) of the model to the actual difficulty of each step — you wouldn't put your most senior engineer on data entry, and you also wouldn't put your newest intern on a task that genuinely requires deep judgment; the skill is in the matching, not in always picking one end of the spectrum.

### Practical Example — Model Tiering

```text
Task: Classify a support ticket into one of 5 categories
   → Use a smaller/cheaper, faster model (simple classification task)

Task: Draft a nuanced response to an angry enterprise customer
   → Use the larger, more capable model (requires nuance, tone control)

Task: Decide which tool to call next in a simple agent loop
   → Use a smaller model with clear tool descriptions and few-shot examples
```

Why these specific assignments, and not the reverse: the classification task has a small, fixed, well-defined answer space (exactly 5 categories) — there's a clear right answer, and a smaller model's pattern-matching capability is entirely sufficient to get it right reliably, the same way you wouldn't need advanced judgment to sort mail into 5 labeled bins. The customer-response task, by contrast, has no fixed correct answer, requires reading emotional tone, balancing competing goals (de-escalate the customer, protect the company's position, sound authentically empathetic rather than robotic) — exactly the kind of open-ended, nuanced judgment where a more capable model's extra training and capacity genuinely produces better, more appropriate output. The tool-selection task sits in between conceptually but leans toward the small-model side specifically *because* Module 7.2 already told you the fix for ambiguous tool selection is clearer descriptions and few-shot examples, not a bigger model — investing in prompt quality here is both cheaper and more effective than throwing more model capability at an underlying description-writing problem.

---

## Lesson 22.2 — Caching

### Concept Explanation

**Caching**, as a general engineering idea, avoids paying for the same computation twice by storing the result of a computation the first time and reusing that stored result on subsequent, sufficiently-similar requests instead of redoing the work from scratch. In LLM systems this idea shows up in two genuinely different forms, operating at two different layers of the system, and it's worth understanding both the mechanism and the trade-off of each separately.

**Prompt/context caching** exploits a specific mechanical fact about how an LLM processes its input: processing tokens (turning them into the model's internal numerical representations, per Module 2's explanation of tokenization and the model's internal machinery) is itself a real computational cost, and if the *exact same* leading portion of a prompt — say, a long, unchanging system prompt plus a large reference document — appears at the start of many different requests, a provider can, in principle, do that processing work once, store the intermediate result, and reuse it on every subsequent request that shares that same exact prefix, only doing fresh processing for whatever new text comes after it. From your perspective as the developer, this typically shows up as a meaningfully cheaper and faster response on any call whose prompt starts with a chunk of text the provider has recently seen and cached — the mechanism is invisible to you beyond the pricing and latency benefit, but understanding *why* it works (same prefix → provider can skip redoing the same processing) tells you exactly how to structure your prompts to benefit from it: put the stable, unchanging parts of your prompt (system instructions, reference documents, tool schemas) *first*, and put the part that changes on every call (the specific user question) *last* — if you interleave them, or put the changing part first, you break the shared-prefix property the caching mechanism relies on.

**Response caching** operates at a completely different layer — it's not about the provider's internal processing at all, but about your own application logic deciding "I've already computed the full answer to this exact (or near-identical) question before; let me just hand back that stored answer instead of calling the LLM again at all." This is conceptually the simplest possible optimization (a dictionary lookup instead of an API call) but it only applies where it's actually valid: if the same or a very similar query is likely to recur *and* the correct answer doesn't change over time (a company's static return policy, an FAQ answer), response caching can eliminate the LLM call entirely for that request. It does **not** apply to per-user personalized answers, or to answers that depend on data that changes frequently (today's weather, this user's current account balance) — caching a stale answer to a question whose correct answer has since changed produces exactly the kind of confidently-wrong output Module 17 warns about, just from a different root cause (a stale cache instead of model hallucination).

### A Common Question

**"How does prompt caching interact with my agent's growing conversation history — does every new turn break the cache?"** This depends on exactly what changed between calls, and it's worth being precise here because it affects how you structure agent state (Module 16). If your agent's conversation grows by strictly *appending* new messages to the end (the pattern used throughout Module 7's tool-calling loop), the earlier portion of the prompt stays identical between consecutive calls, and a caching-aware provider can still reuse that unchanged prefix, paying only for the newly appended tokens. If instead you modify or re-summarize earlier parts of the history between calls (a legitimate technique for managing context-window limits, per Module 2.2 and Module 8), you break the shared-prefix property for that specific change, and the cache benefit resets from that point forward. This is a genuine trade-off worth knowing about: context summarization saves tokens by shrinking what you send, but it can simultaneously reduce your ability to benefit from prompt caching, because you've changed the previously-cached prefix. Neither technique is strictly better — which one wins depends on how much of your cost comes from raw prompt size versus how much comes from repeated, un-cached reprocessing of a stable prefix.

**"Is response caching just for exact duplicate questions, or can it handle near-duplicates too?"** A naive implementation only catches exact string matches (the identical question, character for character), which is useful but limited — a customer asking "what's your return policy?" and another asking "how do returns work?" would be treated as entirely different queries and both would hit the LLM. A more sophisticated implementation uses the embedding techniques from Module 9: embed the incoming question, check whether it's semantically similar (via vector similarity) to any recently-cached question, and if the similarity is high enough, serve the cached answer instead of calling the LLM again. This is a more powerful but also riskier technique — set the similarity threshold too loose, and you risk serving a cached answer to a question that's actually meaningfully different from the one that was originally cached, which is a subtle but real correctness risk worth testing carefully (Module 19's evaluation techniques apply directly here).

### Visual Diagram

```text
Call 1: [Long System Prompt + Doc] + "Question A" → LLM (full cost)
                     ↓ (system prompt + doc cached)
Call 2: [Cached System Prompt + Doc] + "Question B" → LLM (much cheaper — only
                                                         "Question B" is new)
```

**How to read this diagram:** the key detail is what stays identical between Call 1 and Call 2 — "Long System Prompt + Doc" is the exact same text in both calls, sitting at the same position (the front of the prompt), while only "Question A" versus "Question B" differs. This shared, unchanging prefix is exactly what a caching-aware provider can detect and reuse; if Call 2 had instead used a slightly reworded system prompt, or put the document in a different position, the two calls would no longer share an identical prefix, and the cost/speed benefit shown here would disappear even though the actual content is almost the same.

---

## Lesson 22.3 — Tool and Context Optimization

### Concept Explanation

These two optimizations target two different places cost silently accumulates in an agent system, and it's worth understanding both mechanisms rather than treating them as one generic "be efficient" instruction.

**Tool optimization** targets wasted *tool calls* specifically — every tool call is, itself, typically an HTTP request to some external system (Module 0.7), which costs time (latency) directly, and often costs money indirectly (many third-party APIs charge per call, and even "free" internal APIs consume server resources). Beyond the direct cost of the call itself, every tool result that comes back also gets fed into the agent's next LLM call as additional context (Module 7's tool-loop pattern), which means a wasteful tool call doesn't just cost the price of that one API call — it also inflates the token count of every subsequent LLM call in that same agent run, compounding the waste. The fix is twofold: **avoid redundant calls** (don't re-fetch data the agent's own state already holds from two steps ago — this requires the agent's execution loop to actually check its existing state before deciding a new tool call is needed, rather than blindly calling a tool every time a related question comes up) and **batch related calls where possible** (if an agent needs weather for five cities, one tool call that accepts a list of five cities and returns five results is cheaper and faster than five separate round-trips, each with its own network latency and its own separately-priced request).

**Context optimization** targets the size of what actually gets sent to the LLM on each call, applying the same principles established in Module 2.2 (context windows have a real cost, and stuffing them full has diminishing or even negative returns due to context rot) and Module 10 (RAG's whole premise is retrieving *only* the relevant portion of a large knowledge base, rather than sending everything). Concretely: retrieve only what's relevant instead of dumping entire documents into context (a targeted RAG retrieval of the 3 most relevant paragraphs costs a fraction of the tokens that pasting an entire 50-page document would, and — per Module 10 — often produces a *better* answer too, since the model isn't hunting for the relevant needle in an irrelevant haystack); and summarize or compress older conversation history instead of keeping it verbatim forever (once a multi-turn agent conversation grows long, the earliest turns are often far less relevant to the current decision than the most recent ones — replacing them with a short summary preserves the gist while dramatically shrinking the token count of every subsequent call).

### A Common Question

**"If summarizing old history saves tokens, why not summarize aggressively from the very first turn?"** Because summarization is lossy by definition — a summary, by construction, discards detail in exchange for brevity, and if the discarded detail turns out to matter later (a specific number, a precise constraint the user mentioned early on), the agent genuinely loses access to information it would have needed. This is a real trade-off, not a free win: the right amount of summarization depends on how far back "still might matter" realistically extends for your specific task. A common practical pattern is to keep the most recent N turns verbatim (since recent context is disproportionately likely to be immediately relevant) while summarizing everything older than that — giving you most of the token savings while preserving full fidelity for what's most likely to matter right now.

### Practical Example

```python
# BAD: re-fetches the same weather data on every reasoning step
for step in agent_steps:
    weather = get_weather("Pune")  # wasteful repeated tool call
    ...

# GOOD: fetch once, reuse from state
if "weather" not in state:
    state["weather"] = get_weather("Pune")
weather = state["weather"]
```

*Walking through why the "BAD" version is actually wasteful, mechanically:* if `agent_steps` has, say, 6 iterations, the "BAD" loop makes 6 separate real network calls to the weather API for the exact same city, paying for (and waiting on) 6x the necessary latency and (if the weather API is metered) 6x the necessary cost, purely because nothing in the loop checked whether it already had this information. The "GOOD" version applies the same state-checking pattern you first saw in Module 16's `AgentState` — `if "weather" not in state` is a one-line guard that asks "have I already done this work?" before doing it again, and the fix costs nothing extra in code complexity; it's simply remembering to ask the question before acting, the same discipline behind avoiding any redundant computation in ordinary software engineering, just applied here to a tool call whose "redundant computation" happens to cost real API-call money on every repetition rather than just wasted CPU cycles.

---

## Lesson 22.4 — Batch Operations

### Concept Explanation

**Batch operations** process many items together in fewer, larger calls instead of one call per item — and it's worth distinguishing two separate savings mechanisms bundled into this one technique, because they apply in different situations. The first mechanism is reduced *overhead*: every individual API call carries some fixed cost beyond the actual content being processed (network round-trip latency, any fixed per-request processing on the provider's side) — combining 100 small requests into fewer, larger ones amortizes that fixed overhead across more actual work, the same way shipping 100 small packages together in one truck is cheaper than sending 100 separate delivery trucks, even though the total weight being moved is identical. The second mechanism is provider-offered **batch pricing**: many LLM providers offer a meaningfully discounted per-token rate specifically for requests submitted through a "batch" processing mode, where you submit a large set of requests together and receive results after some delay (often within a few hours) rather than instantly — this is economically viable for the provider because it lets them schedule that work into spare computational capacity rather than guaranteeing instant availability, and the savings get passed through to you in exchange for accepting that delay.

This second mechanism is why batching is explicitly described as best suited for "non-time-sensitive workloads" — the discount is a direct trade for giving up immediate response time, so it only makes sense for tasks where waiting is actually acceptable (classifying yesterday's support tickets overnight, generating tomorrow's report ahead of time) and actively wrong for tasks where a user is waiting in real time for a response (an interactive chat agent cannot use batch mode, because "responds sometime in the next few hours" fails the basic requirement of an interactive product).

### A Worked Cost Comparison

To make the actual dollar impact of these optimizations concrete rather than abstract, consider a hypothetical support-ticket classification agent processing 10,000 tickets per day, where each classification call uses roughly 300 input tokens (ticket text plus a short classification prompt) and 20 output tokens (just the category label):

| Approach | Model Tier | Est. Cost per 10,000 Tickets/Day | Why |
|---|---|---|---|
| Naive: largest model, real-time, individual calls | Large | ~$45.00 | Every call pays the large model's per-token rate, with no batching or caching applied |
| Model tiering only: small model, real-time, individual calls | Small | ~$4.50 | A simple 5-category classification task (Lesson 22.1) doesn't need the large model's extra capability — roughly a 10x reduction from tier alone |
| Model tiering + prompt caching: small model, shared classification-prompt prefix cached | Small | ~$3.20 | The fixed instructional part of the prompt (the category definitions, repeated identically on every call) is cached rather than reprocessed in full each time |
| Model tiering + caching + batch mode: small model, submitted as one overnight batch job | Small | ~$1.60 | Batch-mode pricing discount stacks on top of the already-reduced small-model, cached-prompt cost |

*(These figures are illustrative, built from realistic relative pricing ratios between model tiers and batch discounts, not live provider pricing — always check current published rates before budgeting a real system, since the underlying dollar figures change over time even though the relative savings *pattern* shown here remains representative.)*

The point this table is making isn't the specific dollar figures — it's that these four optimizations are **multiplicative, not alternative**: model tiering alone gets you roughly a 10x reduction, and each additional technique (caching, batching) stacks a further discount on top of whatever came before it, rather than being four separate options where you pick just one. A team that only applies the first optimization (tiering) and stops there is leaving a further ~3x savings on the table for zero additional accuracy cost, purely by also applying caching and batch mode where the workload's time-sensitivity allows it.

### Practical Example

```text
BAD (100 separate calls):
for email in 100_emails:
    classify(email)   # 100 separate LLM calls

GOOD (batched):
classify_batch(100_emails)   # fewer calls, or one batch job, processing all
                              # 100 together — cheaper and often faster overall
```

*Explanation:* the "BAD" loop makes 100 independent round-trips, each paying whatever fixed per-request overhead exists and each billed at standard (non-batch) rates; `classify_batch(100_emails)` either combines them into fewer real-time requests (reducing per-request overhead) or submits them through a provider's dedicated batch API (unlocking discounted batch pricing, per the table above) — which specific mechanism applies depends on your provider's API, but the underlying principle (don't pay the fixed cost of a round-trip 100 separate times when you can pay it once) is the same either way.

### Key Takeaways
- Cost scales with token volume, model tier, and number of LLM calls — every one of these is a lever you can pull, and (as the worked cost comparison shows) they compound rather than being mutually exclusive choices.
- Match model capability to task difficulty instead of defaulting to the most powerful (and expensive) model everywhere — validate the downgrade empirically using Module 19's evaluation framework rather than guessing.
- Caching, context optimization, and batching are complementary — apply all three where applicable rather than picking just one; the worked example above shows each one stacking a further discount on top of the others.

### Common Mistakes
- **Defaulting to the largest available model for every single agent step "to be safe."** As the worked example shows, this multiplies cost by roughly 10x for tasks that never needed the extra capability in the first place, with little to no measurable quality benefit for simple, well-scoped sub-tasks — "safe" here is an untested assumption, not a verified conclusion, and Module 19's evaluation techniques exist specifically to replace that assumption with actual evidence.
- **Re-sending large static context (system prompts, reference documents) on every call without leveraging prompt caching where the provider supports it.** This throws away a discount that costs nothing to claim beyond structuring your prompt with the stable content first (Lesson 22.2) — it's a case where more savings is available for free, simply by being aware the mechanism exists.
- **Not monitoring cost per task type in production (Module 19–20).** Cost problems often go unnoticed until the bill arrives, precisely because — unlike a functional bug that produces a visibly wrong answer a user might report — an inefficient but *correct* agent produces the right output every time, giving no natural signal that anything is wrong until someone actually looks at the per-task-type cost breakdown.
- **Applying batch mode to time-sensitive, interactive workloads.** This is a mismatch between the optimization and the task's actual requirements — the delay batch mode requires in exchange for its discount is fundamentally incompatible with a user waiting in real time for a response, and forcing it onto an interactive product trades cost savings for a broken user experience, which is rarely the right trade.

### Exercise
For a 3-agent content pipeline (Research → Write → Edit from Module 14), assign an appropriate model tier (small/medium/large) to each agent's task and justify your choice.

### Challenge
Design a cost-monitoring dashboard (list the fields/breakdowns) that would let a team see cost per agent, per model tier, and per task type — enough detail to identify exactly where money is being spent inefficiently.

### Knowledge Check
1. Why does model tiering (matching model size to task difficulty) reduce cost without necessarily reducing quality?
2. What's the difference between prompt caching and response caching?
3. Give one concrete example of a wasteful, avoidable repeated tool call.
