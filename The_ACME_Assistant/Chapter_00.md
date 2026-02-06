## Chapter 0: Before the First Line of Code — A Playbook for Product Strategy

The Manager sat at her desk, but she wasn't looking at code. She was writing a **Job Description**. She knew that hiring the most brilliant "Prodigy" in the world would be a waste of company resources if she didn't know exactly what problem they were supposed to solve. She needed to know how this new hire would make the team faster, the customers happier, or the business more profitable. She needed a plan before the "hiring" even began.

This chapter is that plan. Before we touch the machinery of Chapter 1, we must master the **Strategy**.

---

### 0.1 Finding the Right Problem: Where LLMs Excel

Not every problem is an "AI problem." If you use a sledgehammer to hang a picture frame, you’ll break the wall. In engineering, using an LLM where a simple database query or a regex script would work is expensive, slow, and over-engineered.

**The What & Why: Problem-Solution Fit**

* **What it is:** Identifying tasks where language complexity is the bottleneck, not logic.
* **Why we need it:** LLMs are "probabilistic" (they guess the next best word). They excel at **Summarization, Synthesis, and Extraction**. They struggle with **Perfect Arithmetic** or **Fixed Logic**.

> **The Manager’s Analogy:** Don't hire a poet to do your taxes, and don't hire a math professor to write your brand's greeting card. Use the Prodigy for what they are best at: understanding and generating human-like communication.

---

### 0.2 The AI Opportunity Scorecard

How do you pick between ten different ideas? You score them. We evaluate every potential ACME Assistant feature on four axes:

1. **Value:** Does this save 10 hours a week or 10 seconds?
2. **Feasibility:** Do we actually have the data to teach the model, or is it a mystery?
3. **Risk:** If the model gets this wrong, is it a funny typo or a legal disaster?
4. **Cost:** Will the API bill be higher than the value created?

---

### 0.3 Defining Success: The Metrics Tree

If the Sages ask the Manager, "Is the Prodigy doing a good job?", she can't just say, "They seem smart." She needs a **Metrics Tree**.

* **Business KPI (The Trunk):** "Reduced support ticket volume by 20%."
* **User Metric (The Branches):** "User gave the answer a 'Thumbs Up'."
* **Technical Metric (The Leaves):** "The model cited the correct document 95% of the time."

**Trade-offs & Decisions: Accuracy vs. Usefulness**
A model can be 100% "accurate" by saying "I don't know" to everything, but it is 0% useful. We decide here to balance **Groundedness** (staying true to facts) with **Helpfulness**.

---

### 0.4 The LLM Opportunity Canvas

This is your one-page contract. Before the ACME Assistant project kicks off, the Manager fills this out to align the Sages (stakeholders) and the Builders (engineers).

| Section | Description |
| --- | --- |
| **The User Pain** | "Employees spend 4 hours a week searching for HR policies." |
| **The AI Solution** | "A RAG-powered Assistant that answers from the internal handbook." |
| **Data Sources** | "Confluence, PDF Manuals, Slack #hr-faq." |
| **Critical Guardrails** | "Must never reveal salary data; must cite sources." |
| **Success Signal** | "40% reduction in 'Where is...?' tickets to HR." |

---

### 🛠️ Hands-On Lab: The Strategy Workshop

**Goal:** Create the strategic blueprint for the ACME Assistant.

1. **Draft the Canvas:** Fill out the LLM Opportunity Canvas for a "Knowledge Assistant."
2. **Identify the "Anti-Goal":** Explicitly list what the assistant **will not** do (e.g., "It will not give financial advice").
3. **Define the Baseline:** How is the problem solved today? (e.g., "Manual search in a messy Wiki").

**Acceptance Criteria:**

* A completed One-Page Canvas.
* At least three defined "Success Metrics" (1 Business, 1 User, 1 Technical).
* A signed-off "Risk Assessment" for hallucinations.

---

### Checklists: Strategy Readiness

* [ ] Do we have a specific user in mind?
* [ ] Is the data for this task accessible and clean?
* [ ] Have we defined what "Failure" looks like?
* [ ] Is a "Human-in-the-loop" possible for high-risk actions?

**What's Next:**
Now that the Manager has her "Job Description" and the Sages have signed off, it's time to build the workshop. **Chapter 1: The LLM Application Stack** is where we move from the drawing board to the machinery.
