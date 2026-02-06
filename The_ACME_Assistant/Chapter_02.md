## Chapter 2: Model Selection & Architecture

The Workstation was built, the templates were ready, and the rules were set. Now, the Manager had to find the right **Prodigy** to sit at the desk. She didn't just want the one with the most famous name; she wanted the one who was best at *this specific job*. She decided to run a **Bake-Off**—a series of tests to see which candidate offered the best balance of brains, speed, and cost.

In this chapter, we transition from the architecture of the app to the **Engine** that powers it. We will learn how to choose, test, and swap models without rewriting our entire system.

---

### 2.1 The Decision Matrix: Brains vs. Budgets

Not every task requires a genius. If the job is just to summarize a single email, hiring a world-class expert is a waste of money. We evaluate our candidates across four key metrics:

1. **Quality (The "IQ"):** How well does it follow the instructions and stay grounded in facts?
2. **Latency (The "Speed"):** How long does the user have to wait for the first word (TTFT)?
3. **Cost (The "Salary"):** How much does the API provider charge per 1,000 tokens?
4. **Reliability:** How often does the service go down or return an error?

**The What & Why: Model Cascades**

* **What it is:** A strategy where you use a small, fast model for easy tasks and "escalate" to a larger, smarter model only when things get complex.
* **Why we need it:** It allows the ACME Assistant to be fast and cheap 90% of the time, while still being brilliant when it matters.

---

### 2.2 Running the Bake-Off: The Golden Task Set

The Manager doesn't just ask a candidate, "Are you smart?" She gives them a **Golden Task Set**—a collection of 20–50 real-world questions with "Golden Answers" that she knows are correct.

**The Intuitive Analogy: The Audition**
If you’re hiring a singer, you don't ask about their favorite color; you ask them to sing the specific songs they will perform in the show. In our Bake-Off, we test models on the exact JSON schemas and data we built in Chapter 1.

---

### 2.3 The Architectural Choice: Hosted vs. Self-Hosted

**Trade-offs & Decisions: Control vs. Convenience**

* **Hosted APIs (e.g., OpenAI, Anthropic, Google):** Great for starting fast. You don't have to manage servers, but you are at the mercy of their pricing and privacy policies.
* **Self-Hosted (e.g., Llama 3 on vLLM):** Total control over data and privacy. High upfront cost in hardware (GPUs) and engineering time.
* **Our Choice:** We start with **Hosted APIs** to move fast, but we build our code using a "Gateway" pattern so we can switch to a self-hosted model later without changing the rest of the app.

---

### 🛠️ Hands-On Lab: "The Model Bake-Off"

**Goal:** Test two different models (e.g., a "Small" model and a "Large" model) against your Chapter 1 API and record the results in a decision matrix.

**Step 1: Set Up the Gateway**
Create a simple `ModelGateway` class in your code. This ensures that the rest of your app doesn't care *which* model is answering; it just cares that the "Contract" is fulfilled.

**Step 2: Create the Test Suite**
Define 5 "Golden Questions" related to the ACME Knowledge Assistant (e.g., "What is the holiday policy?").

**Step 3: Run and Measure**
Execute the tests for both models. Use your `trace_id` to record:

1. **Success:** Did it follow the JSON schema? (Yes/No)
2. **Latency:** Time to first token (TTFT).
3. **Accuracy:** Does the answer match the "Golden Answer" rubric?

**Acceptance Criteria:**

* A completed CSV or JSON file showing metrics for at least two different models.
* A "Winner" selected based on your weighted decision matrix.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Picking the Biggest Model by Default":** This is how you go bankrupt. Always test if a smaller, cheaper model can do the job first.
* **"Hard-Coding the Model Name":** Never put `model="gpt-4"` directly in your business logic. Use a configuration variable so you can swap it in one second.

---

### Checklists: Model Readiness

* [ ] API keys for at least two models (e.g., a "Pro" and a "Flash" version).
* [ ] Decision matrix weights agreed upon (e.g., is speed more important than cost?).
* [ ] Gateway pattern implemented to hide the model provider from the Orchestrator.

**What's Next:**
We have the Workstation (Ch 1) and we have the Prodigy (Ch 2). Now, we need to teach the Prodigy how to communicate. In **Chapter 3: Prompt Engineering Fundamentals**, we move from simple questions to **Versioned Instructions** that act as the Prodigy's "Employee Handbook."
