## Chapter 16: Case Study — The Analytics Copilot (NL→SQL)

The Librarian was a success, but the Sages (the executives) had a different pain point. They were tired of waiting days for the Data Team to write SQL queries for simple reports. They wanted to ask, "How many customers in Texas bought the XJ-900 last month?" and get an answer instantly. The Manager realized the Prodigy needed to become a **Data Analyst**.

In this chapter, we build a **Natural Language to SQL (NL→SQL)** engine. We move from retrieving text to generating code that interacts with structured data.

---

### 16.1 The Challenge: The "SQL Hallucination"

When asked to write code, the Prodigy can be "too creative." It might guess that a table is named `Customer_Orders` when the real database calls it `tbl_ord_2024`.

* **The Risk:** A hallucinated column name doesn't just give a wrong answer; it causes a system error.
* **The Solution:** We don't ask the model to guess. we provide the **Schema Context**.

---

### 16.2 The Implementation: Schema Injection

The Manager’s strategy is: "Show them the map, but don't give them the keys." We provide the Prodigy with a **Read-Only Schema Map**—a text description of our tables, columns, and data types.

**What & Why: Semantic Metadata**

* **What it is:** Adding descriptions to your columns (e.g., `cust_id`: "The unique ID for each customer").
* **Why we need it:** It helps the Prodigy understand that `rev_amt` actually means "Revenue Amount."

---

### 16.3 The Logic: The NL→SQL Workflow

We don't just run whatever code the Prodigy writes. We use a four-step safety loop:

1. **Draft:** The Prodigy writes the SQL based on the Schema Map.
2. **Explain:** The Prodigy explains in plain English: "I am going to count all customers where the state is TX..."
3. **Validate:** An automated script checks the SQL for "Forbidden Keywords" (like `DELETE` or `DROP`) and verifies it is valid SQL.
4. **Execute:** The result is fetched from a **Read-Only Replica** of the database and turned into a chart or table.

---

### 16.4 The "Golden Rule" of Data: Read-Only Replicas

**Trade-offs & Decisions: Production vs. Replica**

* **Production Access:** Fast, but the Prodigy could accidentally slow down the whole company with a "Heavy Query."
* **Read-Only Replica:** A "copy" of the data used only for questions. It’s safer and ensures that even if the Prodigy writes a "bad" query, the main business keeps running.
* **Our Decision:** We **always** use a Read-Only Replica for AI Analytics.

---

### 🛠️ Hands-On Lab: "Building the Analytics Copilot"

**Goal:** Create an endpoint that converts a natural language question into a valid SQL query against a mock database.

**Step 1: The Schema Prompt**
Create a `data_expert` prompt that includes a Markdown table of your database schema (Tables: `Users`, `Products`, `Sales`).

**Step 2: The SQL Sanitizer**
Write a Python function that uses a library (like `sqlglot`) to parse the model's SQL. If it finds any command that isn't a `SELECT`, block it immediately.

**Step 3: Visualization**
Update the Orchestrator so that if the result is a list of numbers, it returns a JSON object formatted for a charting library (like `Recharts` or `Chart.js`).

**Acceptance Criteria:**

* The Assistant correctly identifies which table to query for a given question.
* The system blocks any query containing `DROP`, `INSERT`, or `UPDATE`.
* The response includes both the data table and the SQL query used to find it (for transparency).

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"The Schema Dump":** Don't give the model 500 tables at once. Use a "Router" to pick the 5 most relevant tables for the specific question.
* **"Blind Execution":** Never run AI-generated SQL against a live production database without a timeout and a row limit (e.g., `LIMIT 100`).

---

### Checklists: Analytics Copilot Readiness

* [ ] Read-only database user/replica created.
* [ ] Schema descriptions (Metadata) added for all key tables.
* [ ] SQL validation and sanitization logic is active.

**What's Next:**
The Assistant can now find facts and analyze numbers. But the final frontier is **Action**. In **Chapter 17: Case Study — The Autonomous Ops Agent**, we see what happens when we let the Assistant manage the company's infrastructure—with the ultimate safety net in place.
