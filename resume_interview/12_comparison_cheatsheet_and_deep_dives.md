# 12. Interview Comparisons & Core Flows Cheat Sheet

> [!TIP]
> This guide provides direct, rapid-fire answers to the most common "What is the difference between X and Y?" questions, as well as step-by-step explanations of core AI workflows like Attention and RAG.

---

## 1. Backend Frameworks: FastAPI vs Flask vs Django

When asked why you chose FastAPI for MeetAI and Siren Eyes, use these comparisons.

| Feature | FastAPI | Flask | Django |
|---|---|---|---|
| **Best For** | High-performance async APIs, ML model serving | Small microservices, prototyping | Full-stack web apps, CMS, rapid CRUD |
| **Concurrency** | Native `async`/`await` (ASGI) | Traditionally synchronous (WSGI) | Traditionally synchronous (now adding ASGI) |
| **Speed** | Very fast (Starlette + Pydantic) | Moderate | Slower (heavy batteries included) |
| **Data Validation** | Automatic via Pydantic | Manual or third-party (Marshmallow) | Built-in form validation |
| **Documentation** | Automatic Swagger/OpenAPI | Manual or third-party extensions | Third-party (drf-yasg) |
| **"Batteries"** | Minimal (bring your own ORM) | Minimal (microframework) | Batteries-included (Auth, Admin panel, ORM) |

### 🎤 How to answer in an interview:
*"I chose **FastAPI** for our AI systems because AI model inference is often I/O bound when calling external services, and requires heavy WebSocket usage for streaming audio. FastAPI's native async support and Starlette foundation handle thousands of concurrent WebSocket connections efficiently. Additionally, its tight integration with Pydantic ensures the JSON payloads sent to and from our LLMs are strictly validated automatically, which isn't natively supported in Flask or Django."*

---

## 2. Databases: MongoDB vs PostgreSQL

You used MongoDB in MeetAI. Be prepared to defend this choice against traditional SQL.

| Feature | MongoDB (NoSQL) | PostgreSQL (SQL / Relational) |
|---|---|---|
| **Data Structure** | Unstructured/Semi-structured JSON (BSON) | Structured tables (Rows/Columns) |
| **Schema** | Flexible / Dynamic schema | Rigid schema (requires migrations) |
| **Scaling** | Horizontal scaling (Sharding) out of the box | Vertical scaling is easier; Horizontal is harder |
| **Relationships** | Not ideal for deep relationships (embeds documents) | Excellent for complex Joins (Foreign Keys) |
| **ACID Compliance** | Document-level ACID (Multi-doc ACID in recent versions) | Strict, complete ACID compliance |
| **Best For** | Chat logs, transcripts, IoT sensor data, rapid iteration | Financial systems, user billing, highly structured data |

### 🎤 How to answer in an interview:
*"In MeetAI, I designed a pluggable database architecture, but heavily utilized **MongoDB** for production. Meeting transcripts are highly unstructured—some turns have action items, some have nested speaker metadata, and the schema evolves rapidly as we add new AI features. MongoDB's document-based nature allowed us to store a full meeting's data in a single JSON-like document without writing complex SQL JOINs, making read operations extremely fast."*

---

## 3. Deep Dive: Transformers & Attention

If the interviewer asks: *"Explain to me how a Transformer works and what Attention actually is."*

### What is Attention?
Attention is a mathematical mechanism that allows a neural network to focus on specific parts of the input sequence when predicting a certain output, just like a human pays attention to specific words in a sentence to understand its context.

**The "Cocktail Party" Analogy:**
Imagine you are at a noisy party. You are able to tune out the background noise and "pay attention" only to the person talking to you. Self-attention does this for words: it calculates how strongly every word in a sentence is related to every other word.

### Step-by-Step: The Self-Attention Mechanism (Q, K, V)
Think of it like a database retrieval process:
1.  **Query (Q):** What I am currently looking for (the current word).
2.  **Key (K):** What I contain (labels for all other words).
3.  **Value (V):** The actual content/meaning of the word.

**The Math (Simplified):**
1.  For the word "bank" in *"I sat by the river bank"*, the model creates a **Query**.
2.  It compares this Query against the **Keys** of all other words via a **Dot Product**.
3.  The dot product between "bank" and "river" will be very high (they are highly related). The dot product between "bank" and "sat" will be lower.
4.  These scores are passed through a **Softmax** function to turn them into percentages (e.g., "bank" pays 80% attention to "river", 15% to "sat", 5% to "I").
5.  Finally, we multiply these percentages by the **Values** to get a new, context-aware vector for the word "bank".

*(In this context, the model learns that "bank" means the side of a river, not a financial institution.)*

### Q&A on Transformers
*   **Q: What is Multi-Head Attention?**
    *   **A:** Instead of having one attention mechanism, the model has multiple (e.g., 8 or 12 "heads"). One head might focus on grammar (subject-verb agreement), while another focuses on semantic meaning, allowing the model to capture multiple relationships simultaneously.
*   **Q: Why do Transformers need Positional Encodings?**
    *   **A:** Unlike RNNs, which read text sequentially (word by word), Transformers process all words at the exact same time (in parallel) to utilize GPU acceleration. Without positional encodings added to the input, the model wouldn't know the difference between *"The dog bit the man"* and *"The man bit the dog"*.

---

## 4. Deep Dive: The RAG Flow (Retrieval Augmented Generation)

If asked to explain your MeetAI RAG pipeline from scratch.

### The Problem RAG Solves
LLMs are frozen in time (knowledge cutoff) and hallucinate facts because they guess the next word based on probability. RAG fixes this by giving the LLM an "open book" to read from before answering.

### Step-by-Step Flow

#### Phase 1: Data Ingestion (Index Time)
1.  **Extraction:** Take the raw meeting transcript from MongoDB.
2.  **Chunking:** Split the transcript into smaller, meaningful pieces. *(In MeetAI, you use conversational turns, e.g., overlapping windows of 6 speaker turns, to preserve Q&A context).*
3.  **Embedding:** Pass each chunk through an embedding model (like `bge-small-en-v1.5` via `fastembed`). This model converts the text into a dense vector (an array of numbers representing its semantic meaning).
4.  **Storage:** Save these vectors and the original text into a Vector Database (like Qdrant).

#### Phase 2: Retrieval & Generation (Query Time)
5.  **User Query:** The user asks, *"What did we decide about the marketing budget?"*
6.  **Query Embedding:** Pass the user's question through the *exact same* embedding model to get a query vector.
7.  **Vector Search (Cosine Similarity):** The Vector DB compares the query vector against all chunk vectors to find the closest matches in multi-dimensional space.
8.  **Context Construction:** The top 3 most relevant chunks are retrieved.
9.  **Augmented Prompting:** The backend builds a prompt:
    *   *System:* "You are a helpful assistant. Answer strictly using the context provided below."
    *   *Context:* [Insert the 3 retrieved transcript chunks here]
    *   *Question:* "What did we decide about the marketing budget?"
10. **Generation:** The LLM reads the context and generates an accurate, grounded answer without hallucinating.

### Q&A on RAG
*   **Q: What is Cosine Similarity?**
    *   **A:** It measures the angle between two vectors. If the angle is 0 degrees, the cosine is 1 (they mean the exact same thing). If they are 90 degrees apart, cosine is 0 (unrelated). It is preferred over Euclidean distance because it normalizes for the length of the text.
*   **Q: How do you handle a user asking a follow-up question in RAG?**
    *   **A:** We use a "Query Reformulation" step. We pass the chat history and the new question (e.g., *"Who approved it?"*) to an LLM first, asking it to rewrite the query into a standalone search term (*"Who approved the marketing budget?"*). Then we embed that rewritten query.
