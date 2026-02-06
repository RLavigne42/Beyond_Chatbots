# Chapter 12: Evaluation & Evals

**Synopsis:** If you don’t test it, you don’t know it. This chapter turns “seems good” into measured quality. You’ll build an eval program that spans offline goldens, safety probes, cost/latency, and online A/B. We’ll design rubrics and metrics, wire a reproducible harness into CI with regression gates, and add dashboards so you can spot—and stop—bad changes to prompts, models, RAG, memory, or integrations.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Ch. 1–11; basic statistics; experience with CI.

## Learning Objectives

- Define an eval taxonomy (task, system, safety, cost/latency) and choose fit-for-purpose metrics.
- Create golden sets, adversarial probes, and a coverage matrix that evolves with your product.
- Implement a reproducible eval harness that logs traces and supports model/prompt/RAG variants.
- Add CI regression gates with significance checks; run canaries and shadow tests.
- Interpret eval deltas, triage regressions, and decide rollback vs. ship.

## 12.1 Why Evals (and Why Now)

LLM systems shift when anything changes: model weights, prompts, retrieved data, tools, latency budgets. Evals make changes observable, comparable, and reversible. The goal is not the perfect metric; it’s the stable contract that blocks harmful regressions and guides iteration.

## 12.2 Eval Taxonomy

| Layer | Purpose | Typical Metrics |
| --- | --- | --- |
| Task-level (offline) | Validate a prompt/model on a bounded task | Exact/F1, rubric score (0–5), schema adherence % |
| System-level (offline) | Exercise the full stack (RAG, memory, tools) | Task success %, groundedness, tool success % |
| Safety | Detect violations & jailbreaks | Refusal correctness, leakage rate, toxicity/PII flags |
| Performance/Cost | Keep UX usable and bills bounded | TTFT, p95 latency, cost/request, cache hit-rate |
| Online (A/B) | Measure real impact | CSAT, task completion, re-ask rate, incident rate |

Rule of thumb: offline for gating, online for confidence. Gate with goldens; confirm with A/B.

## 12.3 Datasets: Goldens, Probes, and Coverage

**Golden sets (deterministic):**

- 100–500 cases per domain, stored in `evals/goldens.jsonl` with fields: `{id, task_type, input, context?, expected, rubric_id?}`.
- Pin to specific prompt/schema versions and model IDs for reproducibility.

**Adversarial probes (safety):**

- Prompt injection strings, PII bait, policy edge cases, tool-argument exploits.
- Store expected outcome: `{status: refused|sanitized|allowed} + reason`.

**Coverage matrix:**

- Map tasks × entities × languages × formats. Track % coverage and focus on gaps.

**Synthetic augmentation:**

- Use model-assisted generation to expand cases, then human review 10–20% for drift.

> **Pro Tip: Build a Human-in-the-Loop (HITL) Curation Flow**
>
> Your evaluation datasets are living assets, not one-time exports. Building a simple human-in-the-loop (HITL) process is a production best practice. This doesn't need to be complex; it can start as a shared spreadsheet or a simple web form where subject matter experts (SMEs) can:
>
> - Review and grade a sample of recent production requests.
> - Correct flawed answers, creating new "golden" examples.
> - Add new, challenging test cases they encounter in their work.
>
> This feedback loop is the most reliable way to prevent your evaluation set from becoming stale and to ensure it reflects real-world usage patterns.

## 12.4 Rubrics & Metrics that Work

- Exact/F1 for extraction and classification.
- Schema adherence: strict JSON validation; count repairs.

**Groundedness (RAG):**

- Citation coverage (answer spans are supported by a cited snippet).
- Contradiction rate (answer contradicts sources).

**Refusal correctness:** correct refuse vs. over/under-refusal.

**Helpfulness/Rubric score (0–5) for open tasks—graded by:**

- Hybrid judging: LLM-as-judge with a fixed judge prompt + calibration set, plus periodic human spot checks.

**Stats & significance**

- Report mean/median & bootstrap 95% CI.
- Predefine MDE (minimum detectable effect) to avoid overreacting to noise.

> **Design Review: The Perils and Promise of LLM-as-Judge ⚖️**
>
> Using an LLM to grade another LLM's output is powerful for scaling evaluation, but it's fraught with risk if not handled carefully. Remember these principles:
>
> - **Use a Stronger, Separate Judge:** Never use the same model (or a weaker one) to judge outputs. Always use a best-in-class model (like GPT-4o or Claude 3.5 Sonnet) as your judge to ensure it has the reasoning capacity to evaluate nuanced responses.
> - **Beware of Bias:** LLM judges can be biased towards their own stylistic quirks, verbosity, or specific formatting. Calibrate your judge against a set of human-graded examples to ensure its ratings align with yours.
> - **Control for Position Bias:** When doing pairwise comparisons (A vs. B), always run the comparison twice with the order swapped (B vs. A) and check for agreement. Models often have a bias towards the first answer presented.
> - **Pin Everything:** The judge's model version and its system prompt are critical parts of your evaluation harness. Pin them just as you would your production model to ensure reproducible results.

## 12.5 Eval Harness Architecture

**Inputs:** model(s), prompt package IDs, retrieval config, memory flags, tools manifest, datasets.

**Runner:** executes each case, captures `trace_id`, raw messages, retrieved docs, tool calls, tokens, timings.

**Graders:** schema validator, exact/F1 calculators, rubric judges, groundedness checkers.

**Outputs:**

- `runs/{timestamp}/cases.jsonl` (one line per case)
- `runs/{timestamp}/summary.json` (aggregates per task + overall)
- `diffs/{baseline_vs_candidate}.json` (delta with CIs)

**Reproducibility**

- Pin model version, prompt_id/hash, schema_id, retrieval index version, and judge prompt_id.

## 12.6 CI Integration & Regression Gates

**Per-PR quick gate (1–3 min):**

- 30–50 “smoke” goldens + all safety probes.
- Gate on: schema adherence (≥98%), refusal correctness (≥99%).

**Nightly full eval:**

- All goldens & adversarial sets; compute quality delta vs. main.

**Gating policy (example)**

- Fail if schema adherence drops > 1 pp, or safety leakage increases > 0.2 pp, or p95 TTFT ↑ > 20%.

> **Author's Note:** A simple flowchart would effectively illustrate the CI/CD pipeline for evaluations. It would visually separate the fast, per-PR "smoke tests" from the comprehensive "nightly full evaluation" triggered on a merge to the main branch.

## 12.7 Online Evals: A/B and Shadow

- **A/B:** Sticky assignment by user/session. Primary KPIs: task completion, CSAT, incident rate.
- **Shadow:** Run candidate in parallel without user exposure; log answers & metrics; compare offline.

## 12.8 Safety Evals & Red Teaming

- Prompt injection suite, data exfiltration, toxicity/PII, tool abuse.
- Red team drills: quarterly adversarial sprints; convert findings → permanent probes.

## 12.9 Cost & Latency Evals

Track TTFT, tokens/sec, p50/p95 latency, token usage, cost/request for each run and by stage.

## 12.10 Triage & Root Cause

When a delta fails: slice by task, diff traces, classify failures, and assign blame to a system component.

## Hands-On Lab: “Build the Eval Harness & Gate the Pipeline”

**Goal:** Create an eval harness for ACME Assistant with goldens, safety probes, groundedness checks, and cost/latency metrics. Wire it into CI with regression gates and nightly full runs; add a dashboard.

### Step 1 — Repo Layout

Create a structured `/evals` directory.

### Step 2 — Metrics & Graders

Implement graders for schema, citations, and refusals.

### Step 3 — Harness Runner

Build the script to execute cases and capture traces.

### Step 4 — CI Gates

Add a fast PR gate and a nightly full run.

### Step 5 — Nightly & Dashboards

Publish reports and update dashboards.

## Acceptance Criteria

- Reproducible runs (pinned model/prompt/index/judge)
- Smoke suite completes ≤ 3 min in CI; full suite produces diffs & CIs
- Regression gate blocks on schema/safety drops; emits actionable diff

## Design Reviews & Anti‑Patterns

“Judge with the same model you’re testing.” Use a fixed, separate judge (or multiple).  
Overfitting to goldens. Rotate fresh cases; audit for leakage.  
Single metric worship. Track a basket (quality, safety, cost, latency).

## Checklists

### Dataset & Rubrics

- [ ] Goldens cover tasks/locales/formats; coverage gaps documented
- [ ] Safety probes reflect current policies & red-team findings

### Harness & CI

- [ ] Trace capture with prompt/hash, model ID, retrieval index version
- [ ] PR gate + nightly full eval configured

## Quick Quiz

- When would you prefer pairwise preference judging over rubric scoring?
- Name two metrics that specifically target RAG groundedness.
- What must be pinned to make eval runs reproducible?
- Give one gating rule you’d enforce on every PR.

## What’s Next

**Chapter 13 — LLMOps: Deployment & Monitoring**
