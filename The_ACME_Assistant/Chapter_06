## Chapter 6: Vector Search & RAG Deep Dive

The Corporate Library was stocked, but there was a new problem: the library was too big. If the Prodigy had to read every "index card" to find an answer, the user would be waiting for hours. The Manager needed to give the Prodigy a **Super-Fast Librarian**—someone who could hear a question and immediately pull the three most relevant cards from a collection of thousands.

In this chapter, we build the **Retrieval-Augmented Generation (RAG)** engine. This is the heart of the Assistant’s intelligence, combining the model's reasoning with our company’s specific facts.

---

### 6.1 Vector Search: Searching by Meaning

Traditional search looks for exact words. If you search for "laptop," it might miss a document about "notebooks." We need **Vector Search**.

**What & Why: Embeddings**

* **What it is:** A process that turns a chunk of text into a long list of numbers (a "vector") that represents its meaning.
* **Why we need it:** It allows the Assistant to find documents that are *related* to a question, even if they don't share the same exact keywords.

---

### 6.2 Hybrid Search: The Best of Both Worlds

**The Manager’s Logic: Accuracy + Context**
Sometimes, exact words *do* matter—like a part number (e.g., "XJ-900"). Vector search might get "close," but we need the exact match.

* **Lexical Search (BM25):** Finds exact words and codes.
* **Vector Search:** Finds concepts and meanings.
* **Hybrid Search:** Combines both scores to find the perfect result.

---

### 6.3 Reranking: The Second Opinion

Retrieving 50 chunks is easy; picking the best 5 is hard.

* **The What & Why: Reranking**
* **What it is:** A second, smaller, and smarter model that looks at the top 50 results and re-orders them by how well they *actually* answer the specific query.
* **Why we need it:** It significantly reduces "noise" and ensures the Prodigy only sees the highest-quality information.



---

### 6.4 Context Assembly & Citations

Once we have the best chunks, we must present them to the Prodigy. But we can't just shove them in. We must provide **Provenance**.

**The Intuitive Analogy: The Footnotes**
If the Prodigy gives an answer, the Manager asks, "How do you know that?" By including the `source_id` in the prompt, we force the Prodigy to cite its work (e.g., "According to the [Benefits Manual], you have 20 days of vacation.").

**Trade-offs & Decisions: Context Length vs. Cost**

* **More Chunks:** Better chance of finding the answer, but more expensive and the model might get "Lost in the Middle."
* **Fewer Chunks:** Cheaper and faster, but might miss the answer.
* **Our Decision:** We use **k=6** (top 6 chunks) with **Reranking** to ensure high quality without bloat.

---

### 🛠️ Hands-On Lab: "Building the RAG Engine"

**Goal:** Create a function that takes a user query, finds the best chunks from your library, and builds a "Grounded" prompt for the LLM.

**Step 1: The Vector Store**
Load your JSON chunks from Chapter 5 into a Vector Database (like `Chroma`, `Pinecone`, or `FAISS`). Generate embeddings for each.

**Step 2: The Retriever**
Implement a `retrieve(query)` function that returns the top 5 most similar chunks.

**Step 3: The Grounded Prompt**
Create a new prompt template:

```text
Use the following context to answer the question. If the answer is not in the context, say "I don't know."

Context:
{context_chunks}

Question: {user_query}

```

**Acceptance Criteria:**

* The Assistant correctly answers a question using information found in the local library.
* The Assistant says "I don't know" when asked a question that is *not* in the library.
* The response includes at least one citation (e.g., "[Source: manual.pdf]").

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Vector Dogma":** Don't assume vector search is magic. If users search for specific IDs or names, you *must* use Hybrid Search.
* **"Stuffing the Context":** Don't give the model 20 chunks. It will get confused. Use a Reranker to find the best 5 instead.

---

### Checklists: RAG Readiness

* [ ] Vector database selected and populated.
* [ ] Embedding model matches the one used during ingestion.
* [ ] "I don't know" fallback policy tested and enforced.

**What's Next:**
Our Assistant is now a knowledgeable researcher. But sometimes, "knowing" isn't enough—the Assistant needs to "behave" better. In **Chapter 7: Hybrid RAG + Fine-Tuning**, we look at how to tune the Prodigy's personality without losing its library card.
