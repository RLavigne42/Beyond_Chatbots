# Chapter 11: Systems Integration — APIs, DBs, and Workflows

**Synopsis:** LLMs speak JSON; businesses speak APIs, databases, and workflows. This chapter turns model outputs into safe, auditable function calls and transactions. You’ll design an integration layer with contracts, adapters, validation, idempotency, retries/circuit breakers, and compensations. We’ll cover read vs. write paths, queue-backed workflows, rate limits/backpressure, and end-to-end observability—then wire ACME Assistant into a CRM API and a warehouse with user confirmations.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Ch. 1–10; comfortable with REST/JSON, SQL, basic queuing.

## Learning Objectives

- Convert structured LLM outputs into validated function calls to APIs/DBs.
- Build an adapter layer (anti-corruption boundary) with schemas and policy checks.
- Implement idempotency, retries, exponential backoff with jitter, and circuit breakers.
- Design safe writes: propose → preview/diff → confirm → commit (with receipts).
- Orchestrate durable workflows (synchronous vs. async, sagas/compensations).

## 11.1 Integration Goals & Failure Modes

**Goals:** correctness, safety, explainability, and operability.

**Common failure modes:** schema drift, partial writes, non-idempotent endpoints, rate-limit storms, silent truncation.

Principle: LLM outputs never touch external systems directly. They pass through a contract-first integration layer that can say no.

## 11.2 Contracts: From Model to Function Calls

Define task contracts the model must produce, and tool contracts the system will execute. Validation path: `task_contract → tool_contract → policy checks` (RBAC/tenant/rate-limit).

## 11.3 Adapters (Anti-Corruption Layer)

Adapters translate domain-neutral model outputs into system-specific API calls.

- **Input adapters:** normalize enums, map locales, coerce types, sanitize strings.
- **Output adapters:** shape API responses to a consistent internal DTO.
- **Policy hooks:** check tenant scope, allowed fields, max changes, PII filters.

> **Pro Tip: Your Adapter is a Bouncer and a Translator 🛡️**
>
> The "Anti-Corruption Layer" is a powerful but abstract concept. A simpler way to think about your adapter is as a very strict nightclub bouncer who is also a professional translator.
>
> - **It's a Bouncer:** It stands between the creative, sometimes unpredictable LLM and your critical backend systems. It checks every incoming request (the LLM's output) against a strict list (the JSON schema and policies). If the request doesn't meet the rules (e.g., missing fields, invalid priority), it's rejected before it can cause problems inside.
> - **It's a Translator:** The LLM speaks a generic language of "intents" and "arguments." Your CRM speaks a very specific API dialect. The adapter translates the LLM's request into the exact format the backend system expects, ensuring perfect communication.

## 11.4 Validation & Decoding

Before any external call:

- Schema validate task JSON (strict).
- Decode to typed DTOs (Pydantic/dataclass).
- Policy guard: tenant, role, max risk.
- Redact: remove PII from logs/telemetry.

## 11.5 Idempotency, Retries & Circuit Breakers

- **Idempotency keys:** hash of (intent, normalized_args, user_id).
- **Retries:** exponential backoff + jitter; retry only on safe codes/timeouts.
- **Circuit breaker:** open on consecutive failures/timeouts.

A circuit breaker is an automatic switch that protects your system from failures in a downstream service. Like a circuit breaker in your house, it "trips" (opens) after a certain number of repeated failures (e.g., five consecutive timeouts from the CRM API). While open, it immediately rejects new calls without even trying, preventing your system from wasting resources on a service that is clearly down. After a cool-down period, it will allow a single "test" request through. If that succeeds, the breaker closes and normal operation resumes. This pattern prevents cascading failures.

## 11.6 Reads vs. Writes

- **Reads:** Parameterized SQL only; whitelist tables; row-level security by `tenant_id`.
- **Writes:** Propose → Preview → Confirm → Commit.

> **Author's Note:** The Propose → Preview → Confirm → Commit workflow is the most important safety pattern in this chapter. A clear sequence diagram showing the interaction between the User, the Orchestrator, the LLM, and the external API would be invaluable for the reader.

## 11.7 Workflows: Sync vs. Async, Sagas & Outbox

- Sync for quick, low-risk actions (<2–5s).
- Async for multi-step or flaky networks: enqueue work items.
- Saga pattern: chain local transactions with compensations.
- Outbox pattern: write domain event in same DB tx for reliable publishing.

When using async workflows, it's critical to have a Dead-Letter Queue (DLQ). If a message repeatedly fails to be processed by a worker (e.g., due to a persistent data validation error), instead of retrying forever, the message is moved to the DLQ. This gets the "poisoned" message out of the main queue and allows an operator to inspect it manually without halting the entire workflow.

## 11.8 SQL Integration Safety

Generate SQL via a structured plan → parameterized query. Ground to introspected schema; refuse unknowns. Quota guards: limit rows scanned/returned; paginate.

## 11.9 Rate Limits & Backpressure

- Leaky-bucket per tool & per tenant.
- 429 handling: respect Retry-After; re-queue with backoff.
- Admission control: shed low-priority tasks when saturated.

## 11.10 Secrets, Config, and Multi-Tenant Safety

- Secrets in a vault; short-lived tokens.
- Every request carries `tenant_id`; enforce at all boundaries.

## 11.11 Observability & Audit

- Traces: spans for validation, adapter map, client call, retries.
- Metrics: success rate, p50/p95 latency, retry count, breaker state, DLQ depth.
- Audit log: who, what, when, where, result, confirmation.

## Hands-On Lab: “Wire ACME Assistant to CRM + Warehouse (Safely)”

**Goal:** Add a CRM API integration (create ticket) and a warehouse integration (read-only analytics query) with propose/preview/confirm/commit for writes, idempotency, retries, and observability.

### Step 1 — Contracts & DTOs

Define schemas and data transfer objects.

### Step 2 — Adapters & Clients

Build clients with retries and an adapter for validation/mapping.

### Step 3 — Orchestrator Hooks

Extend the agent from Ch. 10 with the new tools.

### Step 4 — Observability

Add spans, metrics, and audit log entries.

### Step 5 — Evals

Create a test suite for malformed args, policy violations, and idempotency.

## Acceptance Criteria

- 100% of writes require explicit confirmation; receipt stored
- Idempotent duplicate ticket creates result in one ticket
- Breaker trips on ≥5 consecutive 5xx; auto-recovers
- Read queries parameterized; no SQL in prompts

## Design Reviews & Anti‑Patterns

Direct LLM → API writes. Always go through schema validators & adapters.  
No idempotency keys. Retries will duplicate tickets/orders.  
Free-form SQL. Only parameterized, schema-grounded queries.

## Checklists

### Integration Readiness

- [ ] Contracts (task & tool) defined with JSON Schemas
- [ ] Adapters map/validate; policy hooks enforce tenant/RBAC
- [ ] Clients have retries + jitter; circuit breakers configured

### Safety & Compliance

- [ ] Propose/preview/confirm/commit workflow for writes
- [ ] PII redaction in logs; secrets via vault
- [ ] Audit trail: who/what/when/where/result

## Quick Quiz

- Why are idempotency keys essential for LLM-initiated writes?
- When would you choose an async workflow over a sync call?
- How do you prevent SQL injection when the model proposes a query?

## What’s Next

**Chapter 12 — Evaluation & Evals**
