# Part 17 — Glossary

**Agent** — A system built around an LLM that is given a goal, decides its own steps, uses tools, observes results, and continues until the goal is met.

**Agent Loop** — The repeating cycle an agent follows: Observe → Think → Plan → Act → Check Result → Continue/Finish.

**API (Application Programming Interface)** — A way for programs to communicate with other services or systems over a defined interface.

**Checkpointing** — Saving an agent's state at meaningful points during execution so it can recover from failure without restarting from scratch.

**Chunking** — Splitting a large document into smaller pieces before embedding, to improve retrieval precision.

**Context Window** — The maximum number of tokens an LLM can process in a single call, including input and output combined.

**Context Rot** — Degraded model performance/attention caused by overloading the context window with excessive or irrelevant content.

**Critic Pattern** — A multi-agent pattern where a dedicated agent reviews another agent's output and provides feedback for revision.

**Embedding** — A numeric vector representation of text that captures its meaning, allowing semantic comparison between pieces of text.

**Evaluation (Agent Evaluation)** — The practice of measuring an agent's task success, tool accuracy, cost, latency, and reliability, typically via a test dataset.

**Few-Shot Prompting** — Including example input/output pairs in a prompt so the model infers the desired pattern.

**Fine-Tuning** — Further training an existing model on a specific dataset to specialize its behavior (distinct from prompting, which happens at inference time).

**Guardrail** — A pre- or post-execution check that constrains an agent's inputs or outputs to prevent unsafe or invalid behavior.

**Hallucination** — When an LLM generates confident but false or unsupported information.

**Human-in-the-Loop (HITL)** — A design pattern where an agent pauses to request human approval, especially before risky or irreversible actions.

**Inference** — The process of using an already-trained model to generate a response to a new input (as opposed to training).

**LLM (Large Language Model)** — A generative AI model trained on massive amounts of text, capable of understanding and generating human language.

**Memory (Agent Memory)** — Information an agent retains and can retrieve, spanning working, short-term/conversation, and long-term (semantic/episodic) memory.

**Multi-Agent System** — A system composed of multiple specialized agents that collaborate, coordinated by a pattern such as supervisor, pipeline, or debate.

**Orchestration** — The logic that coordinates multiple steps or agents, deciding what runs when and how results flow between them.

**Parameters** — The internal numeric values an LLM learned during training; roughly correlated with model capacity.

**Plan-and-Execute** — A reasoning pattern where an agent generates a full plan upfront, then executes it step by step, replanning only if needed.

**Planning** — Breaking a goal into a concrete, ordered sequence of actionable subtasks.

**Prompt** — The complete text input given to an LLM, including instructions, context, and the task/question.

**Prompt Injection** — An attack where untrusted input (a document, webpage, or message) contains hidden instructions attempting to override an agent's actual instructions.

**Prompt Template** — A reusable prompt structure with placeholders filled in at runtime.

**RAG (Retrieval-Augmented Generation)** — Grounding an LLM's answers by retrieving relevant external documents at query time and injecting them into the prompt.

**ReAct** — A reasoning pattern that interleaves reasoning ("Thought") and tool use ("Action") in a tight loop.

**Reflection** — A reasoning pattern where an agent critiques and revises its own prior output before finalizing it.

**Reranking** — A second-pass, more precise scoring of retrieved candidates in a RAG pipeline, applied after an initial fast/approximate retrieval step.

**Semantic Search** — Finding relevant content by meaning (via embeddings/similarity) rather than exact keyword matching.

**State (Agent State)** — The current snapshot of an agent's progress and context during a run; can be session-scoped or persistent across sessions.

**Structured Output** — Instructing an LLM to respond in a specific machine-readable format (e.g., JSON) rather than free-form prose.

**Supervisor Pattern** — A multi-agent pattern where a central agent delegates tasks to specialist agents and coordinates their outputs.

**System Prompt** — Instructions set by the developer that define an LLM's role, rules, and constraints for an entire session, distinct from the user's own messages.

**Temperature** — A setting controlling randomness/creativity in an LLM's output; lower values are more deterministic, higher values more varied.

**Token** — The atomic unit of text an LLM actually processes; roughly ¾ of a word on average for English text.

**Tool** — A function or API an agent can call to take an action or retrieve information from outside the LLM itself.

**Tool Calling** — The mechanism by which an LLM decides to invoke a tool with structured input, receives the result, and continues reasoning.

**Tool Schema** — A structured (usually JSON) description of a tool's name, purpose, and expected input, used to inform the LLM's tool-selection decisions.

**Training** — The (typically one-time, expensive) process of teaching a model language patterns from massive datasets.

**Vector Database** — A specialized database for storing embeddings and efficiently finding the ones most similar to a given query vector.

**Vector Similarity** — A mathematical measure (e.g., cosine similarity) of how close two embedding vectors are, used as a proxy for semantic similarity.

**Workflow** — A fixed, developer-defined sequence of steps (which may include LLM-powered steps) that does not change its own structure at runtime.
