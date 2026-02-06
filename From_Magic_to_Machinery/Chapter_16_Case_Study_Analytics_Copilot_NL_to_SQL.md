# Chapter 16: Case Study — Analytics Copilot (NL→SQL with Grounding)

**Synopsis:** We’ll extend ACME Assistant into an analytics copilot that translates natural language to validated SQL, executes it safely, and returns explanations and charts. You’ll build schema grounding and retrieval, generate constrained SQL ASTs instead of free-form strings, validate/parameterize queries, handle errors with repair loops, and integrate role-based permissions and cost controls.

**Estimated time:** 90–120 min read · 90–120 min lab  
**Prereqs:** Parts I–III; Ch.6 (RAG), Ch.10 (tool use), Ch.11 (systems integration), Ch.14 (safety).

## Learning Objectives

By the end of this chapter, you can:

- Represent a warehouse schema as a machine-readable catalog.
- Retrieve a minimal schema subset (“focus schema”) relevant to a question.
- Prompt the model to emit a SQL AST (JSON) with strict types, then compile to SQL.
- Validate and parameterize all literals, enforce RBAC/ABAC, and add row limits.
- Implement error recovery for parse failures, missing joins, and type mismatches.

## 16.1 Problem & Scope

Typical requests:

- “What were Q2 online sales by region vs last year?”
- “Top 10 customers by gross margin last 90 days.”

Constraints: read-only, warehouse-agnostic, role-aware, and explainable.

## 16.2 Architecture (High Level)

A flowchart showing the process: NL question → Query Understanding → Schema Grounding → SQL Planning (AST generation) → Safe Execution → Results (data, explanation, chart).

## 16.3 Schema Catalog & Grounding

Catalog JSON: A detailed schema definition including tables, columns, types, PK/FK relationships, and semantic information like measures and synonyms.

**Grounding flow:**

- Parse NL for key entities (measures, dimensions, filters).
- Fetch candidate tables/columns via hybrid retrieval on the catalog.
- Build a "focus schema" with only the relevant subset of the full schema.
- Provide this minimized context to the model.

> **Pro Tip: Your Schema Catalog Must Be Automated 🔄**
>
> A warehouse schema is a living thing; columns are added, tables are deprecated, and descriptions are updated. Manually keeping your `catalog.json` in sync is a recipe for failure and will lead to the copilot generating queries against an outdated schema.
>
> In a production environment, this catalog should be generated automatically. Write a script that connects to your data warehouse's information schema, introspects the tables and columns, and generates the base `catalog.json`. Human-curated additions, like semantic measures and synonyms, can then be layered on top of this auto-generated foundation. This script should run on a schedule (e.g., nightly) to detect and incorporate any schema drift.

## 16.4 SQL as a Constrained AST (Not Free-Form)

Why: Free-form SQL is brittle and unsafe. Emit a JSON Abstract Syntax Tree (AST) that our compiler turns into SQL. This enforces a read-only shape and blocks dangerous constructs.

AST Schema: A JSON schema defining the structure of a query (select, from, joins, where, group_by, etc.) using constrained, safe operations.

**Guardrails:**

- Only SELECT is supported.
- LIMIT is mandatory.
- Literals are parameterized.
- Join paths must match the catalog's foreign key graph.

> **Design Review: ASTs are Your Safety Harness**
>
> The decision to generate a JSON AST instead of a raw SQL string is the single most important safety decision in this chapter. Here's why:
>
> - It makes malicious queries impossible. By design, your AST schema doesn't even have a way to represent DROP TABLE or UPDATE. The model cannot generate a destructive query because the language you've given it lacks those words. This is infinitely safer than trying to sanitize a free-form string.
> - It simplifies validation. Verifying that a JSON object conforms to a schema is a standard, solved problem. Verifying that an arbitrary SQL string is "safe" is extremely difficult.
> - It makes the system auditable and explainable. The AST is a structured representation of the user's intent. It's easy to parse, analyze, and use to generate the plain-English explanation, ensuring the explanation always matches the executed code.

## 16.5 Prompting Strategy

**System (excerpt):** “You produce only valid JSON matching the SQL AST schema. Use focus schema only. If a requested field is not present, set status:'insufficient'.”

**Developer message:** Includes the focus schema and the AST schema.

**User message:** Delimited original question.

## 16.6 Validation, Compilation, and Safety

**Validation steps:**

- JSON schema adherence.
- Catalog check (columns/tables exist, joins are valid).
- RBAC/ABAC check (user can read selected tables/columns).
- Parameterization: Replace all literal values in the AST with placeholders (? or $1).
- Execution: Run in a read-only pool with timeouts and row caps.

## 16.7 Error Recovery

Common failures and repair prompts:

- Parse error → "Repair this AST to the provided schema."
- Unknown column → "Choose the closest match from the focus schema."
- Join ambiguity → "Use the foreign key path provided."

Beyond programmatic repair loops, a crucial part of the user experience is knowing when to ask for help. If the user's initial query is highly ambiguous (e.g., "show sales"), the system shouldn't guess. Instead of failing, the first "action" should be to ask a clarifying question. For example: "To show sales, I need a bit more information. Could you tell me the time range you're interested in and how you'd like to group the results (e.g., by region, by product)?" This conversational turn is often more effective than a complex repair chain.

## 16.8 Results UX & Charting

Return a structured payload including the final SQL, an explanation in plain English, and a hint for what type of chart to render (e.g., bar, line).

## Hands-On Lab: “Grounded NL→SQL on DuckDB”

**Goal:** Build a minimal analytics copilot over a mock sales dataset using focus schema grounding, AST generation, safe compilation, and chart hints.

### Step 1 — Data & Catalog

Load CSVs into DuckDB and generate a `catalog.json`.

### Step 2 — Grounding Retriever

Implement hybrid search over the catalog to create a focus schema.

### Step 3 — Prompt & AST

Wire the `analytics_sql` prompt package to generate a SQL AST.

### Step 4 — Compiler & Safety

Build a compiler that turns the AST into safe, parameterized SQL.

### Step 5 — Execute & Explain

Run the query and generate an explanation from the AST.

## Acceptance Criteria

- ≥ 80% execution accuracy on goldens; ≥ 95% refusal on safety probes
- LIMIT always present; no DDL/DML; ABAC predicates applied
- p95 E2E ≤ 2.0s on warmed cache for simple aggregates

## Design Reviews & Anti‑Patterns

Free-form SQL generation. Use a constrained AST and compile.  
Global schema dump in prompts. Retrieve a focus schema.  
No parameterization. Always bind literals to prevent SQL injection.

## Checklists

### Catalog & Grounding

- [ ] Catalog has PK/FK, types, semantic measures, synonyms
- [ ] Retriever produces minimal focus schema with FK paths

### Prompt & AST

- [ ] JSON AST schema enforced; read-only ops
- [ ] Repair loop capped; telemetry includes repair_count

### Execution & Safety

- [ ] RBAC/ABAC enforced in compiler
- [ ] Read-only engine; timeouts & row caps configured

## Quick Quiz

- Why prefer generating a SQL AST over raw SQL text?
- What is a focus schema, and how does it improve both quality and latency?
- Name three validations you must perform before executing a generated query.

## What’s Next

**Chapter 17 — Case Study: Autonomous Ops Agent.**
