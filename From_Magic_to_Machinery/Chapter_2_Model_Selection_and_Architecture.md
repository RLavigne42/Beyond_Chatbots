# Chapter 2: Model Selection & Architecture

**Synopsis:** Choose, test, and wire the right foundation model(s) for your product. You’ll run a disciplined bake‑off across multiple candidates, design a cost/latency/quality decision matrix, and implement a cascade/router that balances SLOs with budget. The chapter finishes by plugging your chosen model(s) into the Chapter‑1 service with structured outputs and streaming.

**Estimated time:** 90–120 min read · 90–120 min lab  
**Prereqs:** Chapter 1 app, basic Python, ability to call at least two model APIs (or local models).

## Learning Objectives

By the end of this chapter, you can:

- Translate product requirements into measurable model selection criteria.
- Set up a bake‑off harness with golden tasks, safety probes, and time/cost logging.
- Interpret key metrics (TTFT, tokens/sec, schema adherence, refusal quality) and weigh them with a decision matrix.
- Design a multi‑model cascade (primary, fallback, specialists) with routing rules and SLO‑aware timeouts.
- Configure structured output/JSON mode and function calling with validation and retries.

## 2.1 Requirements → Criteria

Map needs to measurable knobs before testing.

- **Quality:** target task success (e.g., ≥85% on golden set), groundedness for RAG tasks, schema adherence ≥98%.
- **Latency:** p95 ≤ X sec end‑to‑end; TTFT ≤ Y sec for streaming UX.
- **Cost:** max $/1k requests; price ceilings per input/output token.
- **Safety:** jailbreak resistance, refusal correctness, toxicity/PII leakage below thresholds.
- **Capabilities:** function calling; long context; multilingual; tool awareness; math/code proficiency.
- **Compliance:** data residency, on‑prem vs. SaaS, logging constraints.

**Tip:** Freeze these as acceptance criteria so the bake‑off has a clear pass/fail bar.

## 2.2 Model Landscape & Architectural Options

- **General‑purpose LLMs:** strong instruction‑following; good default primary.
- **Specialists:** code/maths models, multilingual, summarization‑optimized.
- **Smaller/edge models:** lower latency/cost; good for high‑QPS, light tasks.
- **MoE / long‑context variants:** throughput gains, larger windows; check retrieval fidelity.
- **Hosted APIs vs. self‑hosted:** speed of iteration vs. control/compliance.

**Architecture patterns:**

- **Single‑model:** simplest; rely on prompt rigor and caching.
- **Cascade:** cheap fast model first; escalate to stronger model on triggers.
- **Router:** rule‑ or classifier‑based dispatch to specialized models.
- **Ensemble:** self‑consistency / N‑best with vote (higher cost, higher quality).

## 2.3 Metrics & Instrumentation

**Latency:**

- TTFT (time‑to‑first‑token) for perceived responsiveness.
- TBT (time‑between‑tokens) or tokens/sec for streaming speed.
- p50/p95 per request and per sub‑step (RAG, model, tools).

**Cost:**

- Input/output token prices × usage; embedding costs; caching hit‑rate adjustments.

**Quality:**

- Task success (exact match/F1/rubric scores per task type).
- Schema adherence % (strict JSON validation error rate).
- Groundedness (citation coverage, contradiction rate) for RAG.
- Refusal quality (correctly refuses unsafe/underspecified queries).

**Reliability:**

- Error rate, timeout rate, retry/fallback rate.

Instrument your harness to emit a JSONL per run: one line per case with all metrics + raw traces (sanitized).

## 2.4 Bake‑Off Harness

**Dataset composition (≈80–150 items):**

- 30% core tasks (e.g., domain Q&A, transformation with schema).
- 20% edge cases (long inputs, multilingual names, numerics).
- 20% RAG‑style with short supplied snippets; check grounding.
- 20% safety probes (prompt injection, PII bait, disallowed requests).
- 10% performance micro‑bench (very short + very long prompts/outputs).

**Protocol:**

- Prompts: Use the same system prompt across models; enable JSON/structured output where available.
- Trials: Two modes per model: zero‑shot and few‑shot (2–4 exemplars) to gauge sensitivity.
- Warmup: discard first N runs to avoid cold‑start bias; cache embeddings if used.
- Runs: Execute all cases per model; collect metrics, costs, and traces.
- Analysis: compute per‑bucket scores; run paired deltas and significance (e.g., bootstrap CI on accuracy).

> **Pro Tip: Model-Specific Shims**
>
> While using an identical prompt is a fair baseline, you may find some models underperform due to sensitivity to phrasing. For a more advanced bake-off, consider creating minor "shims" or "adapter prompts" that wrap the core instruction in the model's preferred format (e.g., using XML tags for one model vs. Markdown for another) while keeping the substance of the request identical.

**Harness sketch:**

```
inputs/                # prompts & cases
  goldens.jsonl       # {id, task_type, input, expected?, context?}
  rubric.yaml         # grading rules per task_type
harness/
  run.py              # loops models × cases; logs metrics
  grade.py            # exact/F1/rubric; schema validator
  report.ipynb        # plots & decision matrix
output/
  runs/*.jsonl        # raw per‑run results
  summary/*.json      # aggregated metrics
```

## 2.5 Decision Matrix

Normalize metrics to 0–1 and weight by business priorities.

**Example weights:** Quality 0.45, Latency 0.20, Cost 0.15, Safety 0.15, Features 0.05.

These weights are not arbitrary; they must be a direct reflection of your product's primary goals from section 2.1. A real-time conversational agent might weight Latency at 0.40, while an offline document analysis tool would weight Quality much higher, perhaps at 0.60.

**Example row (illustrative):**

| Model | Q(0.45) | L(0.20) | C(0.15) | S(0.15) | F(0.05) | → | Score |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- | :---: |
| A | .86 | .72 | .63 | .90 | .80 | | 0.80 |
| B | .82 | .88 | .77 | .78 | .60 | | 0.80 |
| C | .90 | .55 | .40 | .92 | .90 | | 0.77 |

### The Qualitative Tie-Breaker: Human Review

Automated metrics are essential for scaling evaluation, but they don't tell the whole story. A model with a 92% score might produce robotic, unhelpful text, while one with an 89% score is more creative and aligned with your brand's tone.

**Protocol:** After automated grading, manually review a sample of 20-30 outputs, paying special attention to:

- **Failures:** Why did it fail? Was it a subtle misunderstanding or a complete refusal?
- **Safety Probes:** Was the refusal graceful and helpful, or blunt and uninformative?
- **"Good Enough" Answers:** Compare passing answers across models. Which one is more useful, clear, or well-structured?

**Artifact:** Add a "Qualitative Score" or "Reviewer's Notes" column to your decision matrix. This is often the most important factor for breaking ties and making the final call.

Tie‑break with qualitative notes (ecosystem maturity, roadmap, support SLAs). Keep the spreadsheet under version control; decisions are artifacts.

## 2.6 Cascades & Routing

**Triggers to escalate to a stronger model:**

- Schema validation failure after one repair attempt.
- Low confidence/self‑check (model returns a calibrated score or fails a verifier prompt).
- Safety route: sensitive topics → safety‑tuned model.
- Complexity heuristics: long inputs, many constraints, math/code present.
- Timeouts: TTFT exceeds threshold → cut over.

**Router rules (pseudo):**

```
if safety_sensitive(q):
    return call(model="safe_guarded")
if is_complex(q) or contains_math_or_code(q):
    return call(model="advanced")
res = call(model="fast")
if not valid(res) or low_confidence(res):
    return call(model="advanced")
return res
```

**Operational knobs:**

- Per‑tenant routing (enterprise vs. free tier).
- Budget guard: cap daily spend; degrade gracefully to retrieval‑only.
- Shadow runs: send a % of traffic to candidate model for offline comparison.

## 2.7 Structured Output & Function Calling

- Prefer JSON mode / structured outputs with strict schema and enums.
- Repair loop: if invalid JSON, attempt one constrained repair; else escalate.
- Function calling: define tool schemas; validate arguments; idempotent design.
- Streaming JSON: buffer and emit only well‑formed objects; don’t leak partials to clients unless flagged as experimental.

**Schema snippet (task_result):**

```json
{
  "type": "object",
  "required": ["status", "data", "notes"],
  "properties": {
    "status": {"enum": ["ok", "refused", "uncertain"]},
    "data": {"type": "object"},
    "notes": {"type": "string"}
  }
}
```

## 2.8 Safety in Selection

- Run jailbreak suite; score refusal correctness and leakage.
- Evaluate PII redaction and policy adherence under pressure.
- Prefer models with tool‑use safety (argument filters) if you plan actions.
- Consider data handling: logging, retention, geo, encryption, isolation.

## 2.9 Deployment Considerations

- Hosted API: fastest to start; watch quotas, rate limits, regionality.
- Self‑hosted: vLLM/TPU/GPU; enable dynamic batching; autoscale; observability.
- Caching: prompt hash → output cache; TTL; invalidation on prompt/model change.
- Versioning: immutable model IDs; blue/green prompt rollout; rollback hooks.

## Hands‑On Lab: “Bake‑Off & Cascade”

> **Pro Tip: Lab Scope & Scaffolding**
>
> This is a substantial lab. To keep it focused on the core learning objectives—analysis and implementation—the book's public repository provides significant scaffolding. You will find a pre-populated `goldens.jsonl` file and nearly complete harness scripts. Your task will be to wire in your model API keys, run the evaluation, and then use the results to implement the routing logic, not to write boilerplate code from scratch.

**Goal:** Evaluate three models, choose a primary + fallback, and implement a router in the Chapter‑1 service.

### Step 1 — Prepare Data

Create `inputs/goldens.jsonl` with ≥100 cases across the buckets in §2.4. Add `rubric.yaml` with scoring rules; include schema validation checks.

### Step 2 — Implement Harness

`harness/run.py` executes (model × mode × case), measures TTFT, tokens/sec, total tokens, cost, and captures traces. `harness/grade.py` computes per‑bucket metrics + safety scores. Emit `output/runs/*.jsonl` and an aggregated `summary.json` per model.

### Step 3 — Analyze & Decide

Load summaries into a notebook; compute normalized scores and the decision matrix. Document trade‑offs and the chosen routing policy.

### Step 4 — Wire the Cascade

In the Chapter‑1 service, add `router.py` implementing rules from §2.6. Add timeouts and escalation paths; log `router_decision` in traces. Enforce JSON schema with one repair attempt before fallback.

### Step 5 — Regression & SLOs

Add CI job that runs a smaller golden subset on PRs; block if score drops > threshold. Add dashboards for p50/p95 TTFT, tokens/sec, cost/request, fallback rate.

## Acceptance Criteria

- Bake‑off report with metrics and chosen weights committed to repo.
- Router enforces schema adherence and safety routes; fallback < 15% under nominal load.
- p95 TTFT meets target on primary; budget within ceiling at P50 traffic model.
- CI gate on goldens + alerting for latency/cost regressions.

## Stretch Goals

- Confidence verifier step before escalation (self‑critique or small verifier model).
- Cost‑aware routing by tenant tier; add shadow tests for a 4th candidate.
- Offline distillation: fine‑tune a small model on high‑quality outputs from the strong model.

## Design Reviews & Anti‑Patterns

- “Pick the biggest model.” Costs explode; measure and route instead.
- “Latency means fastest model only.” UX is TTFT + streaming; optimize pipeline, not just model.
- “Ignore safety in bake‑off.” You’ll ship regressions; include refusal/PII probes.
- “One prompt fits all models.” Minor differences matter; keep a core prompt + per‑model shims.
- “Deploy without rollback.” Version and canary prompts/models.

## Checklists

### Selection Readiness

- [ ] Criteria & thresholds defined and documented
- [ ] Diverse golden set incl. safety & RAG probes
- [ ] Harness logs TTFT/tokens/sec/cost with trace IDs
- [ ] Normalization + weighting method agreed

### Deployment Readiness

- [ ] Router with clear triggers & timeouts
- [ ] JSON mode + schema validation + repair
- [ ] Fallback models configured & tested
- [ ] Dashboards & alerts in place

## Quick Quiz

- Which metric most strongly correlates with perceived responsiveness in chat UIs?
- Name three reliable triggers to escalate in a cascade.
- How would you normalize heterogeneous metrics for a decision matrix?
- Why include safety probes in a model bake‑off?

## What’s Next

**Chapter 3 — Prompt Engineering Fundamentals:** Build a versioned prompt package with A/B flags, schema‑first design, and a rollback strategy; then re‑evaluate your chosen models under the new prompt suite.
