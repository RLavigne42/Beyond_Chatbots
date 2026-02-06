## Chapter 13: LLMOps (Deployment & Monitoring)

The Manager had a brilliant Prodigy and a perfect scoreboard. Now, she needed to build the **Stadium**. It wasn't enough to have one Prodigy working at one desk; she needed a system that could handle thousands of users at once, stay awake 24/7, and alert her the second something went wrong. She needed to move from a "project" to a **Platform**.

In this chapter, we enter the world of **LLMOps**. We will learn how to deploy, scale, and monitor the ACME Assistant so it remains fast, cheap, and reliable under the pressure of real-world traffic.

---

### 13.1 The Production Pipeline: From Commit to Cloud

In a professional workshop, you don't just "copy-paste" code to the server. You use a **Pipeline**.

* **Continuous Integration (CI):** Every time a Builder changes a prompt or a line of code, the "Final Exam" from Chapter 12 runs automatically.
* **Continuous Deployment (CD):** If the exam passes, the new version is packaged into a "Container" (Docker) and shipped to the cloud.

---

### 13.2 Observability: The Manager’s Dashboard

The Manager can't watch every conversation, but she needs to see the **Health** of the system at a glance. We track three types of data:

1. **System Metrics:** CPU usage, memory, and API "Uptime."
2. **LLM Metrics:** Tokens per second, Latency (TTFT), and Cost per user.
3. **Quality Metrics:** Average "Groundedness" scores and user "Thumbs Up/Down" ratios.

**The What & Why: Tracing**

* **What it is:** Using a "Trace ID" (from Ch 1) to stitch together every step of a single request—from the user's question to the RAG search, the tool call, and the final answer.
* **Why we need it:** When a user complains, "The Assistant gave me a weird answer," tracing allows the Manager to replay the exact moment the "Prodigy" got confused.

---

### 13.3 Scaling: Handling the Crowd

**Trade-offs & Decisions: Vertical vs. Horizontal Scaling**

* **Vertical Scaling:** Buying a bigger, more expensive server. (Expensive and has a "ceiling").
* **Horizontal Scaling:** Adding more small servers and using a **Load Balancer** to distribute the work.
* **Our Decision:** We use **Horizontal Scaling** for our FastAPI app and **Provider Load Balancing** (e.g., rotating between two different API keys) to ensure we never hit "Rate Limits" during busy hours.

---

### 13.4 Semantic Caching: Saving Time and Money

**The Intuitive Analogy: The Frequently Asked Questions List**
If 100 people ask the Prodigy the exact same question ("What is the holiday policy?"), the Manager doesn't want to pay the Prodigy to think about it 100 times. We build a **Semantic Cache**. Before the request hits the model, we check a database: "Have we answered something very similar to this recently?" If yes, we serve the old answer instantly for zero cost.

---

### 🛠️ Hands-On Lab: "The Production Dashboard"

**Goal:** Set up a monitoring dashboard that tracks the cost and latency of your ACME Assistant in real-time.

**Step 1: The Logger**
Update your Orchestrator to calculate the `duration` of the request and the `token_count` of the response. Log this data to a structured file or a service like `Prometheus`.

**Step 2: The Semantic Cache**
Implement a simple cache using your Vector Database (from Ch 6). If a new query is 98% similar to a cached query, return the cached result instead of calling the LLM.

**Step 3: The Alert**
Create a simple "Watchdog" script. If the average latency stays above 5 seconds for more than a minute, send a "High Latency" alert to Slack.

**Acceptance Criteria:**

* A dashboard (even a simple console summary) showing: Total Cost, Avg Latency, and Cache Hit Rate.
* The system serves cached answers for identical queries in under 100ms.
* Every log entry contains the `prompt_version` and `model_id`.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Flying Blind":** If you aren't tracking your token costs daily, you will get a "Bill Shock" at the end of the month that could shut down the project.
* **"Scaling the Model, Not the App":** Most bottlenecks happen in your Python code or your database, not the LLM. Optimize your "Plumbing" before you buy a bigger GPU.

---

### Checklists: LLMOps Readiness

* [ ] Docker container builds successfully.
* [ ] Monitoring for "Rate Limit" errors (429s) is in place.
* [ ] Budget alerts are set up in your LLM provider's dashboard.

**What's Next:**
Our stadium is built, and the crowd is pouring in. But with great power comes great responsibility. In **Chapter 14: Safety, Security & Governance**, we build the final, unbreakable guardrails to keep our Prodigy—and our company—safe.

