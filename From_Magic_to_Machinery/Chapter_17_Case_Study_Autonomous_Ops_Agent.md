# Chapter 17: Case Study — Autonomous Ops Agent (Ticket Triage with Confirmations)

**Synopsis:** We’ll turn ACME Assistant into an operations co-pilot that triages incidents, proposes a safe, reversible plan, executes actions behind confirmation gates, and verifies or rolls back changes. You’ll define tool safety contracts, add policy-as-code, design a confirmation UX, and wire audit trails so humans stay in control.

**Estimated time:** 90–120 min read · 90–120 min lab  
**Prereqs:** Parts I–III; Ch.10 (agents), Ch.11 (systems integration), Ch.13 (LLMOps), Ch.14 (safety).

## Learning Objectives

By the end of this chapter, you can:

- Model ops runbooks as tool contracts with preconditions, idempotency, and rollback.
- Build a plan → review → execute → verify → rollback loop with human-in-the-loop confirmations.
- Enforce policy-as-code (RBAC, change windows, risk scoring) before any side-effects.
- Instrument an ops agent with traces, approvals, and tamper-evident audit logs.
- Evaluate with scenario goldens and safety drills (dry-runs, chaos faults).

## 17.1 Problem & Scope

Typical requests/incidents:

- “Error rate spiking on checkout-api in prod.”
- “Scale search-api to handle Black Friday traffic.”
- “Rollback the last deployment of payments.”

Constraints: Production safety first (dry-run by default), least privilege, reversibility, and explainability.

## 17.2 Architecture Overview

Signals (alerts, tickets, chat) → Triage Classifier → Planner → Policy Engine → Review & Confirmation → Executor → Post-Checks & Verification → Outcome (close/rollback/escalate) + Audit & Notifications

Cross-cutting: trace_id, approval_id, idempotency_key, tamper-evident audit.

> **Author's Note:** This end-to-end workflow is the most complex in the book and would greatly benefit from a clear diagram. The text-based flow should be converted into a full-page flowchart illustrating the journey of a signal from triage to final resolution, highlighting the key decision points like the Policy Engine and Human Confirmation gates.

## 17.3 Tool Catalog & Safety Contracts

Divide tools into read-only (safe) and mutating (require policy + confirmation).

**Read-only (examples):** `get_metric`, `query_logs`, `deployment_status`.

**Mutating (examples):** `k8s_rollout_restart`, `k8s_scale`, `feature_flag_set`.

**Contract schema (snippet):**

```json
{
  "name": "k8s_scale",
  "kind": "mutating",
  "preconditions": ["deployment_exists(namespace,name)"],
  "side_effects": ["changes_replicas"],
  "idempotency_key": "k8s_scale:{namespace}:{deployment}:{replicas}",
  "rollback": {"tool": "k8s_scale", "args_from": ["previous_replicas"]},
  "risk": {"base": 2, "multipliers": ["if env=='prod' then +2"]}
}
```

Rule: The agent never calls a mutating tool until: prechecks pass → policy approves → human confirms.

## 17.4 Planning Loop & Confirmation UX

**Plan format (structured):** A JSON object with `plan_id`, `summary`, `evidence`, `steps` (including kind, tool, args, success criteria), and a rollback plan.

**Human gate:** Render a diff-like review showing Why, What, Risk, Reversibility, and Policy checks, with buttons for Approve, Reject, Ask for change, Run as Dry-Run.

> **Design Review: The Human is the Co-Pilot, Not the Passenger**
>
> The confirmation gate is the most critical safety feature of an ops agent. It's essential to frame this interaction correctly: the agent is a co-pilot that proposes a plan, but the human operator is the pilot-in-command who makes the final decision.
>
> Your UX should empower this relationship. The Approve and Reject buttons are a minimum. A truly effective UI should also allow the operator to edit the plan—for example, changing a restart to a scale operation or adjusting a parameter. This keeps the human in full control, using the agent's proposal as a well-researched starting point rather than a take-it-or-leave-it demand.

## 17.5 Execution Engine

- Prechecks: existence, quotas, maintenance windows.
- Idempotency: include `idempotency_key` on each call.
- Circuit breakers: abort series on failed guard condition.
- Partial failure handling: stop → propose alternative → re-confirm.

> **Pro Tip: Make "Dry Run" the Default**
>
> For maximum safety, every mutating tool should operate in "dry run" mode by default. The execution engine should require an explicit `live_run: true` flag, which can only be set after a plan has passed all policy and human confirmation gates. This principle of "safe by default" ensures that an accidental or unapproved tool call can, at worst, only simulate a change, not cause one.

## 17.6 Verification & Rollback

- Define health gates (SLOs/SLIs) tied to the plan.
- Auto-rollback triggers: error rate worsens, rollout fails, verify step timeout.
- Post-verify soak timer (e.g., 5–10 minutes).
- Close ticket with a decision record (what changed, approvals, metrics before/after).

In the real world, monitoring systems can be flaky. A verification step might fail because the metrics API timed out, not because the system is unhealthy. To prevent unnecessary rollbacks, build resilience into your verification logic. For example, retry the health check 2-3 times with a short delay before declaring a failure. If the check is still failing, you can have a secondary, simpler health check (like a basic HTTP health endpoint) as a tie-breaker before triggering a full rollback.

## 17.7 Governance & Compliance

- Least privilege: each tool runs under the smallest service account needed.
- SOD (separation of duties): author ≠ approver for prod.
- Policy-as-code: evaluate plans with a rule engine before showing for approval.
- Audit: append-only log with hash chaining.

## Hands-On Lab: “Ops Agent with Confirmations (Mock K8s)”

**Goal:** Build a minimal agent that triages a ticket, proposes a restart, requires approval, executes, verifies, and rolls back on failure.

### Step 1 — Mock Environment

Create a fake Kubernetes API with in-memory state.

### Step 2 — Tool Adapters

Implement read-only and mutating tools.

### Step 3 — Policy Engine

Build a simple JSON-based rules evaluator.

### Step 4 — Planning Prompt

Create a prompt to generate a structured plan.

### Step 5 — Confirmation & Execution

Simulate a chat approval and execute the plan.

### Step 6 — Audit Chain

Implement the append-only, hash-chained log.

## Acceptance Criteria

- Mutating calls never executed without approval in prod.
- Each plan includes verify and rollback steps.
- On failed verify, rollback restores prior revision.
- Audit log contains trace_id, approval_id, and hash chain.

## Design Reviews & Anti‑Patterns

Open-ended tool calls. Always declare schemas, preconditions, timeouts.  
No verify/rollback. Every mutating step must pair with both.  
Hidden approvals. Store who approved what and when.  
Prompt-only safety. Enforce safety in code/policy; prompts are not controls.

## Checklists

### Tooling & Policy

- [ ] Tools categorized (read-only vs mutating), with contracts.
- [ ] Policy-as-code covers env, risk, SOD, windows.

### Execution Safety

- [ ] Confirmation gates for all mutating actions in prod.
- [ ] Health gates + timers; auto-rollback thresholds defined.

## Quick Quiz

- Why must every mutating action include both verify and rollback steps?
- Name three policy checks that should run before showing a plan for approval.
- What is the purpose of an idempotency key in tool execution?

## What’s Next

**Chapter 18 — Emerging Trends & Future Directions.**
