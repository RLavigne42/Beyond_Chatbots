# Chapter 13: LLMOps — Deployment & Monitoring

**Synopsis:** You’ve got quality signals (Ch. 12). Now you need to ship—safely, fast, and cheaply. This chapter turns ACME Assistant into a production service with versioned rollouts (blue/green prompts, model canaries), autoscaling & batching, caching, speculative decoding & streaming, SLOs and error budgets, full-stack observability, and incident response. You’ll also stand up cost governance so improvements don’t bankrupt you.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Ch. 1–12; basic cloud/container familiarity; metrics/alerts fluency.

## Learning Objectives

- Choose a serving topology (hosted API vs. self-hosted) and enable autoscaling + batching.
- Implement multi-layer caching and admission control to hit latency targets.
- Ship changes with blue/green prompts, model canaries, and automatic rollback.
- Define and enforce SLOs (latency, availability, quality proxies, cost) with error budgets.
- Build dashboards, alerts, traces, and an on-call playbook.

## 13.1 Serving Topologies

**Hosted API (vendor)**

- **Pros:** minimal ops, rapid iteration, global SLA.
- **Cons:** hard rate limits, opaque internals, data/geo constraints.

**Self-hosted (GPU/TPU + serving runtime)**

- **Pros:** control, custom batching/kv-cache, cost leverage.
- **Cons:** ops heavy; you own scaling, upgrades, failures.

**Hybrid**

- Hosted for spikes/sensitive models; self-host primary or specialist models.

## 13.2 Throughput & Latency Fundamentals

- **TTFT (time-to-first-token):** most visible to users.
- **Tokens/sec:** steady-state streaming speed.
- **p95/p99 end-to-end latency:** includes RAG, memory, tools.

**Levers**

- Batching (continuous batching): combine similar requests for massive throughput.
- KV cache reuse: reuse prompt kvs across turns.
- Speculative decoding: draft with a small model, verify with the large one.
- Parallelization: overlap RAG fetch + prompt render + model warmup.

> **Pro Tip: What is the KV Cache? 🤔**
>
> The KV cache is one of the most important optimizations for conversational AI. Here’s the simple version:
>
> When a model processes a prompt, it performs a huge number of calculations to understand the relationships between the tokens. The results of these calculations are called the Key/Value (KV) states.
>
> The KV cache saves these states in memory. When you send the next message in a conversation, the model doesn't need to re-calculate the entire history. It just loads the saved states from the KV cache and computes the new tokens. This dramatically speeds up generation and reduces the number of tokens you need to re-process, saving both time and money.

## 13.3 Caching Layers (and Keys)

**Output cache (deterministic tasks)**

- Key: `model_id|prompt_hash|schema_id|input_hash|ctx_hash|retrieval_index_version`

**Embedding cache**

- Key: `embedding_model|text_hash`

**Retrieval cache**

- Cache top-k doc IDs per query hash.

## 13.4 Admission Control, Backpressure, and Rate Limits

- Token-budget admission: reject or degrade requests expected to be too long.
- Per-tenant leaky bucket rate limiting.
- Queue caps to bound concurrent generations.
- Degradation modes: turn off RAG, shrink n-best, or switch to fast model.

## 13.5 Streaming & Speculative Decoding

- Streaming UX: emit a stable header early (title/status), then content.
- Speculative: small drafter proposes tokens; verifier accepts/rejects.

## 13.6 Release Management: Blue/Green & Canaries

Version everything: `prompt_id@semver`, `model_id@commit`, `retrieval_index@timestamp`.

**Blue/Green prompts**

- Deploy green to 10–20% traffic; compare against blue; promote on pass.

**Model canaries**

- Route 1–5% to candidate model; collect shadow traces.

**Auto-rollback triggers:** p95 latency > threshold, safety leakage ↑, schema adherence ↓.

> **Author's Note:** A simple diagram would make these two release strategies instantly clear. We should include a figure showing a traffic router for both a blue/green and a canary deployment. The Blue/Green diagram would show a router splitting traffic between two large, distinct pools (e.g., 85% to Blue, 15% to Green). The Canary diagram would show a main traffic pool with a router peeling off a very small percentage (e.g., 3%) to a new, smaller canary instance.

## 13.7 SLOs & Error Budgets

Define user-observable SLOs, not internal targets.

**Examples**

- **Availability:** Successful responses / total ≥ 99.9%.
- **Latency:** p95 TTFT ≤ 1.5s, p95 end-to-end ≤ 5s.
- **Quality proxy:** Schema adherence ≥ 98%.

**Error budget policy**

- If burned > 25% mid-window → freeze non-critical deploys.
- If burned > 50% → mandatory rollback or degradation mode.

> **Pro Tip: Your Error Budget is a Financial Budget for Risk**
>
> Service Level Objectives (SLOs) and error budgets can seem abstract. Think of them like this:
>
> Your SLO (e.g., 99.9% availability) is a promise you make to your users.
>
> Your Error Budget is the 0.1% of the time you are allowed to break that promise over a given period (e.g., a month).
>
> You "spend" this budget on taking risks: deploying new features, planned maintenance, or absorbing unexpected incidents. If your budget runs low, you do the same thing you would with money: you stop spending. All non-critical releases are frozen, and the team's focus shifts to reliability work until the budget is healthy again.

## 13.8 Observability: Traces, Metrics, Logs

- **Traces:** spans for retrieval, prompt render, model call, tool calls, validators, adapters.
- **Metrics (per layer):** RAG recall, model TTFT, tool success %, router fallback rate, safety refusal %.
- **Logs:** structured JSON; no PII.

A best practice is to aggregate the most critical SLOs from each layer—quality (adherence, groundedness), latency (p95 TTFT), cost ($/request), and safety (refusal correctness)—onto a single, high-level dashboard. This "single pane of glass" is the first place an on-call engineer looks during an incident to quickly assess the overall health of the system before diving into more detailed traces and logs.

## 13.9 Incident Response & Playbooks

Runbook content: Detection, Triage checklist, Rollback procedure, Comms templates.

War-room hygiene: One commander, one scribe. Freeze deploys until stable.

## 13.10 Cost Governance (FinOps)

- Tag everything: tenant, feature, model, prompt version, region.
- Budgets & alerts: daily/weekly caps; “slow-drain” alerts.
- Routing by cost: free tier → fast model; enterprise → higher ceiling.

## Hands-On Lab: “Ship ACME Assistant (Blue/Green + Canary + SLOs)”

**Goal:** Deploy ACME Assistant with blue/green prompt rollout, a model canary, autoscaling, caching, and full observability + alerts.

### Step 1 — Serving & Scaling

Choose topology and enable autoscaling.

### Step 2 — Caching

Implement output and retrieval caches.

### Step 3 — Blue/Green + Canary

Configure a router for gradual rollouts.

### Step 4 — SLOs, Dashboards, Alerts

Create SLOs, dashboards, and alerts.

### Step 5 — Cost Governance

Add per-tenant budget caps and a degradation switch.

## Acceptance Criteria

- Blue/green prompt rollout completes with no SLO breaches; rollback works.
- Canary traffic isolated; automatic rollback triggers on threshold breach.
- p95 TTFT ≤ 1.5s; end-to-end p95 ≤ 5s.
- SLO dashboards + paging alerts live; error budget policy documented.

## Design Reviews & Anti‑Patterns

Deploying prompts/models without pins. Immutable IDs only.  
Chasing p50 latency. Users feel p95; optimize tails.  
Ignoring cost until month-end. Real-time budgets + alerts.

## Checklists

### Release Readiness

- [ ] Prompt/model/index versions pinned; blue/green + canary plan
- [ ] Auto-rollback conditions defined & tested

### Observability & Safety

- [ ] Traces with per-stage spans; metrics for SLOs; PII-safe logs
- [ ] SLOs & error budgets documented; alerts wired

## Quick Quiz

- Which metric most affects perceived responsiveness, and which levers improve it?
- Name three conditions that should trigger automatic rollback during a canary.
- What belongs in a cache key to prevent stale responses after a prompt change?

## What’s Next

**Chapter 14 — Safety, Security & Governance**
