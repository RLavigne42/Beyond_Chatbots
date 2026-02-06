## Chapter 1: The LLM Application Stack

The Manager had her job description. Now, she needed to build the **Onboarding Framework**. She knew that if she just sat the Prodigy down at a random desk with a blank sheet of paper, the work would be a mess. She needed to build a "Workstation" that enforced rules, tracked progress, and ensured that every output followed a standard format.

In this chapter, we move from strategy to **Machinery**. We are going to build the basic skeleton of the ACME Assistant: a structured, instrumented API service.

---

### 1.1 The 9-Layer Blueprint

Before we write code, we must understand the architecture. An LLM application isn't just a "prompt and a response." It is a stack of nine cooperating layers that wrap around the model to make it behave.

**What & Why: The Orchestrator**

* **What it is:** The "brain" of our code (often called the Orchestrator) that coordinates between the user, the model, and our data.
* **Why we need it:** Models are **stateless**. They don't know who the user is, what time it is, or what happened ten seconds ago unless the Orchestrator tells them.

---

### 1.2 Contract-First Design: Standard Operating Procedures

The Manager's first rule for the Prodigy was strict: "Do not give me a wall of text. Fill out this form." In software, we call this a **Schema**.

**The Intuitive Analogy: The SOP Template**
If the Prodigy writes a report differently every day, the rest of the team can't use it. By defining a **JSON Schema**, we are giving the Prodigy a "Standard Operating Procedure." We tell the model exactly what keys we expect (e.g., `answer`, `confidence`, `sources`).

**Trade-offs & Decisions: Structured vs. Natural Language**

* **Natural Language** is easier for the model but breaks your UI.
* **Structured Output (JSON)** is harder for the model but makes your system reliable.
* **Our Decision:** We always use **Contract-First Design**. If the model fails the contract, we don't show the answer to the user; we send it back for repairs.

---

### 1.3 Setting Up the Workshop: The Stack

We will build the ACME Assistant using a modern, lightweight stack:

* **Language:** Python (The industry standard for AI).
* **API Framework:** FastAPI (Fast, typed, and generates documentation automatically).
* **Validation:** Pydantic (To enforce our "Contracts").
* **Observability:** Basic logging with `trace_id` (To track every single conversation).

---

### 🛠️ Hands-On Lab: "Hello, Structured World"

**Goal:** Build a minimal FastAPI service that takes a question and returns a strictly formatted JSON response.

**Step 1: Define the Contract (The Schema)**
Create a Pydantic model that defines the shape of our response.

```python
class AssistantResponse(BaseModel):
    title: str
    answer: str
    sources: List[str]

```

**Step 2: Create the Orchestrator**
Build a simple endpoint `/ask` that accepts a user query. For now, we will mock the LLM response to ensure our "Contract" works.

**Step 3: Instrumentation**
Every request must generate a `trace_id`. This is the "ID Badge" for every task the Prodigy performs, allowing the Manager to look back and see exactly what happened if something goes wrong.

**Acceptance Criteria:**

* `POST /ask` returns a 200 OK.
* Response body matches the `AssistantResponse` schema exactly.
* Every response header includes a `X-Trace-ID`.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Chat First, Structure Later":** Never start with a raw text box. If you can't parse the data, you can't build a product.
* **"Silent Failures":** If the model returns bad JSON, don't just let the app crash. Catch the error, log the `trace_id`, and return a helpful error message.

---

### Checklists: Framework Readiness

* [ ] Python environment and FastAPI installed.
* [ ] Pydantic schemas defined for inputs and outputs.
* [ ] Logging configured to capture `trace_id` and latency.

**What's Next:**
Our "Workstation" is ready, but the chair is empty. Now we need to find the right person for the job. In **Chapter 2: Model Selection & Architecture**, we will run a "Bake-Off" to find the perfect model (the perfect Prodigy) to power our Assistant.
