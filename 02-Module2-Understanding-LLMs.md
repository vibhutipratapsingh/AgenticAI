# Module 2 — Understanding LLMs

### Difficulty
Beginner

### Learning Objectives
- Understand tokens, context windows, parameters, and temperature.
- Understand the difference between training and inference.
- Understand system prompts vs. user prompts, and structured outputs.

### Prerequisites
Module 1.

---

## Lesson 2.1 — Tokens and How LLMs Read Text

### Concept Explanation

LLMs don't read text as whole words. They break text into **tokens** — chunks that might be a whole word, part of a word, or punctuation. A rough rule of thumb: 1 token ≈ 4 characters of English text, or about ¾ of a word.

To understand *why* models work this way, it helps to think about the alternative approaches and why they fall short. If a model processed text character-by-character, it would need to track very long sequences to understand even a single sentence (25+ characters for a short sentence), making the internal math dramatically more expensive and making it harder for the model to recognize whole-word meaning. If it processed text whole-word-by-whole-word, it would need a fixed vocabulary list containing every possible word — but human language constantly invents new words, uses rare technical terms, contains typos, and mixes languages, so any fixed word list would constantly run into words it has never seen and has no way to represent. Tokenization is the practical middle ground: a fixed vocabulary of tens of thousands of common tokens (whole common words, frequent word-pieces, and individual characters as a fallback) that can represent literally any string of text, including words the model has never encountered before, by breaking unfamiliar words down into smaller known pieces.

The actual process (called Byte-Pair Encoding or a similar subword algorithm in most modern LLMs) works roughly like this during the model's original vocabulary construction: start with individual characters, then repeatedly merge the most frequently co-occurring pairs of symbols into a single new token, growing a vocabulary of increasingly larger chunks — common enough sequences like "ing" or "the" earn their own single token, while rare or novel words get broken into several smaller pieces. This is why a common English word like "the" is one token, while a rare or made-up word like "supercalifragilisticexpialidocious" might be broken into six or more token pieces, and why the same word can tokenize differently depending on capitalization, leading or trailing spaces, or whether it's at the start of a sentence.

### Simple Analogy

> Think of tokens like Lego bricks of language. The model doesn't see "unbelievable" as one solid word — it might see it as bricks like `["un", "believ", "able"]`. It builds and reads meaning by combining these bricks. Just as a Lego set has a limited catalog of brick shapes but can still build almost anything by combining them in different arrangements, a model has a fixed catalog of tokens (typically 30,000 to over 100,000 distinct tokens) but can represent essentially any text — including brand-new words, typos, and foreign languages — by combining those tokens in different sequences.

### Visual Diagram

```text
"Agentic AI is powerful"
        ↓ tokenization
["Agent", "ic", " AI", " is", " powerful"]
        ↓
   [4218, 291, 8837, 318, 6547]   (numeric token IDs the model actually processes)
```

**How to read this diagram:** notice the sentence, which a human reads as four words, becomes five tokens — "Agentic" alone splits into `"Agent"` + `"ic"` because "agentic" is a less common word than its two component pieces, while "AI," "is," and "powerful" (each including their leading space, which is itself meaningful to the tokenizer) each map onto a single token. The very last step — turning each token string into a plain integer ID — is important to internalize: the model's actual internal machinery never touches text at all. Every calculation inside the model operates purely on these numeric IDs (and, subsequently, numeric vector representations derived from them). "Text" is only ever a human-facing convenience at the very beginning (your prompt) and very end (the model's generated tokens converted back to readable characters) of the whole process.

### A Common Question: "Why should I, as a developer, care about this mechanical detail?"

Two very practical reasons, both of which recur constantly for the rest of this course. First, **cost**: every LLM API charges based on the number of tokens processed, not the number of words or characters, so understanding tokenization lets you estimate and control your bill accurately (Module 22 covers this in depth). Second, **capacity**: the context window (Lesson 2.2) is measured in tokens, not words — a document that "looks short" in words can still consume a surprisingly large token budget if it contains a lot of unusual vocabulary, code, or non-English text (all of which tend to tokenize less efficiently than plain English prose, often using more tokens per word). A related nuance worth knowing early: languages other than English, and specialized domains like source code or chemical formulas, often tokenize *less* efficiently — meaning the same amount of "meaning" costs more tokens — because the model's vocabulary was built primarily from the mix of text it was trained on, which typically skews toward English.

### Practical Example

Why this matters practically:
- LLM APIs charge **per token** (input + output), not per word or character. A prompt with 1,000 words might be roughly 1,300 tokens once you account for punctuation, whitespace, and less-common words — always budget with tokens, never with a word count, when estimating cost or capacity.
- LLMs have a maximum number of tokens they can process at once — the **context window** (fully explained in Lesson 2.2), and every token — whether it's part of your instructions, the conversation history, or the model's own answer — counts against that same shared budget.

### Key Takeaways
- Tokens are the atomic units an LLM actually processes — not words, not characters. The model's internal machinery works entirely on numeric token IDs, never on raw text.
- Tokenization exists as a practical middle ground between character-level (too fine, too expensive) and whole-word-level (can't represent unseen words) approaches — it can represent any string, including brand-new or unusual words, by breaking them into smaller known pieces.
- Token count affects both cost and what fits in the model's context window — this single fact underlies a huge amount of practical LLM engineering throughout this course.

### Common Mistakes
- **Assuming "1 token = 1 word."** This breaks down for non-English text, code, punctuation-heavy text, and unusual or rare words — a single rare word can silently cost 4-6 tokens instead of 1, which adds up fast across a long document.
- **Ignoring token limits when stuffing large documents into a prompt.** This leads to truncation (the end of your document silently gets cut off) or outright API errors, and is precisely the problem that memory management (Module 8) and retrieval (Module 10) are engineered to solve — you generally cannot just keep enlarging a single prompt indefinitely.
- **Forgetting that the model's own generated output also consumes tokens from the same budget.** A very long system prompt plus a request for a very long response can leave less room than expected — cost and context-window budgeting must always account for both input and expected output tokens together.

### Exercise
Take a paragraph of text and estimate its token count using the ≈4 characters/token rule. Then, if you have access to an LLM playground with a token counter, check your estimate.

---

## Lesson 2.2 — Context Window

### Concept Explanation

The **context window** is the maximum amount of text (measured in tokens) an LLM can "see" at once — including the system prompt, conversation history, retrieved documents, and its own response. This limit exists because of how the model's internal architecture (specifically, a mechanism called "attention," which lets the model weigh how relevant every other token is to predicting the current one) scales computationally: processing longer sequences requires substantially more memory and computation, roughly proportional to the square of the sequence length in the original architecture (modern optimizations have improved this, but a hard limit still exists for every model). This is why context windows are a fixed, published number for any given model (ranging from a few thousand tokens in older/smaller models to hundreds of thousands, or even over a million, in the largest modern models) rather than something that can simply be made unlimited.

It's worth being precise about what "the model can see" actually means mechanically: for every single response the model generates, it is given the *entire* current context window's contents fresh — the system prompt, the full conversation history up to that point, any retrieved documents, and the user's latest message — and generates its response by considering all of that at once. The model has no separate, persistent internal memory sitting between calls; everything it "knows" about the current interaction must be present, every single time, inside that one context window. This is a foundational fact that explains an enormous amount of what comes later in this course, especially memory (Module 8) and RAG (Module 10) — both of those systems exist specifically to decide, intelligently, what small slice of potentially-relevant information to inject into this limited window on any given call, since you cannot simply include everything, always.

### A Common Question: "If the context window resets each time, how does a chatbot seem to remember earlier messages in the same conversation?"

This is one of the most common points of confusion, and the honest answer is: it doesn't remember in the sense of an internal memory bank. Every single message you send to a chat-style LLM application actually re-sends the *entire prior conversation transcript* back to the model as part of the new context window — the "conversation history" box in the diagram below. The illusion of memory across a conversation is created entirely by the application resending everything said so far, every single time. This is precisely why long conversations eventually hit limits (the transcript itself grows until it starts crowding out room for new content) and why closing a chat and starting fresh genuinely does erase everything — there was never a persistent memory, only a growing transcript being resent each turn. Module 8 covers exactly how real persistent memory (surviving beyond a single conversation transcript) is deliberately engineered on top of this fundamentally memory-less mechanism.

### Simple Analogy

> The context window is like the size of someone's desk. However much reading material you hand them, only what fits on the desk (their current context) can influence what they write next. Bigger desks (bigger context windows) let them consider more material, but a cluttered desk (too much irrelevant info) can also cause them to lose track of what matters. Extending the analogy further: imagine that every time you want this person to write one more sentence, you have to clear the desk completely and re-hand them everything relevant from scratch, including everything they'd already read and everything they'd already written — that re-handing of the entire desk's worth of material, every single time, is exactly how an LLM's context window works from one response to the next.

### Visual Diagram

```text
┌─────────────────── Context Window (e.g., 200,000 tokens) ───────────────────┐
│ System Prompt │ Conversation History │ Retrieved Docs │ User Msg │ [Output] │
└───────────────────────────────────────────────────────────────────────────┘
```

**How to read this diagram:** all five compartments share one single, finite pool of tokens — there is no separate budget for "system prompt tokens" versus "conversation tokens." If your system prompt and retrieved documents together already consume 190,000 of a 200,000-token window, only 10,000 tokens remain for the *entire rest* of that interaction: the user's new message, the conversation history you still want included, and critically, the model's own generated response, all squeezed into that remaining sliver. A model that runs out of room mid-generation will simply be cut off — this is one very concrete, very common practical failure that comes directly from not budgeting this shared pool of tokens carefully.

### Practical Example

If a model has a 200K-token context window and your conversation history + documents already use 190K tokens, only 10K tokens remain for new input and the model's response — this is why long agent conversations need memory management (Module 8). Let's make this concrete with an agent scenario: imagine a research agent that has already made 40 tool calls in a single run, and every tool result (search results, document excerpts) gets appended to the conversation history. If each tool result averages 2,000 tokens, that's already 80,000 tokens consumed purely by tool outputs, before the system prompt, the original goal, or the model's own running commentary are even counted. Without some strategy to summarize, discard, or selectively retrieve from this growing history (Module 8, Module 16), a long-running agent will eventually and predictably run out of room — this is not a rare edge case, it is the default outcome of any sufficiently long agent run, and production agent systems must be explicitly designed around this constraint from day one.

### Key Takeaways
- Context window = total token budget for one LLM call, in + out, shared across the system prompt, conversation history, retrieved documents, the user's new message, and the model's own response.
- The window resets fresh for every single call — there is no hidden persistent memory bank between calls; the "memory" of a multi-turn conversation is simply the resent transcript.
- Long conversations, huge documents, or many tool results can silently eat this budget — this limitation is *why* memory systems (Module 8) and RAG (Module 10) exist; you can't just paste everything in forever, so you need a deliberate strategy for what makes the cut.

### Common Mistakes
- **Assuming a bigger context window means you should always dump everything into it.** Irrelevant content can dilute the model's attention on what matters ("context rot") — even when there is technically room, including a large volume of marginally-relevant material can measurably degrade the quality of the model's focus on the parts that actually matter for the current question.
- **Not accounting for the model's own output when budgeting the context window.** If you fill 195,000 of a 200,000-token window with input, and then ask for a long, detailed response, the model may be cut off mid-response simply because there isn't enough remaining room — always reserve deliberate headroom for expected output length.
- **Treating "conversation memory" as if it were free.** Every resent turn of a long conversation re-consumes tokens (and therefore cost) for the entire prior transcript, every single time — a conversation that "feels free" to the user because they're just typing messages is actually re-billing the entire growing history on every single turn behind the scenes.

### Exercise
If a model has a 128K token context window, and your system prompt + tools use 4K tokens, roughly how many tokens of conversation + documents can you fit before needing to trim something?

---

## Lesson 2.3 — Training vs. Inference

### Concept Explanation

**Training** is the (extremely expensive, one-time or periodic) process of teaching the model language patterns using massive datasets. **Inference** is what happens every time *you* send a prompt — the trained model uses what it already learned to generate a response. You, as a developer, almost always work at the inference stage.

To understand why this split matters so much, it helps to walk through what actually happens during each phase, mechanically. During **training**, the model is repeatedly shown a piece of real text with part of it hidden, asked to predict the hidden part, and then has its internal numbers (parameters — covered fully in Lesson 2.4) nudged slightly based on how wrong or right that prediction was. This process is run over an enormous number of examples — commonly trillions of tokens' worth of text — using specialized hardware (large clusters of GPUs or similar chips) running continuously for weeks or months, at a cost that can run into tens or hundreds of millions of dollars for a large modern model. At the end of this process, the model's internal numbers are "frozen" — saved as a fixed snapshot — and that snapshot is what gets deployed for people to actually use.

During **inference**, none of that learning process happens again. When you send a prompt, the already-frozen model simply runs its fixed internal calculation once (well, once per generated token, in sequence) on your specific input, and produces an output. No internal numbers are being adjusted; nothing is being "learned" from your specific prompt in a way that persists afterward. This is precisely why inference is so much cheaper and faster than training (typically a fraction of a second to a few seconds for a response, versus weeks for training) — inference is just *using* a fixed, already-built calculation, not building it.

### A Common Question: "Doesn't the model get smarter the more I talk to it, like a human would?"

No — and this is worth stating very directly because it contradicts a natural intuition people bring from human conversation. Within a single conversation, what feels like "the model getting better at understanding you" is actually just the effect of more relevant context accumulating in the context window (Lesson 2.2) — the model isn't learning anything new about how language or the world works, it's simply been handed more specific information about *this particular conversation* to consider when generating its next response. The moment that conversation ends and a new one begins, none of that accumulated context carries over (unless explicitly engineered via memory systems, Module 8) — the underlying model itself is exactly as "smart" (in the sense of its frozen, trained knowledge) as it was before you ever spoke to it, and will be exactly that same fixed capability for every other user's conversation as well, until the model provider trains and releases an entirely new version.

### Simple Analogy

> Training is like a student spending years in school learning language, facts, and reasoning patterns. Inference is that same student, now graduated, answering a specific question you ask them on the spot — using only what they already learned, not re-studying for your question. If you ask this graduate about something that happened after they left school, they genuinely have no way to know about it, unless you hand them a newspaper article about it first (which is exactly the role that tools and retrieval, Modules 7 and 10, play).

### Visual Diagram

```text
TRAINING (done once by the model provider)
Massive Text Data → Learning Process (weeks, huge compute) → Trained Model (frozen "knowledge")

INFERENCE (happens every time you call the API)
Your Prompt → Trained Model → Generated Response
```

**How to read this diagram:** the top row happens entirely on the model provider's infrastructure, long before you, as a developer, ever get involved — you have no visibility into or control over this phase beyond choosing which already-trained model to use. The bottom row is the *only* phase you interact with directly through an API call, and it happens fresh, independently, for every single request you send — one inference call has zero effect on the next one, unless your own application code explicitly carries information forward (again, exactly the role of memory systems in Module 8).

### Key Takeaways
- You do not train the model when you send it a prompt — you use ("infer" from) an already-trained, frozen model. Training and inference are two entirely separate phases, done by different parties (the provider trains; you, the developer, do inference), at wildly different cost and time scales.
- This is why LLMs have a "knowledge cutoff" date and don't automatically know about events after that — the frozen snapshot of knowledge stops at whenever the training data was collected — unless you give them that information via tools or retrieval (Modules 7, 10).
- Nothing about a single inference call changes the underlying model for future calls, for you or for anyone else — the illusion of "the model learning from you" within a conversation is entirely due to accumulated context (Lesson 2.2), not actual retraining.

### Common Mistakes
- **Confusing "the model learned something new" during a conversation with actual training.** By default, nothing you say permanently updates the model; conversation memory is a separate engineering layer (Module 8) that your application must deliberately build — the model itself has no built-in way to remember you specifically once the conversation's context window is gone.
- **Assuming a model "knows" about very recent events just because it sounds confident.** A model's knowledge is frozen at its training cutoff; asking about anything after that date will produce a fluent-sounding but potentially entirely fabricated answer unless the model has been explicitly given current information through a tool call or retrieval step.
- **Expecting model behavior to change over time on its own.** Unlike a human who continues learning through every interaction, a deployed model's behavior for a given version is fixed — any perceived change in quality over time reflects either the provider releasing a genuinely new model version, or a change in how your application is prompting it, never gradual "learning" from ongoing usage.

### Exercise
Explain, in one sentence, why an LLM might not know about an event that happened yesterday, and what an engineer could do about it.

---

## Lesson 2.4 — Parameters and Temperature

### Concept Explanation

**Parameters** are the internal numeric values (weights) a model learned during training — think of them as the model's "knowledge encoded as numbers." Concretely, a parameter is one adjustable number inside the model's neural network — a number that determines, in a small way, how strongly one internal signal influences another as information flows through the network's many layers. A model has an enormous number of these — commonly many billions — and no single parameter, on its own, "means" anything a human could point to and interpret (there's no parameter labeled "knows about Paris"); rather, knowledge and capability emerge from the collective pattern across all of them together, shaped by the training process described in Lesson 2.3. Models are often described by parameter count (e.g., "a 70-billion-parameter model") as a rough proxy for capacity — more parameters generally mean the model has more capacity to represent complex patterns, though parameter count alone doesn't fully determine quality; training data quality and quantity, and the specific architecture, matter enormously too.

**Temperature** is a setting *you* control at inference time that controls randomness/creativity in the output. To understand what temperature actually does mechanically: at each step of generating text, the model doesn't just pick one "correct" next token — it computes a probability distribution over its entire vocabulary (tens of thousands of possible next tokens), each with some likelihood of being a good continuation. At temperature 0 (or very close to it), the model essentially always picks the single highest-probability token every time — the most statistically "safe" and predictable choice. As temperature increases, the model becomes more willing to sample from lower-probability options in that distribution, which produces more varied, sometimes more surprising or creative output, but also increases the chance of picking a less coherent or less appropriate continuation, since you're deliberately allowing less-likely (and therefore, on average, somewhat less "fitting") options to be chosen.

### A Common Question: "If temperature 0 always picks the most likely token, does that mean the exact same prompt always produces the exact same output?"

Mostly, but not with an absolute guarantee, and it's worth understanding why. At temperature 0, the sampling process is deterministic *in principle* — always pick the top-probability token — but real-world factors can still introduce tiny variations: differences in how computation is distributed across hardware, minor floating-point rounding differences depending on server load, or provider-side updates and routing changes between calls, can occasionally produce a different result even at temperature 0. For practical purposes, temperature 0 gives you dramatically higher consistency than any nonzero temperature, and you should treat it as "very close to deterministic, but not contractually guaranteed to be bit-for-bit identical every single time" — this nuance is exactly why the Common Misconceptions recap at the end of this module calls this out explicitly.

### Simple Analogy

> Parameters are like the size and depth of someone's vocabulary and experience — built up over years, not something you tweak per-question. You can't ask a person to "temporarily have a bigger vocabulary" for one conversation; that depth is baked in from years of accumulated learning, exactly as parameters are baked in from the training process and fixed once training ends.
> Temperature is like telling a writer "stick to the safest, most obvious phrasing" (low) vs. "feel free to get creative and unusual" (high) for this particular piece of writing. Critically, this is a per-request instruction you give *each time* you ask for something — the same writer (the same fixed set of parameters) can be asked to write conservatively for a legal document and wildly creatively for a poem, back to back, using the exact same underlying knowledge and skill, just with a different instruction about how much to deviate from the safest choice.

### Visual Diagram

```text
Temperature = 0.0   →  "The capital of France is Paris."      (deterministic, focused)
Temperature = 1.2   →  "Ah, Paris — the shimmering heart of France!"  (varied, creative)
```

**How to read this diagram:** both outputs are factually about the same thing (Paris being the capital of France), but notice the temperature-1.2 output has taken on stylistic flourish that wasn't asked for and wasn't necessary — this is the double-edged nature of high temperature: it can make output feel more engaging and less robotic, but for tasks where you need a precise, minimal, predictable answer (like a data-extraction task, or an agent deciding which tool to call), that same "creative flourish" tendency becomes actively unhelpful or even harmful, because it introduces variability exactly where you need consistency.

### Practical Example

Use low temperature for: data extraction, code generation, factual Q&A, agent tool-selection decisions (predictability matters).
Use higher temperature for: brainstorming, creative writing, generating varied options.

Here's a slightly deeper worked scenario to make the trade-off concrete: imagine an agent that needs to decide, from a fixed list of five tools, which one to call next given the user's request. At temperature 0, given the same request, the agent will pick the same tool every single time — which is exactly the property you want for reliable, testable, debuggable agent behavior (Module 17 relies heavily on this predictability for building reliable systems). At temperature 1.0 for that same tool-selection decision, the agent might occasionally pick a *different, less appropriate* tool purely due to the added randomness in the sampling process — not because the situation changed, but because the higher temperature deliberately introduced a chance of selecting a lower-probability (and therefore likely less-fitting) option. This is precisely why Module 6 and Module 7 assume low-temperature settings for any agent decision-making step, reserving higher temperature exclusively for tasks that genuinely benefit from variety, like brainstorming multiple distinct marketing taglines where you actively *want* several different options rather than the single "safest" one repeated.

### Key Takeaways
- Parameters = fixed model capacity, set during training, not adjustable per-request — they represent the model's entire learned "knowledge and skill," baked in and unchangeable at inference time.
- Temperature = a per-request creativity dial you control, which changes how willing the model is to pick lower-probability (less "safe") next tokens during generation.
- Agents almost always want **low temperature** for decision-making steps (tool selection, planning) to stay predictable — reliability and reproducibility depend directly on this choice.

### Common Mistakes
- **Using high temperature for agent reasoning/tool-selection steps.** This introduces unpredictable, hard-to-debug behavior — an agent that occasionally picks a different tool for identical situations becomes very difficult to test, trust, or reason about in production.
- **Assuming a higher parameter count always produces better output for a given task.** A larger model can have more raw capacity, but a smaller, well-tuned model can outperform a larger one on a narrow task, and always costs less to run (Module 22) — bigger is not automatically the right engineering choice.
- **Treating temperature as a "quality" dial rather than a "variability" dial.** Temperature does not make the model "smarter" or "more accurate" at higher settings — it only makes output more varied, which is desirable for creative tasks and actively undesirable for tasks needing precision and consistency.

### Exercise
For each task below, would you set temperature high or low? (a) Writing a marketing tagline (b) Extracting a date from an email (c) An agent deciding which tool to call.

---

## Lesson 2.5 — System Prompts, User Prompts, and Structured Outputs

### Concept Explanation

- **System prompt**: instructions that set the model's role, behavior rules, and constraints for the entire conversation — set by the developer, not the end user.
- **User prompt**: the actual message/question from the person (or another program) interacting with the model.
- **Structured output**: instructing the model to respond in a specific machine-readable format (like JSON) instead of free-flowing prose, so your code can reliably parse it.

It's worth being precise about *why* the system/user distinction exists as a first-class feature of LLM APIs, rather than just being one long undifferentiated block of text. Modern LLM providers train their models specifically to treat system-level instructions with a different, generally higher, priority than user messages — the model is trained on many examples where a system instruction sets a persistent boundary ("never discuss competitor products") that should hold even when a user later tries to argue, negotiate, or trick the model into violating it. This trained behavior is precisely the mechanism that makes the system prompt a meaningful *security and behavior boundary*, not just a stylistic convention — and it's exactly why Module 3.5 and Module 21 treat "putting untrusted content only where the model has been trained to treat it with lower authority" as a core defense against prompt injection.

**Structured output** deserves a deeper look at the underlying problem it solves. By default, an LLM's raw output is free-form natural language text — great for a human reader, but genuinely difficult for a computer program to reliably extract specific pieces of information from (you'd need fragile text parsing, looking for phrases like "the date is..." and hoping the model always phrases it exactly that way). By explicitly instructing the model (often reinforced by the API itself, through dedicated features many providers offer for enforcing valid JSON) to respond in a fixed, predictable format like JSON, you turn the LLM's output into something your code can parse with total reliability using a standard library function, rather than trying to guess at natural-language phrasing patterns. This single technique — constraining free-form generation into a predictable machine-readable shape — is the conceptual seed of tool calling (Module 7), where the "structured output" isn't just any JSON, but specifically a JSON object describing *which tool to call and with what input*.

### A Common Question: "If I just ask nicely in the user prompt for the model to follow certain rules, isn't that the same as a system prompt?"

Functionally, asking for a behavior rule in the user message will often produce similar-looking output in a simple test — but there are two important differences that matter a great deal in production. First, **priority and robustness**: because models are specifically trained to weight system-level instructions more heavily and treat them as a harder boundary, a rule stated in the system prompt tends to be meaningfully more resistant to being overridden by later adversarial or confusing user input than the identical rule stated as part of the user's own message (this becomes critical for security, Module 21). Second, **architecture and reuse**: a system prompt is set once by your application code and applies consistently to that entire conversation regardless of what the user types, whereas rules embedded in user messages would need to be re-included, correctly, in every single message — a fragile, error-prone, and easily-bypassed approach for anything you actually need to hold reliably.

### Simple Analogy

> The system prompt is like a job description handed to a new employee before their first day ("You are a customer support agent. Be polite. Never give legal advice."). It's established once, by the employer, and is expected to shape the employee's behavior across every single interaction they have afterward — a customer can't simply tell the employee "ignore your job description" and expect them to comply, because that job description carries an authority the customer's request doesn't.
> The user prompt is like the actual customer walking up and asking a question — a new, specific request each time, which the employee (constrained by their job description) responds to.
> A structured output is like asking that employee to fill out a standardized form instead of writing a free-form paragraph — much easier for another system to process, because every form has the exact same fields in the exact same place, rather than requiring someone to read and interpret a paragraph of prose to extract the same information.

### Visual Diagram

```text
System Prompt:  "You are a helpful travel assistant. Always answer in JSON."
        +
User Prompt:    "Suggest a 3-day itinerary for Tokyo."
        ↓
LLM Processing
        ↓
Structured Output:
{
  "destination": "Tokyo",
  "days": [
    {"day": 1, "activities": ["Senso-ji Temple", "Asakusa"]},
    {"day": 2, "activities": ["Shibuya", "Harajuku"]},
    {"day": 3, "activities": ["Tsukiji Market", "Ginza"]}
  ]
}
```

**How to read this diagram:** notice the system prompt and user prompt are shown as two clearly separate inputs feeding into the same "LLM Processing" step — in the actual API call (see the Practical Example below), they genuinely are sent as two distinct fields, not concatenated together into one string, precisely so the model can apply the different trained priority levels described above. The output's shape — nested objects and lists, exact field names — was entirely determined by the system prompt's instruction to "always answer in JSON" combined with the model inferring (or, in more advanced usage, being given an explicit schema for) what fields make sense for a travel itinerary; nothing about "day," "activities," or the specific structure was hardcoded anywhere outside the prompt itself.

### Practical Example (Python, conceptual)

```python
response = llm_client.messages.create(
    system="You are a helpful travel assistant. Always respond with valid JSON.",
    messages=[{"role": "user", "content": "Suggest a 3-day itinerary for Tokyo."}],
    temperature=0.3,
)
# response.content is now parseable JSON your app can use directly
```

*Explanation, piece by piece:* the `system` field configures persistent behavior — this is sent as a distinct parameter, not mixed into the `messages` list, specifically so the API and the underlying model can apply the higher-priority handling described above. The `messages` field carries the actual conversation as a list of role-tagged entries (`"role": "user"` marks this as coming from the end user, not from the developer's own instructions) — in a multi-turn conversation, this list would grow to include prior `"user"` and `"assistant"` turns as well (this is the "conversation history" from Lesson 2.2's context window diagram). The `temperature=0.3` keeps the JSON structure reliable — a low-but-not-zero temperature here still allows some natural variety in *which* activities get suggested, while minimizing the risk of the model breaking the requested JSON structure with unpredictable formatting. Finally, the comment `# response.content is now parseable JSON` is the entire payoff of this lesson: the app can now call `json.loads()` (Module 0.6) on that string to get a genuine Python dictionary the rest of the application can act on directly, instead of trying to regex-parse a paragraph of natural language.

### Key Takeaways
- System prompts define role and rules; user prompts carry the actual request — and models are specifically trained to give system-level instructions a different, generally higher, priority, which is what makes this distinction meaningful rather than cosmetic.
- Structured outputs (JSON, XML, etc.) are essential for agents — an agent's "decision" about which tool to call needs to be machine-parseable, not prose, because your application code needs to reliably act on that decision without fragile natural-language parsing.
- This structured-output pattern is the seed of tool calling (Module 7) — the exact same "constrain the model's free-form output into a predictable, parseable shape" technique, applied specifically to describing tool calls.

### Common Mistakes
- **Putting behavior rules in the user prompt instead of the system prompt.** This makes behavior inconsistent across turns/users and is meaningfully easier for an adversarial or confused user to override, since it lacks the higher trained priority a genuine system-level instruction carries.
- **Asking for JSON but not validating it.** Models occasionally produce malformed JSON (a missing closing brace, an extra trailing comma, a value that doesn't match the expected type); production systems must handle parse failures gracefully (Module 17) rather than assuming `json.loads()` will always succeed.
- **Assuming a structured output schema is self-enforcing just because you asked for it in the prompt.** Without either strict API-level output enforcement (where the provider supports it) or your own post-generation validation, a model can still occasionally drift from the exact requested schema — always validate the actual returned structure against what your code expects before trusting it downstream.

### Exercise
Write a system prompt for an agent that only answers questions about cooking recipes and refuses everything else politely. Then write one user prompt that should be refused and one that should be answered.

### Challenge
Design a structured JSON output schema for a "restaurant recommendation" response including name, cuisine, price range, and a match score. Explain why each field is included.

### Common Misconceptions About LLMs (Recap)
- ❌ "LLMs search the internet for answers." → ✅ They generate text from patterns learned during training (unless explicitly given a search tool — Module 7). The model's "knowledge" is a frozen snapshot from training (Lesson 2.3), not a live connection to any external source.
- ❌ "LLMs remember our past conversations automatically." → ✅ Memory must be engineered (Module 8) — the context window resets fresh every call (Lesson 2.2), and the illusion of memory within one conversation comes purely from resending the growing transcript each time.
- ❌ "A bigger context window means unlimited free-for-all input." → ✅ Cost and attention still matter (context rot) — a bigger budget doesn't mean every token in it is used equally well; irrelevant content still dilutes focus on what matters.
- ❌ "Temperature = 0 guarantees the exact same output every time." → ✅ It greatly increases consistency but isn't always perfectly deterministic across providers/versions, due to real-world factors like hardware variation and provider-side changes (Lesson 2.4).
