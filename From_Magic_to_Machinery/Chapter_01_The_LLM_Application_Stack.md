# Chapter 1: The LLM Application Stack

**Synopsis:** A big‑picture tour of the layered architecture for production LLM systems. We establish shared vocabulary, a reference diagram, and the minimum viable patterns you’ll use throughout the book. You’ll leave with a mental model and a tiny, instrumented app you can extend in later chapters.

**Estimated time:** 60–90 min read · 45–60 min lab  
**Prereqs:** Python basics, JSON, command line.

## Learning Objectives

By the end of this chapter, you can:

- Map any LLM use case onto a layered architecture.
- Explain responsibilities and interfaces for each layer (model, prompts, data/RAG, memory, tools, evals, safety, ops).
- Define a contract-first interaction (structured outputs + validation) between an orchestrator and a model.
- Stand up a minimal, streaming LLM service with logging, request IDs, and a golden test set.

## Mental Model

Think of an LLM app as orchestration around a stateless model. The model predicts tokens; the system supplies purpose, context, guardrails, and persistence. Every reliable product emerges from nine cooperating layers:

1) Core Model
2) Prompts
3) Specialization (RAG / Fine‑tune)
4) Data Foundation
5) State/Memory
6) Tools/APIs
7) Observability/Evals
8) Safety/Governance
9) Performance/Reliability

> **Design Review: Why This Stack? A Story of Failures**
>
> Why can't you just wrap a simple API call in a web framework and ship it? You can, but it will break. This entire stack exists because simple solutions fail in predictable ways. Consider the evolution of a failing application:
>
> - **The Brittle UI:** You launch your app. The first model update changes the output format from `{"answer": "..."}` to `{"response": "..."}`. Your entire front-end, which expected the answer key, breaks. This motivates Layer 2 (Prompts), where you enforce a strict output schema, making your application a contract-first system.
> - **The Stale Knowledge:** A user asks about recent company news, but the model's knowledge cutoff is two years ago. It politely apologizes for being out of date. This motivates Layer 3 & 4 (Specialization & Data Foundation), creating a RAG pipeline to inject fresh, proprietary data at runtime.
> - **The Unbearable Slowness:** Users complain that the app feels sluggish. You realize every query goes to your most powerful—and slowest—model. This motivates Layer 9 (Performance & Reliability), where you introduce caching and a multi-model cascade to handle simple queries quickly and cheaply.
> - **The Dangerous Answer:** The model gives dangerously incorrect advice on a compliance question, causing a near-disaster. This motivates Layer 8 (Safety & Governance), adding guardrails, filters, and human-in-the-loop confirmations for sensitive topics.
>
> Each layer in our stack is a response to a real-world failure mode. By starting with this architecture, we are building a system that anticipates and mitigates these failures from day one.

## Reference Diagram (to be redrawn later)

```
User ↔ UI/Client
│
▼
Orchestrator / Router ───────────────────────────────────────────────────────────────
│ │ │ │ │ │
│ │ │ │ │ │
Prompts Retrieval (RAG) Memory Tools/APIs Safety Observability
│ │ │ │ │ │
└────────────┴───────┬─────┴────────────┴──────────────┴──────────────┘
▼
LLM Gateway → Model(s) (primary, fallback, batch)
│
▼
Structured Output / Stream → UI
```

Two cross‑cutting concerns wrap all calls: Safety (input/output filtering, confirmation for side‑effects) and Observability (traces, metrics, evals).

## Layer‑by‑Layer: Responsibilities, Interfaces, Metrics, Pitfalls

### 1) Core Model

- **Purpose:** Token prediction engine.
- **Interface:** `(instructions, context, input) → text|json` (stream or batch)
- **Decisions:** Model family/size; context window; function calling/JSON mode; cascade/fallback policy.
- **Health Metrics:** accuracy on golden tasks, token latency (ttft/tbt), cost/request, error rate.
- **Pitfalls:** Single‑model monoculture; ignoring structured outputs; letting the model choose formats ad‑hoc.

### 2) Prompts (Control Plane)

- **Purpose:** Direct model behavior and output shape.
- **Interface:** Versioned templates + schemas; feature flags; A/B switch.
- **Decisions:** System vs. user roles; JSON‑first schemas; critique/refine patterns; delimiting and injection resistance.
- **Health Metrics:** schema adherence %, rollback frequency, prompt diff → quality delta.
- **Pitfalls:** Prompt sprawl without versioning; hidden dependencies; mixing instructions with user data.

### 3) Specialization (RAG & Fine‑Tuning)

- **Purpose:** Inject fresh knowledge (RAG) and adjust behavior/style (fine‑tune/PEFT).
- **Interface:** `retrieve(query|emb) → docs`; model registry for tuned variants.
- **Decisions:** Hybrid retrieval (BM25+vector), reranking; what to tune vs. what to retrieve.
- **Health Metrics:** groundedness/citation coverage; retrieval recall@k; drift/regression rates of tuned models.
- **Pitfalls:** Over‑chunking; stale indexes; using fine‑tune to encode facts that change.

### 4) Data Foundation

- **Purpose:** Ingest, clean, chunk, embed, index; maintain lineage & freshness.
- **Interface:** Incremental ingestion jobs; metadata schema (source, timestamp, access controls).
- **Decisions:** Chunk size/overlap respecting structure; deduplication; embedding model; index type (HNSW/IVF).
- **Health Metrics:** duplicate ratio, coverage, embedding refresh lag, index recall/latency.
- **Pitfalls:** Blind PDF dumps; no lineage; unbounded PII exposure; one‑time ingestion with no updates.

### 5) State & Memory

- **Purpose:** Persist context across requests/sessions/users with privacy.
- **Interface:** session buffer, summaries, vector memory; TTLs & redaction policies.
- **Decisions:** What to remember; summarization cadence; consent; encryption at rest.
- **Health Metrics:** retrieval hit‑rate from memory, context length utilization, privacy incidents.
- **Pitfalls:** Storing raw transcripts forever; mixing users’ data; context bloat.

### 6) Tools & Integrations

- **Purpose:** Let the LLM call deterministic capabilities (DB, search, calculators, business APIs).
- **Interface:** Tool contracts (schema, auth, rate limits), confirmations for side‑effects.
- **Decisions:** Allowlist vs. catalog; sandboxing; retries/idempotency; transactional boundaries.
- **Health Metrics:** tool success %, rollback count, mean tool latency, incident rate.
- **Pitfalls:** Free‑form tool calls; no human confirmation; lack of idempotency.

### 7) Observability & Evals

- **Purpose:** Trace every request; prevent regressions.
- **Interface:** trace IDs, prompt/response capture (PII‑aware), golden sets, CI gates.
- **Decisions:** What to log and retain; eval rubrics; pass/fail thresholds; shadow testing.
- **Health Metrics:** pass rate on goldens, schema violations, hallucination score, SLOs.
- **Pitfalls:** Shipping prompt changes without tests; no rollback; dashboards without alerts.

### 8) Safety & Governance

- **Purpose:** Keep behavior compliant and secure.
- **Interface:** input/output filters, policy engine, permissioning, audit log.
- **Decisions:** refusal styles, PII redaction, role‑based access, rate limits.
- **Health Metrics:** violation rate, user‑confirmed actions %, false‑positive filters.
- **Pitfalls:** One‑time red team only; no user confirmation for irreversible actions.

### 9) Performance & Reliability

- **Purpose:** Balance cost/latency/quality; fail gracefully.
- **Interface:** caching, batching, streaming, speculative decoding; circuit breakers; fallback models.
- **Decisions:** cache keys/TTL; cascade policy; timeouts; partial results UX.
- **Health Metrics:** p95 latency, cost/query, fallback rate, cache hit‑rate, error budgets.
- **Pitfalls:** Lowest‑latency at any cost (quality crater); largest model everywhere (cost blow‑up).

## Minimal Viable Pattern: Contract‑First Orchestration

1. Define an output schema (JSON).
2. Write a system prompt that binds the contract (never free‑form text).
3. Assemble context (recent chat, retrieved docs, memory).
4. Call the model in streaming mode; incrementally validate/repair JSON chunks.
5. Log everything with a `trace_id`; attach retrieved doc IDs.
6. Enforce safety (refusals, PII masking) before emitting to UI.

**Pseudocode (language‑agnostic):**

```
schema = {...}  # JSON Schema dict
msg = build_messages(system=SYSTEM_PROMPT(schema), user=user_input, context=ctx)
trace_id = new_trace_id()
with llm.stream(msg, response_format="json") as stream:
    buffer = ""
    for chunk in stream:
        buffer += chunk
        maybe_emit_partial(parse_json_safe(buffer))
result = validate_or_repair(parse_json_safe(buffer), schema)
log(trace_id, messages=msg, result=result, context_ids=ctx.ids)
return result
```

## Hands‑On Lab: “Hello, Structured World”

**Goal:** Build a tiny HTTP service that accepts a question and returns a structured JSON answer (title, answer, sources[]), streamed to the client, with tracing and a golden test.

### 1. Scaffold

Create a new repo: `acme-hello-llm` with folders:

- `/service` (FastAPI or similar)
- `/prompts`
- `/tests`
- `/scripts`
- `/docs`

Add a `.env.example` with `MODEL=` and `API_KEY=` placeholders.

### 2. Contract & Prompt

**JSON Schema**

```json
{
  "type": "object",
  "required": ["title", "answer", "sources"],
  "properties": {
    "title": {"type": "string"},
    "answer": {"type": "string"},
    "sources": {"type": "array", "items": {"type": "string"}}
  }
}
```

**System Prompt (excerpt):**

> You are a precise assistant. Always return ONLY valid JSON matching the provided schema. If you lack sufficient information, set sources to [] and explain limits in answer.

### 3. Service (FastAPI sketch)

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
import json, uuid

app = FastAPI()

@app.post("/ask")
async def ask(req: Request):
    body = await req.json()
    q = body.get("q", "")
    trace_id = str(uuid.uuid4())
    # assemble messages: system + user (omitted for brevity)
    def token_stream():
        buffer = ""
        for tok in llm_stream(messages, json_mode=True):
            buffer += tok
            # Optionally emit well‑formed partials
            yield tok
        # persist trace with buffer
        save_trace(trace_id, messages, buffer)
    return StreamingResponse(token_stream(), media_type="text/event-stream")
```

### 4. Safety & Validation

- On stream completion: `validate_or_repair(json, schema)`. If unrecoverable → return a structured error object `{error, trace_id}`.
- Redact PII patterns from answer before emitting.

### 5. Observability

Log: `trace_id`, latency (TTFT, TBT), token counts, model name, prompt hash. Store the last 100 traces locally for inspection (rotate daily).

### 6. Golden Tests

Add `/tests/goldens.yaml` with 5–10 Q→expected patterns (regex on answer, required keys, max tokens). Create a test runner that calls `/ask` with a mock model or a fixed seed and asserts schema adherence + minimal content checks.

## Acceptance Criteria

- API streams tokens (or chunks) under 2s TTFT on dev hardware.
- 100% schema adherence on the golden set.
- Traces persisted with `trace_id` and prompt hash.
- Clear refusal on ambiguous/unsafe inputs.

## Stretch Goals

- Client SSE viewer with a progress bar.
- Add a tiny BM25 retrieval over `/docs` and include 0–2 file names in sources.
- Implement a simple fallback model on timeout.

## Design Reviews & Anti‑Patterns

- “Chat first, structure later.” Invert it: design the schema and tests before prompts.
- “One prompt to rule them all.” Maintain a prompt package per task; version and test.
- “RAG as a band‑aid.” Fix data quality and chunking before tuning retrieval knobs.
- “Log everything forever.” Keep PII out of logs; set TTLs and access controls.
- “Biggest model everywhere.” Start with cascades and caching; measure.

## Checklists

### Architecture Readiness

- [ ] Output schema defined and documented
- [ ] Prompt template versioned (with ID) and testable
- [ ] Safety policy (PII, refusals) integrated
- [ ] Tracing enabled (request/response, latency, tokens)
- [ ] Golden tests and regression gate in place
- [ ] Cost Visibility: Is there a mechanism to track or estimate the cost per request or per user?

### Operational Readiness

- [ ] Timeouts, retries, fallback policy
- [ ] Basic cost dashboard (tokens × price)
- [ ] Error budget & alert thresholds agreed

## Quick Quiz

- Name three cross‑cutting concerns that apply to every model call.
- What belongs in the orchestrator vs. the prompt?
- When would you pick retrieval over fine‑tuning for specialization?
- List two metrics that indicate your RAG is healthy.

## What’s Next

**Chapter 2 — Model Selection & Architecture:** we’ll run a lightweight bake‑off across three models, introduce cascades and structured output modes, then wire your selection back into the app you built here.
