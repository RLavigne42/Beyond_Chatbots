## Chapter 17: Case Study — The Autonomous Ops Agent

The Assistant could find facts and analyze numbers, but the IT Ops team had the most grueling job of all: monitoring the company’s infrastructure 24/7. When a server ran out of memory or a service went down at 3:00 AM, a human had to wake up to fix it. The Manager realized the Prodigy could act as a **First Responder**.

In this chapter, we build an **Autonomous Ops Agent**. We move from "Text-to-Action" to **Proactive Intervention**, utilizing the **Managerial Approval Gate** to ensure that "Autonomous" doesn't mean "Dangerous."

---

### 17.1 The Challenge: High Stakes, High Velocity

In IT Operations, mistakes are expensive. If the Assistant restarts the wrong database, the company loses revenue.

* **The Risk:** Logic errors in an agent can lead to "Command Loops" where the AI repeatedly tries to fix a problem and accidentally makes it worse.
* **The Solution:** We implement **Standard Operating Procedures (SOPs)** as code.

---

### 17.2 The Implementation: The "Propose-Preview-Confirm" Loop

The Manager established a strict chain of command. The Assistant never acts in secret. Every action follows a three-step visibility cycle:

1. **Propose:** The Assistant analyzes the logs and says, "I see the `auth-service` is lagging. I recommend a graceful restart."
2. **Preview:** The Assistant shows the exact CLI command it intends to run (e.g., `kubectl rollout restart deployment/auth-service`).
3. **Confirm:** A human engineer receives a notification in Slack and must click **[Approve]** or **[Deny]**.

---

### 17.3 Tool Intelligence: The "Diagnostic" Agent

Before fixing anything, the Assistant must perform a **Diagnostic Run**. We give the Prodigy "Observability Tools" (from Ch 10):

* `check_logs(service_name)`
* `get_memory_usage(node_id)`
* `list_recent_deployments()`

**The What & Why: State Verification**

* **What it is:** Checking the system state *after* an action is taken to see if it worked.
* **Why we need it:** To prevent the "Looping Problem." If a restart didn't fix the lag, the Assistant shouldn't just keep restarting; it should stop and escalate to a human expert.

---

### 17.4 Safety: The Blast Radius

**Trade-offs & Decisions: Scoped Permissions**

* **Full Admin:** The Assistant can do anything. (Extreme risk).
* **Scoped Permissions (The Blast Radius):** The Assistant’s API key only has permission to "Restart" or "Clear Cache," never to "Delete" or "Modify IAM Roles."
* **Our Decision:** We use the **Principle of Least Privilege**. The Agent’s "Hands" are physically unable to touch the "Self-Destruct" buttons.

---

### 🛠️ Hands-On Lab: "The First Responder"

**Goal:** Build an agentic workflow that detects a mock "Low Disk Space" error and proposes a "Clear Logs" action.

**Step 1: The Event Trigger**
Create a script that sends a mock error message to your Assistant: `{"status": "error", "message": "Disk usage at 92% on Server-B"}`.

**Step 2: The Reasoning Chain**
The Assistant must use its `check_directory_size` tool to find the culprit (e.g., `/var/log`).

**Step 3: The Slack Approval**
Use a Slack Webhook to send a message with two buttons: **Approve Clean-up** and **Ignore**. Only when the "Approve" button is clicked does the `delete_old_logs` function run.

**Acceptance Criteria:**

* The Assistant correctly identifies the root cause before proposing a fix.
* No files are deleted unless the "Approve" signal is received by the Orchestrator.
* The system logs the `user_id` of the human who approved the action for auditing.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Silent Automation":** Never let an agent perform an "Update" or "Delete" without a log entry. If the system breaks, you need to know it was the AI that did it.
* **"The Feedback Loop of Death":** If the Agent’s action causes a new error, and the Agent tries to fix that error, you can spiral into a $10,000 API bill in minutes. Implement a "Kill Switch" for total requests per hour.

---

### Checklists: Ops Agent Readiness

* [ ] Human-in-the-loop (HITL) system is tested and reliable.
* [ ] API credentials for the Agent follow the Principle of Least Privilege.
* [ ] Escalation path (PagerDuty/Slack) is defined for when the AI is "Confused."

**What's Next:**
Our Assistant is now a researcher, an analyst, and a responder. We have reached the peak of the mountain. In our final chapters, we look at where the mountain goes from here. In **Chapter 18: Emerging Trends**, we look at the next generation of AI "Prodigies."
