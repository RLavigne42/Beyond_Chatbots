## Chapter 12: Evaluation & Evals (Regression Gates)

The Manager had built a powerful system, but she had a nagging worry: "Is it actually getting better, or just different?" Every time she updated a prompt or swapped a model, she was terrified she might be fixing one bug while creating ten new ones. She couldn't rely on "vibes" or a few random tests anymore. She needed a **Scientific Scoreboard**.

In this chapter, we build the **Evaluation Harness**. We will move from guessing to measuring, creating a "Regression Gate" that ensures no update is ever deployed unless it is proven to be an improvement.

---

### 12.1 The "Vibes" vs. "Values" Problem

In traditional software, code either works or it doesn't. In AI, the output can be "mostly right" but "slightly weird."

**What & Why: LLM-as-a-Judge**

* **What it is:** Using a very large, smart model (like a "Director" model) to grade the work of our smaller "Prodigy" model based on a strict rubric.
* **Why we need it:** Human managers don't have time to read 10,000 chat logs. We need an automated way to score accuracy, tone, and safety at scale.

---

### 12.2 The Evaluation Rubric: The Manager's Red Pen

The Manager doesn't just give a grade of "A" or "F." she grades based on four specific **Dimensions**:

1. **Groundedness:** Did the Prodigy stick to the facts in the Library, or did it make things up?
2. **Relevance:** Did the answer actually address the user's specific question?
3. **Tone:** Did it sound like an ACME employee (professional and helpful)?
4. **Conciseness:** Did it answer the question without unnecessary rambling?

---

### 12.3 Creating the "Golden Dataset"

**The Intuitive Analogy: The Final Exam**
You can't grade a student if you don't have the answer key. A **Golden Dataset** is a collection of 50–100 "perfect" examples: the exact question, the exact RAG context, and the exact "Gold Standard" answer. Every time we change the code, we make the Prodigy retake this "Final Exam."

---

### 12.4 The Regression Gate: Protecting Production

**Trade-offs & Decisions: Manual vs. Automated Evals**

* **Manual Evals:** Deeply accurate but slow and expensive.
* **Automated Evals:** Fast and cheap but can sometimes miss subtle nuances.
* **Our Decision:** We use **Automated Evals** for every code change, but we require a **Manual Review** of the "failed" cases before we push to production. This is our **Regression Gate**.

---

### 🛠️ Hands-On Lab: "Building the Scoreboard"

**Goal:** Create an automated script that compares two different prompt versions and tells you which one is more "Grounded" in the facts.

**Step 1: The Test Runner**
Write a script that takes 10 questions from your Golden Dataset and runs them through your Chapter 3 Prompt Registry (v1 and v2).

**Step 2: The Judge Prompt**
Create a specialized "Judge" prompt. It should receive the *Context*, the *Answer*, and the *Rubric*, and return a JSON score from 1 to 5.

**Step 3: The Delta Report**
Generate a summary: "Version 2 is 15% more grounded than Version 1, but 10% slower."

**Acceptance Criteria:**

* An automated test suite that produces a "Scorecard" for every run.
* A clear definition of "Pass/Fail" (e.g., "Must score at least 4/5 on Groundedness to deploy").
* A log of every evaluation stored alongside the `trace_id`.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Evaluating on the Training Set":** Don't test the Prodigy using the exact same examples you used to train it. That’s just checking its memory, not its intelligence.
* **"The 100% Trap":** Don't wait for a perfect 5/5 score. In the world of LLMs, "Better than yesterday" is the goal. If you wait for perfection, you’ll never ship.

---

### Checklists: Eval Readiness

* [ ] Golden Dataset of at least 50 varied questions created.
* [ ] "Judge" model identified and rubric written.
* [ ] Evaluation script integrated into the CI/CD pipeline.

**What's Next:**
We have our scoreboard. We know our Prodigy is ready for the big leagues. Now, we need to build the "stadium" where they will work. In **Chapter 13: LLMOps**, we look at the infrastructure required to run the ACME Assistant at scale.
