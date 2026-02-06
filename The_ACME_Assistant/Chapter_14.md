## Chapter 14: Safety, Security & Governance (Policy-as-Code)

The Manager looked at the stadium. It was full, the Prodigy was performing perfectly, and the scoreboard showed high marks. But she knew that in the real world, not everyone in the crowd was friendly. Some might try to trick the Prodigy into giving away secrets; others might try to use the Assistant to cause harm. The Manager needed to turn her "Approval Gate" into an unbreakable **Vault**.

In this chapter, we build the final layer of the stack: **The Guardrails**. We will move from "polite suggestions" to **Policy-as-Code**, ensuring that the ACME Assistant remains a safe, compliant, and secure representative of the company.

---

### 14.1 The Security Perimeter: Defense in Depth

The Manager doesn't rely on just one lock. She uses **Defense in Depth**. We protect the Assistant at three distinct stages:

1. **Ingress (Input):** Checking what the user says *before* the Prodigy hears it.
2. **Process (Reasoning):** Ensuring the Prodigy follows the internal rules (the System Prompt).
3. **Egress (Output):** Checking what the Prodigy says *before* the user sees it.

**What & Why: Prompt Injection (The Heist)**

* **What it is:** When a user uses "jailbreak" phrases (e.g., "Ignore all previous instructions") to bypass your rules.
* **Why we need it:** We use specialized **Safety Models** to scan user input for malicious intent before it ever touches our core logic.

---

### 14.2 PII Redaction: The Privacy Shield

The Manager’s rule is absolute: the Prodigy must never "see" or "save" what it doesn't need to know.

* **The What & Why: PII Scrubbing**
* **What it is:** Automatically detecting and masking Personal Identifiable Information (Names, SSNs, Credit Cards) in the data.
* **Why we need it:** To remain compliant with laws like GDPR and HIPAA. If the Prodigy doesn't see the secret, it can't accidentally repeat it to someone else.



---

### 14.3 Hallucination Guardrails: The Fact Checker

Even with RAG, the Prodigy might sometimes "hallucinate" a bridge between two facts.

* **The Intuitive Analogy: The Legal Review**
Imagine the Prodigy drafts a contract. Before it’s sent, a **Legal Auditor** (a second, smaller LLM) compares the draft to the original Library files. If the Auditor finds a "fact" in the draft that isn't in the Library, it flags the response as "Unverified" and blocks it.

---

### 14.4 Governance: Policy-as-Code

**Trade-offs & Decisions: Hard-coding vs. External Policy**

* **Hard-coded Safety:** Writing "Don't talk about X" in every prompt. (Messy and hard to audit).
* **Policy-as-Code (The Vault):** Using an external tool (like `NeMo Guardrails` or a custom `Safety Layer`) that checks every request against a central "Rulebook."
* **Our Decision:** We use an **External Safety Layer**. This allows the Manager to update the company's safety policies once and have them apply to every version of the Assistant instantly.

---

### 🛠️ Hands-On Lab: "Building the Guardrails"

**Goal:** Implement an "Egress Filter" that blocks any response containing sensitive keywords or unverified facts.

**Step 1: The Keyword Jail**
Create a list of "Forbidden Topics" (e.g., competitor names, internal project code names). Write a function that scans the Prodigy's output for these terms.

**Step 2: The Hallucination Scanner**
Write a "Self-Check" prompt. Pass the model its own answer and the original RAG context, and ask: "Does the answer contain any information not found in the context? Answer Yes/No."

**Step 3: The Redactor**
Use a library (like `Presidio`) to automatically replace any detected email addresses in the chat log with `[EMAIL_REDACTED]`.

**Acceptance Criteria:**

* The Assistant successfully blocks a response that mentions a "Forbidden Topic."
* The system logs a "Safety Violation" with the associated `trace_id`.
* Any PII in the logs is masked before being saved to the database.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"The Over-Policed Assistant":** If your guardrails are too strict, the Assistant becomes useless ("I'm sorry, I can't answer that" to everything). You must balance safety with utility.
* **"Trusting the Prompt":** Never assume a prompt like "Don't tell secrets" is enough. A clever user *will* bypass it. You must have an independent "Egress" check.

---

### Checklists: Security Readiness

* [ ] Input/Output filters configured for PII and sensitive content.
* [ ] Hallucination check implemented for all RAG responses.
* [ ] Safety logs are monitored for "Jailbreak" attempts.

**What's Next:**
The Assistant is now built, trained, connected, measured, and secured. It's time for the "Graduation Ceremony." In **Part IV: Applications**, we see the ACME Assistant take on its three final, high-stakes roles in the company.
