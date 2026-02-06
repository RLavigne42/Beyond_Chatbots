## Part III: Systems in the Wild (Delegating with Guardrails)

**Prologue to Part III:** At the end of Part II, we forged an expert. Our ACME Assistant, equipped with deep knowledge through RAG and refined skills through fine-tuning, is a powerful cognitive engine. But so far, this expert has worked in a pristine lab under perfect conditions.

Part III is about giving our expert a real job. We are moving from model capabilities to **system reliability**. In the "wild," the Prodigy must handle the chaos of human conversation, the permanence of database writes, and the responsibility of operational safety.

---

## Chapter 9: State & Memory

The Prodigy was brilliant, but they had a frustrating flaw: every morning, they walked into the office and forgot who everyone was. A user would spend twenty minutes explaining a complex problem, only for the Prodigy to say, "Hello! I'm the ACME Assistant. How can I help you today?"

The Manager realized that **Statelessness**—the AI's tendency to live only in the present moment—was a dealbreaker for a real product. The Prodigy needed a **Digital Notebook**.

In this chapter, we add request, session, and user memory. We will build a system that remembers what it should, forgets what it must, and respects privacy above all else.

---

### 9.1 The "Digital Notebook": Scopes of Memory

Not all memories are equal. The Manager established four distinct "folders" in the Prodigy's notebook:

| Scope | Lifetime | The Manager's Logic |
| --- | --- | --- |
| **Request** | Seconds | "Just the scratchpad for this one specific task." |
| **Session** | Minutes/Hours | "Remember what we've been talking about in this chat." |
| **User** | Permanent | "Remember this person's name, role, and preferences." |
| **Global** | Forever | "Shared knowledge that applies to everyone at ACME." |

**The What & Why: Context Bloat**

* **What it is:** Shoving every single past message into the prompt.
* **Why we need to avoid it:** It makes the model slower and much more expensive. We must learn to **summarize** and **prune** the notebook.

---

### 9.2 Summarization Buffers (Session Memory)

**The Intuitive Analogy: The Running Summary**
Imagine a 50-page conversation. Instead of making the Prodigy re-read all 50 pages every time, we have them write a **Rolling Summary**. Every few minutes, they update a one-paragraph recap of what has been decided so far. This keeps the "notebook" slim and focused.

---

### 9.3 User Memory: Profiles and Facts

**The What & Why: Vector Memory**

* **What it is:** Using a small, private vector database to store "atomic facts" about a user (e.g., "User prefers Python over Java").
* **Why we need it:** When a user returns after a week, we can perform a quick "semantic search" of their notebook to recall only the facts relevant to the new question.

---

### 9.4 Privacy & The "Right to Forget"

**Trade-offs & Decisions: Personalization vs. Privacy**

* **Personalization:** Makes the Assistant feel "magic" and helpful.
* **Privacy:** Is a legal and ethical requirement.
* **Our Decision:** We implement **Just-in-Time Consent**. We never save a personal fact without asking the user: "May I remember that for next time?" We also provide a "Forget Me" button that hard-deletes the user's notebook folder.

---

### 🛠️ Hands-On Lab: "The Persistent Assistant"

**Goal:** Implement a session summary buffer and a "User Preference" store using a simple database.

**Step 1: The Session Store**
Create a table in a database (like SQLite or Redis) that stores `session_id`, `user_id`, and a `summary` string.

**Step 2: The Summarizer**
Write a background task that triggers after every 5 messages. It sends the transcript to a "Cheap" model (from Ch 2) and asks for a 2-sentence summary.

**Step 3: The Context Assembler**
Update your Orchestrator (from Ch 1) to fetch the `summary` and the `user_preferences` and inject them into the System Prompt before calling the model.

**Acceptance Criteria:**

* The Assistant can recall the user's name or a previous topic from 10 messages ago using only the summary.
* The system correctly prunes old messages to stay under a 2,000-token "Memory Budget."
* A `DELETE /memory/{user_id}` endpoint successfully clears the stored facts.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Storing Secrets":** Never let the Prodigy write down passwords or credit card numbers in the notebook. Use a PII-filter to scrub the text before it hits the database.
* **"Hallucinated Memories":** Sometimes the model "thinks" it learned a fact that the user never said. Always show the user their "Profile" so they can correct mistakes.

---

### Checklists: Memory Readiness

* [ ] Database for session and user state is configured.
* [ ] TTLs (Time-to-Live) are set so old sessions are deleted automatically.
* [ ] Privacy policy and "Clear Memory" functionality are visible to the user.

**What's Next:**
Our Prodigy can now remember and reason. But they are still just "talking." It's time to give them **Hands**. In **Chapter 10: Agents & Tool Use**, we teach the Prodigy how to use the company's software to get things done.
