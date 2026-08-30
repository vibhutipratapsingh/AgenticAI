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

### Simple Analogy

> Think of tokens like Lego bricks of language. The model doesn't see "unbelievable" as one solid word — it might see it as bricks like `["un", "believ", "able"]`. It builds and reads meaning by combining these bricks.

### Visual Diagram

```text
"Agentic AI is powerful"
        ↓ tokenization
["Agent", "ic", " AI", " is", " powerful"]
        ↓
   [4218, 291, 8837, 318, 6547]   (numeric token IDs the model actually processes)
```

### Practical Example

Why this matters practically:
- LLM APIs charge **per token** (input + output), not per word or character.
- LLMs have a maximum number of tokens they can process at once — the **context window**.

### Key Takeaways
- Tokens are the atomic units an LLM actually processes — not words, not characters.
- Token count affects both cost and what fits in the model's context window.

### Common Mistakes
- Assuming "1 token = 1 word" — this breaks down for non-English text, code, and unusual words.
- Ignoring token limits when stuffing large documents into a prompt (leads to truncation or errors).

### Exercise
Take a paragraph of text and estimate its token count using the ≈4 characters/token rule. Then, if you have access to an LLM playground with a token counter, check your estimate.

---

## Lesson 2.2 — Context Window

### Concept Explanation

The **context window** is the maximum amount of text (measured in tokens) an LLM can "see" at once — including the system prompt, conversation history, retrieved documents, and its own response.

### Simple Analogy

> The context window is like the size of someone's desk. However much reading material you hand them, only what fits on the desk (their current context) can influence what they write next. Bigger desks (bigger context windows) let them consider more material, but a cluttered desk (too much irrelevant info) can also cause them to lose track of what matters.

### Visual Diagram

```text
┌─────────────────── Context Window (e.g., 200,000 tokens) ───────────────────┐
│ System Prompt │ Conversation History │ Retrieved Docs │ User Msg │ [Output] │
└───────────────────────────────────────────────────────────────────────────┘
```

### Practical Example

If a model has a 200K-token context window and your conversation history + documents already use 190K tokens, only 10K tokens remain for new input and the model's response — this is why long agent conversations need memory management (Module 8).

### Key Takeaways
- Context window = total token budget for one LLM call, in + out.
- Long conversations, huge documents, or many tool results can silently eat this budget.
- This limitation is *why* memory systems and RAG exist — you can't just paste everything in forever.

### Common Mistakes
- Assuming a bigger context window means you should always dump everything into it — irrelevant content can dilute the model's attention on what matters ("context rot").

### Exercise
If a model has a 128K token context window, and your system prompt + tools use 4K tokens, roughly how many tokens of conversation + documents can you fit before needing to trim something?

---

## Lesson 2.3 — Training vs. Inference

### Concept Explanation

**Training** is the (extremely expensive, one-time or periodic) process of teaching the model language patterns using massive datasets. **Inference** is what happens every time *you* send a prompt — the trained model uses what it already learned to generate a response. You, as a developer, almost always work at the inference stage.

### Simple Analogy

> Training is like a student spending years in school learning language, facts, and reasoning patterns. Inference is that same student, now graduated, answering a specific question you ask them on the spot — using only what they already learned, not re-studying for your question.

### Visual Diagram

```text
TRAINING (done once by the model provider)
Massive Text Data → Learning Process (weeks, huge compute) → Trained Model (frozen "knowledge")

INFERENCE (happens every time you call the API)
Your Prompt → Trained Model → Generated Response
```

### Key Takeaways
- You do not train the model when you send it a prompt — you use ("infer" from) an already-trained model.
- This is why LLMs have a "knowledge cutoff" date and don't automatically know about events after that — unless you give them that information via tools or retrieval (Modules 7, 10).

### Common Mistakes
- Confusing "the model learned something new" during a conversation with actual training — by default, nothing you say permanently updates the model; conversation memory is a separate engineering layer (Module 8).

### Exercise
Explain, in one sentence, why an LLM might not know about an event that happened yesterday, and what an engineer could do about it.

---

## Lesson 2.4 — Parameters and Temperature

### Concept Explanation

**Parameters** are the internal numeric values (weights) a model learned during training — think of them as the model's "knowledge encoded as numbers." Models are often described by parameter count (e.g., "a 70-billion-parameter model") as a rough proxy for capacity.

**Temperature** is a setting *you* control at inference time that controls randomness/creativity in the output. Low temperature (e.g., 0) makes output more deterministic and focused; high temperature (e.g., 1.0+) makes it more varied and creative.

### Simple Analogy

> Parameters are like the size and depth of someone's vocabulary and experience — built up over years, not something you tweak per-question.
> Temperature is like telling a writer "stick to the safest, most obvious phrasing" (low) vs. "feel free to get creative and unusual" (high) for this particular piece of writing.

### Visual Diagram

```text
Temperature = 0.0   →  "The capital of France is Paris."      (deterministic, focused)
Temperature = 1.2   →  "Ah, Paris — the shimmering heart of France!"  (varied, creative)
```

### Practical Example

Use low temperature for: data extraction, code generation, factual Q&A, agent tool-selection decisions (predictability matters).
Use higher temperature for: brainstorming, creative writing, generating varied options.

### Key Takeaways
- Parameters = fixed model capacity, set during training, not adjustable per-request.
- Temperature = a per-request creativity dial you control.
- Agents almost always want **low temperature** for decision-making steps (tool selection, planning) to stay predictable.

### Common Mistakes
- Using high temperature for agent reasoning/tool-selection steps — this introduces unpredictable, hard-to-debug behavior.

### Exercise
For each task below, would you set temperature high or low? (a) Writing a marketing tagline (b) Extracting a date from an email (c) An agent deciding which tool to call.

---

## Lesson 2.5 — System Prompts, User Prompts, and Structured Outputs

### Concept Explanation

- **System prompt**: instructions that set the model's role, behavior rules, and constraints for the entire conversation — set by the developer, not the end user.
- **User prompt**: the actual message/question from the person (or another program) interacting with the model.
- **Structured output**: instructing the model to respond in a specific machine-readable format (like JSON) instead of free-flowing prose, so your code can reliably parse it.

### Simple Analogy

> The system prompt is like a job description handed to a new employee before their first day ("You are a customer support agent. Be polite. Never give legal advice.").
> The user prompt is like the actual customer walking up and asking a question.
> A structured output is like asking that employee to fill out a standardized form instead of writing a free-form paragraph — much easier for another system to process.

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

### Practical Example (Python, conceptual)

```python
response = llm_client.messages.create(
    system="You are a helpful travel assistant. Always respond with valid JSON.",
    messages=[{"role": "user", "content": "Suggest a 3-day itinerary for Tokyo."}],
    temperature=0.3,
)
# response.content is now parseable JSON your app can use directly
```

*Explanation:* the `system` field configures persistent behavior; `messages` carries the actual conversation; low temperature keeps the JSON structure reliable; the app can now `json.loads()` the result instead of parsing free text.

### Key Takeaways
- System prompts define role and rules; user prompts carry the actual request.
- Structured outputs (JSON, XML, etc.) are essential for agents — an agent's "decision" about which tool to call needs to be machine-parseable, not prose.
- This structured-output pattern is the seed of tool calling (Module 7).

### Common Mistakes
- Putting behavior rules in the user prompt instead of the system prompt — makes behavior inconsistent across turns/users.
- Asking for JSON but not validating it — models occasionally produce malformed JSON; production systems must handle parse failures (Module 17).

### Exercise
Write a system prompt for an agent that only answers questions about cooking recipes and refuses everything else politely. Then write one user prompt that should be refused and one that should be answered.

### Challenge
Design a structured JSON output schema for a "restaurant recommendation" response including name, cuisine, price range, and a match score. Explain why each field is included.

### Common Misconceptions About LLMs (Recap)
- ❌ "LLMs search the internet for answers." → ✅ They generate text from patterns learned during training (unless explicitly given a search tool — Module 7).
- ❌ "LLMs remember our past conversations automatically." → ✅ Memory must be engineered (Module 8).
- ❌ "A bigger context window means unlimited free-for-all input." → ✅ Cost and attention still matter (context rot).
- ❌ "Temperature = 0 guarantees the exact same output every time." → ✅ It greatly increases consistency but isn't always perfectly deterministic across providers/versions.
