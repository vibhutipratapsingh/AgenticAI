# Module 9 — Vector Databases and Embeddings

### Difficulty
Intermediate

### Learning Objectives
- Understand what embeddings are and why they're useful.
- Understand vector similarity and vector databases.
- Understand chunking and semantic search.

### Prerequisites
Module 8.

---

## Lesson 9.1 — What Are Embeddings?

### Concept Explanation

An **embedding** is a way of converting text (a word, sentence, or document) into a list of numbers (a "vector") that captures its *meaning*. Texts with similar meaning end up with vectors that are mathematically close together, even if they don't share any exact words.

### Simple Analogy

> Imagine every sentence is represented as a location on a huge map. Similar ideas are located close together — "I love pizza" and "Pizza is my favorite food" would land near each other on this meaning-map, even though they don't share many exact words, while "I love pizza" and "The stock market crashed" would land far apart.

### Visual Diagram

```text
                       MEANING MAP (simplified to 2D)

    AI / ML Topics
           ●  "neural networks"
        ●        ●  "machine learning models"
     "LLMs"

                                          Cooking Topics
                                                 ●  "pizza recipes"
                                             ●  "how to bake bread"

                                                              Sports Topics
                                                                     ●  "football scores"
                                                                 ●  "basketball highlights"
```

In reality, embeddings have hundreds or thousands of dimensions — far more than 2D — but the "closeness = similar meaning" principle is exactly the same.

---

## Lesson 9.2 — Vector Similarity and Vector Databases

### Concept Explanation

- **Vector similarity**: a mathematical measure (commonly *cosine similarity*) of how close two embedding vectors are — higher similarity means more similar meaning.
- **Vector database**: a specialized database built to store millions of embeddings and quickly find the ones most similar to a given query vector (e.g., Chroma, FAISS, Pinecone, Qdrant, Weaviate).

### Visual Diagram

```text
User Question: "How do I reset my password?"
        ↓ (embedding model)
   Query Vector: [0.12, -0.44, 0.91, ...]
        ↓ (similarity search against stored vectors)
Vector Database
 ┌───────────────────────────────────────────┐
 │ Doc A vector  (similarity: 0.89) ← MATCH   │
 │ Doc B vector  (similarity: 0.31)           │
 │ Doc C vector  (similarity: 0.85) ← MATCH   │
 │ Doc D vector  (similarity: 0.05)           │
 └───────────────────────────────────────────┘
        ↓
   Top matches returned: Doc A, Doc C
```

### Practical Example (Conceptual Python)

```python
from embedding_model import embed_text   # produces a vector for a string
from vector_db import VectorDB           # a vector database client

db = VectorDB()

# Store documents
documents = [
    "To reset your password, go to Settings > Security > Reset Password.",
    "Our store hours are 9am to 9pm, Monday through Saturday.",
    "Refunds are processed within 5-7 business days.",
]
for doc in documents:
    db.add(vector=embed_text(doc), metadata={"text": doc})

# Query
query = "I forgot my password, what do I do?"
query_vector = embed_text(query)
results = db.search(query_vector, top_k=2)

for r in results:
    print(r["metadata"]["text"], r["similarity_score"])
```

*Explanation:* `embed_text` converts both stored documents and the incoming query into the same vector space. `db.search` finds the stored vectors closest to the query vector — this is **semantic search**: it works even though the query ("I forgot my password") shares almost no exact words with the matching document ("reset your password").

---

## Lesson 9.3 — Chunking

### Concept Explanation

**Chunking** is splitting a large document into smaller pieces before embedding it. Embedding an entire 50-page document as one vector loses fine-grained detail; embedding it in smaller chunks (e.g., paragraphs or ~300–500 tokens) allows retrieval of just the relevant portion.

### Visual Diagram

```text
Large Document (50 pages)
        ↓ chunking
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ Chunk 1  │ Chunk 2  │ Chunk 3  │  ...     │ Chunk N  │
└──────────┴──────────┴──────────┴──────────┴──────────┘
        ↓ embed each chunk separately
   [vector 1] [vector 2] [vector 3] ... [vector N]
        ↓ store all in vector database
```

### Key Takeaways
- Embeddings turn meaning into numbers that can be mathematically compared.
- Vector similarity search finds relevant content by *meaning*, not exact keyword match — this is the core mechanism behind semantic search and RAG (Module 10).
- Chunking balances retrieval precision (smaller chunks = more targeted matches) against context (too small loses surrounding meaning).

### Common Mistakes
- Chunking too large: retrieval returns bloated, mostly-irrelevant text along with the useful part, wasting context window budget.
- Chunking too small: loses surrounding context needed to understand the retrieved snippet correctly.
- Forgetting to store metadata (source, section, date) alongside chunks — without it, you can't tell the user *where* an answer came from or filter by relevance criteria.
- Assuming vector search alone is always best — exact keyword/numeric matches (e.g., product SKUs) are sometimes better served by traditional search (see "hybrid search" in Module 10).

### Exercise
Given the sentence "The weather is lovely for a picnic today," name two other sentences that should have high embedding similarity to it, and one sentence that should have very low similarity. Explain your reasoning based on meaning, not shared words.

### Challenge
Design a chunking strategy for a 200-page technical manual with numbered sections and code examples. What chunk size would you choose, and what metadata would you attach to each chunk to make retrieval more useful?

### Knowledge Check
1. What does "similarity" mean in the context of embeddings?
2. Why can semantic search find a relevant document even with zero shared keywords?
3. What problem does chunking solve, and what's the trade-off in choosing chunk size?

Continue to **[10-Module10-RAG.md](10-Module10-RAG.md)** — this module's concepts (embeddings, vector databases, chunking) are the building blocks of Retrieval-Augmented Generation.
