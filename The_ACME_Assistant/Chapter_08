## Chapter 8: Fine-Tuning Techniques

The Manager decided it was time for "Advanced Coaching." She didn't want to replace the Prodigy’s entire brain—that would be too expensive and risky. Instead, she wanted to add a small, specialized "Training Module" that focused purely on ACME’s unique behavior.

In this chapter, we go hands-on with **Parameter-Efficient Fine-Tuning (PEFT)** and **LoRA**. We will learn how to surgically adjust the model's weights to enforce discipline, tone, and format—all while keeping our facts safely tucked away in the RAG library.

---

### 8.1 The Turbocharger Analogy: Why PEFT?

In the old days of AI, fine-tuning meant changing every single "neuron" in a model’s brain. It was like re-machining an entire engine block. Today, we have a better way.

> **Pro Tip: The Turbocharger Analogy for PEFT 🏎️**
> Think of the base model as a massive, pre-built engine.
> * **Full Fine-Tuning** is like re-machining the entire engine. It’s powerful but incredibly expensive and leaves you with a heavy, new engine to manage.
> * **PEFT (LoRA/QLoRA)** is like adding a small, high-efficiency **turbocharger**. You get a huge performance boost for your specific task (like acceleration) without altering the core engine. It’s cheaper, faster to install, and you can easily swap it out or turn it off.
> 
> 

---

### 8.2 Curating the Training Set

The Prodigy learns by example. If you give them 1,000 bad examples, they will become a 1,000-times-worse employee.

**What & Why: Supervised Fine-Tuning (SFT)**

* **What it is:** Showing the model pairs of (Input Prompt + Perfect Output).
* **Why we need it:** We need to show the model exactly what "The ACME Way" looks like.
* **The Rule:** Quality > Quantity. 100 perfect examples are worth more than 10,000 mediocre ones.

---

### 8.3 The Model Card: Your Nutrition Label

Before the Manager lets a "Tuned" Prodigy talk to customers, she demands a **Model Card**.

**The What & Why: The Model Card**

* **What it is:** A "nutrition label" for your AI. It lists what data it was trained on, its intended use, its known weaknesses, and its safety scores.
* **Why we need it:** It builds trust. It tells everyone: "This model is safe for HR questions, but do not use it for medical advice."

---

### 🛠️ Hands-On Lab: "Training the ACME Adapter"

**Goal:** Prepare a dataset and simulate the training of a LoRA adapter that enforces a "Professional Citation" style.

**Step 1: The "Gold" Dataset**
Create a `training.jsonl` file. Each line should contain a RAG-style context, a question, and a "Gold" answer that follows our JSON schema and tone.

```json
{"instruction": "Answer using context...", "input": "What is the pet policy?", "output": "{\"status\": \"ok\", \"answer\": \"Pets are welcome in the lobby...\", \"source\": \"Handbook_v2\"}"}

```

**Step 2: Hyperparameter Selection**
Choose your "Rank" (the size of the turbocharger).

* **r=8:** Small and fast. Good for simple style changes.
* **r=32:** Larger. Better for complex logic changes.

**Step 3: Evaluation**
Run your "Base" model and your "Tuned" model side-by-side.

* Did the Tuned model use the JSON format correctly?
* Is the tone consistent?

**Acceptance Criteria:**

* A PII-scrubbed training dataset of at least 50 high-quality examples.
* A draft Model Card describing the version, base model, and intended behavior.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Catastrophic Forgetting":** If you tune a model *too* hard on one task, it might "forget" how to be a general-purpose thinker. Always keep a few general-knowledge examples in your training set.
* **"Training on Hallucinations":** Never use model-generated data that hasn't been human-verified as your training set. You'll just be hard-wiring mistakes into the brain.

---

### Checklists: Tuning Readiness

* [ ] Training data is clean and scrubbed of any private info (PII).
* [ ] Base model is stable and version-pinned (e.g., `llama-3-8b`).
* [ ] Test set is ready to compare "Before" and "After" results.

**What's Next:**
Our Prodigy is now a trained expert with a library card. But they are still "living in the moment"—they don't remember what we talked about yesterday. In **Part III: Systems in the Wild**, we give them a memory. We start with **Chapter 9: State & Memory**.
