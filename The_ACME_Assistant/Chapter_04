## Part II: Enrichment (Specialized Training)

**Prologue to Part II:** In Part I, we assembled the machinery. Our ACME Assistant is a robust, observable, and resilient service. It follows basic instructions and reports its own health. But for all its engineering elegance, it is still a generic machine. It can speak, but it has nothing unique to say.

The fundamental question of Part II is: **How do we transform our reliable machine into a knowledgeable and skillful expert?** We are going to move beyond "guessing the next word" and start "reasoning with data."

---

## Chapter 4: Advanced Prompt Patterns

The Manager watched the Prodigy work. While the "Employee Handbook" (Chapter 3) helped with simple tasks, the Prodigy struggled when things got messy. When asked a multi-part question, they would skip steps. When a user provided contradictory info, the Prodigy would just guess. The Manager realized the Prodigy needed more than just rules—they needed **Mental Models** for solving complex problems.

In this chapter, we move beyond single-shot instructions. We will implement proven reasoning strategies that turn the LLM into a deliberate, multi-step problem solver.

---

### 4.1 The Pattern Catalog: Picking the Right Tool

Not every problem is solved by "just thinking harder." Different tasks require different cognitive approaches.

| Pattern | Best For... | The Manager's Logic |
| --- | --- | --- |
| **Decomposition** | Big, multi-part projects | "Break it down into small tasks first." |
| **ReAct** | Using outside tools | "Think, then act, then see what happened." |
| **Critique & Refine** | High-precision style/safety | "Write a draft, then check it for mistakes." |
| **Self-Consistency** | Ambiguous math or logic | "Try it 3 times and see if you get the same answer." |

**What & Why: Chain-of-Thought (CoT)**

* **What it is:** Asking the model to "think step-by-step" before giving the final answer.
* **Why we need it:** By forcing the model to write out its logic, we significantly reduce "logic hallucinations" where the model jumps to the wrong conclusion.

---

### 4.2 Decomposition: Plan → Solve → Synthesize

**The Intuitive Analogy: The Project Outline**
If you ask the Prodigy to "Analyze the Q4 budget and suggest cuts," they might just pick a random number. Decomposition forces them to:

1. **Plan:** List the categories.
2. **Solve:** Calculate the spend for each.
3. **Synthesize:** Provide the final recommendation based on the numbers.

**Trade-offs & Decisions: Latency vs. Quality**

* **Single-shot:** Fast and cheap, but prone to errors on complex tasks.
* **Decomposition:** Slower and uses more tokens, but drastically increases reliability.
* **Our Decision:** We use Decomposition for any task with more than two logical steps.

---

### 4.3 ReAct (Reason + Act)

**The What & Why: The Feedback Loop**

* **What it is:** A loop where the model writes a "Thought," chooses a "Tool" (like a calculator), looks at the "Observation" (the result), and repeats.
* **Why we need it:** It allows the Assistant to interact with the real world (APIs, DBs) rather than just relying on its internal memory.

---

### 4.4 Critique-and-Refine (The Editor's Pass)

**The Intuitive Analogy: The Peer Review**
Even the best Prodigy makes typos. In this pattern, we use one model (the "Drafting" Prodigy) to write the answer, and another (the "Critic" Manager) to check it against our safety rules and style guide. Only when the Critic is satisfied does the answer go to the user.

---

### 🛠️ Hands-On Lab: "The Multi-Step Reasoner"

**Goal:** Implement a "Critique-and-Refine" pattern to ensure HR answers are always polite and include a disclaimer.

**Step 1: The Drafting Prompt**
Update your prompt package to include `hr_drafter`. Its only job is to answer the user's question.

**Step 2: The Critic Prompt**
Create `hr_critic`. Its job is to look at the draft and return a JSON list of "Violations" (e.g., "Missing mandatory legal disclaimer").

**Step 3: The Orchestration Loop**
Update your FastAPI service to:

1. Call the Drafter.
2. Pass the draft to the Critic.
3. If violations exist, call the Drafter one more time to "Refine" the answer.

**Acceptance Criteria:**

* Traces show both the original draft and the final refined version.
* 100% of test outputs contain the mandatory legal disclaimer.
* The system correctly refuses to "Refine" an answer that is fundamentally unsafe.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Pattern Soup":** Don't use every pattern at once. Use the simplest one that solves the task.
* **"Leaky Thoughts":** Don't show the internal "Chain-of-Thought" to the user unless it's a transparency feature. It can be messy and confusing.

---

### Checklists: Advanced Prompting Readiness

* [ ] Task complexity identified (Simple vs. Multi-step).
* [ ] Logic-heavy tasks use "Think step-by-step."
* [ ] Step caps implemented (e.g., "Max 5 reasoning steps") to prevent infinite loops.

**What's Next:**
Our Prodigy now has a sharp mind. But they still don't have the facts. In **Chapter 5: Data Pipelines & Quality**, we build the Corporate Library that gives our Prodigy access to the company's private knowledge.
