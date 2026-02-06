# Chapter 10: Agents & Tool Use

**Synopsis:** Models predict text; agents turn that text into actions. In this chapter you’ll design a safe, auditable agent that plans, chooses tools, validates arguments, and executes with user confirmations. You’ll compare ReAct (reason-act interleaving) vs. Plan-Execute, create a tool catalog with contracts, add sandboxes and deterministic subroutines, and wire audit logs + dry-run so actions are explainable and reversible.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Ch. 1–9; basic SQL and HTTP APIs.

## Learning Objectives

- Pick a planning loop (ReAct vs. Plan-Execute) for a given task & SLOs.
- Define tool contracts (schemas, auth, rate limits) and validate arguments.
- Implement safety controls: user confirmations, dry-runs, scopes, timeouts, and idempotency.
- Add sandboxes (network/FS permissions) and deterministic subroutines (calc, regex, SQL).
- Trace and audit every step with structured reasoning artifacts and replayable logs.

## 10.1 What Is an Agent?

An agent is an orchestrated loop:

- Perceive (read goal, state, context)
- Plan (decide next action/tool)
- Act (call tool deterministically)
- Observe (ingest tool outputs)
- Repeat / Stop (stopping rules, confirmations)

The model chooses which tool and with what arguments, but tools perform the actual side-effects deterministically and safely.

> **Design Review: When to Build a Tool 🛠️**
>
> Before building an agent, always ask: should this be a tool, or can it be handled by a prompt? A task is a good candidate for a tool if:
>
> - It requires determinism: LLMs are terrible at precise math. A calculator tool is non-negotiable.
> - It needs live data: The model's knowledge is static. A web.search tool is required for real-time information.
> - It interacts with a private system: The model can't access your company's CRM. An update_crm tool is the only way to perform that action.
> - It's a security risk: You should never put database connection strings or raw SQL in a prompt. A sql.query tool provides a safe, sandboxed interface.
>
> If a task doesn't meet these criteria (e.g., reformatting text, summarizing content), a well-crafted prompt pattern from Chapter 4 is often a simpler and more efficient solution.

## 10.2 Planning Loops

**ReAct (Reason + Act, interleaved)**

- **Pros:** adaptive; good for open-ended research and decompositions.
- **Cons:** risk of tool flailing; must cap steps and add stop conditions.

**Plan-Execute**

- **Pros:** stable: first produce a plan (steps, tools, success criteria), then execute deterministically.
- **Cons:** less flexible if the plan is wrong; add mid-execution re-planning triggers.

**Selection Heuristic**

- Exploratory tasks (web research, multi-hop): ReAct with tight step caps.
- Transactional tasks (create ticket, send email, update CRM): Plan-Execute with confirmations.

> **Author's Note:** The difference between these loops is best understood visually. We should include a figure with two side-by-side flowcharts to illustrate the control flow of ReAct versus Plan-Execute. The ReAct diagram should show a tight loop of (Thought → Act → Observe). The Plan-Execute diagram should show two distinct phases: a first phase where the entire plan is generated, and a second phase where the steps are executed sequentially.

## 10.3 Tool Catalog & Contracts

Define each tool as a contract—not a prompt:

```json
{
  "name": "calculator.evaluate",
  "description": "Evaluate a pure arithmetic expression.",
  "args_schema": {
    "type": "object",
    "required": ["expression"],
    "properties": {
      "expression": {"type": "string", "pattern": "^[0-9+\\-*/().\\s]+$"}
    }
  },
  "returns": {"type": "object", "properties": {"value": {"type": "number"}}},
  "auth": "none",
  "rate_limit": "60/m",
  "side_effects": false,
  "sandbox": "no-net, cpu=small, time=100ms"
}
```

Catalog fields (minimum): name, description, args_schema, returns, auth, rate_limit, side_effects, sandbox, idempotency_key?.

Side-effects must be explicitly marked; require user confirmation and dry-run result before commit.

Idempotency: tools that write must accept `idempotency_key` and be safe to retry.

> **Pro Tip: Idempotency is Your Safety Net**
>
> Idempotency is the property of an operation that allows it to be applied multiple times without changing the result beyond the initial application. In plain English: calling a tool once should have the same effect as calling it five times in a row.
>
> Example: A `charge_customer(order_id: "123", amount: 50)` tool must be idempotent. If network issues cause your orchestrator to retry the call, the customer should only be charged once. This is typically handled by passing an `idempotency_key` and having the receiving system track which keys it has already processed.

## 10.4 Tool Safety & Argument Validation

- Validate arguments before invoking a tool (JSON Schema + custom validators).
- Reject/repair once with a targeted prompt: “Argument validation failed: reason → propose repaired args.”
- Enforce rate limits, timeouts, and circuit breakers per tool.
- Least privilege auth tokens; scope to tenant and tool; short TTL.

## 10.5 Sandboxing & Determinism

- Pure functions (calc, regex, date math): run locally, deterministic, no network.
- Web/search: isolate in a fetcher service; strip scripts; limit domains if needed.
- SQL: read-only account for analytics; writes require staging + diff + confirm.

## 10.6 Stopping Rules & Confidence

- Max steps (e.g., ≤6 tool calls).
- No-progress detector: if last 2 steps produce identical observations, stop and ask for guidance.
- Confidence check: verification prompt (“Are constraints satisfied? Cite evidence.”) or a small verifier model.
- Budget guard: cap tokens/spend; degrade to summary of partial results.
- Tool flailing detector: A common failure mode in ReAct loops is "flailing," where the agent gets stuck. Detect this by tracking the agent's state. If the agent calls the same tool with the same arguments twice in a row, or cycles between two actions without making progress, it's flailing. The stop condition should be to halt the loop and report `status:"uncertain"` with the reason "Failed to make progress."

## 10.7 Confirmations & Dry-Runs

For side-effects (tickets, emails, DB writes):

- Dry-run: generate proposed change (diff/SQL preview).
- Summarize risk: fields changed, blast radius, rollback plan.
- Confirm UX: “Proceed / Edit / Cancel.”
- Commit: include `idempotency_key` and save a receipt (external ID, timestamp).

## 10.8 Observability & Audit

- Trace graph: nodes = plan/act/observe steps; edges = dependencies.
- Log prompt_id, plan version, tool invocations, arguments hash, results hash, confirmations, actor (user vs. system).
- Replay: ability to simulate with recorded observations for debugging.

## 10.9 Patterns: Research + Calc + SQL

- **Research:** web.search → scrape.clean → summarize.with.citations
- **Calc:** calculator.evaluate for numerics—never ask the LLM to compute.
- **SQL:** NL→SQL (structured output) → parameterized query (read-only).

## Hands-On Lab: “Research-and-Calc Agent with SQL”

**Goal:** Build an agent that (a) does web research with citations, (b) performs deterministic calculations, and (c) queries a SQL DB, with confirmations for any writes.

### Step 1 — Define Tools (contracts)

`web.search`, `web.fetch`, `calculator.evaluate`, `sql.query`, `sql.propose_update`.

### Step 2 — Choose Planning Loop

Implement Plan-Execute as default.

### Step 3 — Orchestrator Skeleton (pseudo)

Write the main loop that validates plans, handles confirmations, executes steps, and logs results.

### Step 4 — Safety & Policy

Enforce domain allow-lists and read-only SQL.

### Step 5 — Evals

Test task success, citation coverage, and tool accuracy.

## Acceptance Criteria

- Plans are valid JSON; 0 schema errors after one repair attempt
- Citations present and resolvable for research answers (≥ 90% coverage)
- All calculations done via `calculator.evaluate`
- Trace graph recorded per run with tool timings and outcomes

## Design Reviews & Anti‑Patterns

Free-form tool calls. Always enforce schemas & validation pre-invoke.  
LLM doing math. Route to deterministic calculators.  
Unbounded ReAct loops. Step caps + no-progress detection are mandatory.  
Silent side-effects. Every write has dry-run, human confirmation, and idempotency.

## Checklists

### Tooling Readiness

- [ ] Contracts defined with JSON Schemas and return types
- [ ] Sandboxes and least-privilege auth in place
- [ ] Idempotency for side-effect tools

### Agent Readiness

- [ ] Planning loop chosen (default Plan-Execute) with step caps
- [ ] Confirmations + dry-run UX for side-effects
- [ ] Stop rules, re-plan triggers, and budget guards

## Quick Quiz

- When would you choose ReAct over Plan-Execute, and why?
- Name three fields every tool contract must include.
- What’s the safest way to handle DB writes initiated by an agent?
- How do you detect “tool flailing,” and what’s your stop condition?

## What’s Next

**Chapter 11 — Systems Integration: APIs, DBs, and Workflows**
