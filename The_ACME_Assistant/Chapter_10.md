## Chapter 10: Agents & Tool Use

The Prodigy could now remember faces and facts, but they were still "trapped in a box." They could tell a user how to file an expense report, but they couldn't actually file it. They could explain that a server was down, but they couldn't restart it. The Manager realized that for the Prodigy to be truly valuable, they needed **Hands**.

In this chapter, we move from "Chat" to **Agents**. We will teach the Prodigy how to use the company’s software tools safely, transforming the Assistant from a talker into a **Doer**.

---

### 10.1 The Tool Contract: Defining the "Hands"

The Manager didn't just give the Prodigy a blank check. She gave them a specific **Toolbelt**. Each tool in that belt came with a strict set of instructions—a **Tool Contract**.

**What & Why: Function Calling**

* **What it is:** A structured way to tell the model: "Here are the buttons you are allowed to press and the information I need from you to press them."
* **Why we need it:** It forces the model to provide precise parameters (like an ID number or a date) instead of just vague descriptions.

---

### 10.2 The Reasoning Loop: Plan → Tool → Observe

**The Intuitive Analogy: The Repair Kit**
Imagine the Prodigy is a technician sent to fix a leaky pipe.

1. **Plan:** They look at the pipe and decide they need a wrench.
2. **Tool:** They reach into the kit and grab the wrench (the tool call).
3. **Observe:** They turn the wrench and check if the leaking stopped.
4. **Repeat:** If it’s still leaking, they try a different tool.

This loop—often called **ReAct** (Reasoning and Acting)—is what turns a simple model into an **Agent**.

---

### 10.3 Tool Security: The Sandbox

**Trade-offs & Decisions: Direct vs. Mediated Access**

* **Direct Access:** Letting the Prodigy write and run its own code or talk directly to the production database. (Extremely dangerous; high risk of "The Heist").
* **Mediated Access (The Sandbox):** The Prodigy can only call specific, pre-written functions that have been vetted by the Manager.
* **Our Decision:** We use **Mediated Access**. The Prodigy never "writes" a database query; it only calls a `get_user_by_email()` function that *we* wrote and secured.

---

### 10.4 The Golden Rule: Human-in-the-Loop (HITL)

**The What & Why: The Managerial Sign-off**

* **What it is:** For any tool that is **mutative** (changes, deletes, or spends something), the Agent must pause and show its plan to a human.
* **Why we need it:** To prevent the Prodigy from accidentally deleting a database or sending a confusing email to a client while "trying to be helpful."

---

### 🛠️ Hands-On Lab: "Giving the Assistant Hands"

**Goal:** Implement a "Weather" and "Holiday Calculator" tool that the Assistant can call to answer complex user requests.

**Step 1: Define the Tools (JSON Schema)**
Create a schema for a `get_holiday_balance` tool that requires a `user_id` and a `year`.

**Step 2: The Tool Dispatcher**
Write a "Dispatcher" in your FastAPI code. If the model says, "I want to call `get_holiday_balance`," the Dispatcher runs the actual Python code and feeds the result back to the model.

**Step 3: The "Proposed Action" UI**
Create a special response type where, if the model wants to *book* a holiday, it returns a "Proposal" that the user must click "Confirm" on before the code actually runs.

**Acceptance Criteria:**

* The Assistant successfully detects when it needs to use a tool to answer a question.
* The Assistant correctly extracts arguments (like `user_id`) from the conversation.
* The Assistant waits for a "Confirm" signal before executing any mock "delete" or "book" action.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Swiss Army Knife Prompts":** Don't give the Agent 50 tools at once. It will get confused and "hallucinate" tool calls. Give it only the tools relevant to the current task.
* **"Implicit Execution":** Never, ever let an Agent perform an irreversible action (like deleting a user) without an explicit human "OK."

---

### Checklists: Agent Readiness

* [ ] Tools are defined with strict Pydantic schemas.
* [ ] All tool code is "Read-Only" unless a human-in-the-loop is active.
* [ ] Error handling is in place: if a tool fails, the Agent is told *why* so it can try again.

**What's Next:**
Our Prodigy can now talk, remember, and act. But to work in the real world, they need to play well with others. In **Chapter 11: Systems Integration**, we look at how to connect our Assistant to the complex web of APIs and workflows that make up the modern office.
