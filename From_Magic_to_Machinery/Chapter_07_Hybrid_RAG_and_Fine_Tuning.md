# Chapter 7: Hybrid RAG + Fine-Tuning

**Synopsis:** RAG gives you up-to-date facts; fine-tuning shapes how the model behaves. This chapter shows how to combine them: RAG for knowledge, a light behavioral tune (e.g., LoRA/PEFT) for tone, schema fidelity, and domain phrasing. You’ll evaluate when hybrid beats RAG-only or tune-only, model cost/latency, add caches, and ship a measured rollout.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Chapters 1–6; familiarity with PEFT/LoRA helpful.

## Learning Objectives

- Decide when to fine-tune vs. rely on prompts and RAG.
- Choose a hybrid architecture pattern that fits your SLOs and data realities.
- Run a small PEFT tune for behavior/style while keeping facts in RAG.
- Model latency/cost and add caches (prompt, retrieval, answer shards).
- Evaluate with A/B/C baselines (RAG-only vs. tune-only vs. hybrid).

## 7.1 When to Combine (and When Not To)

**Use Hybrid when…**

- You need consistent format/tone (e.g., legal summaries, support replies) beyond what prompts deliver.
- You must enforce schema adherence at very high rates (≥99%) without heavy prompt verbosity.
- Domain jargon or phrasings should be standardized (brand style, ICD-10/finance codes).
- RAG already controls factuality; the gap is behavior, not knowledge.

**Avoid/Delay Fine-Tuning when…**

- The issue is missing/poor data (fix pipeline + retrieval first).
- Requirements are volatile—prompts/flags iterate faster and cheaper.
- You’re trying to bake in changing facts—keep those in RAG.

> **Pro Tip: The Smart Intern Analogy 🎓**
>
> Think of your RAG and fine-tuning systems as training a new, very smart intern.
>
> RAG is like giving the intern a well-organized, perpetually up-to-date binder of company documents. This makes them knowledgeable. They can look up any fact you need.
>
> Fine-tuning is like training the intern on your company's specific communication style, formatting templates, and professional tone. This makes them well-behaved. They answer questions in the right way.
>
> You need both. You wouldn't ask an intern to memorize the entire binder (that's what RAG is for), and you wouldn't expect them to know your company's email etiquette without training (that's what fine-tuning is for).

## 7.2 Architecture Patterns

- **RAG → Tuned Generator (most common):** Retrieve context → generate with a behavior-tuned model that expects structured inputs and emits structured JSON.
- **Tuned Reranker + RAG → Base Generator:** Train a tiny reranker (cross-encoder) for ordering; keep base generator. Good when generation is fine but retrieval ordering is weak.
- **Dual Tune (rare):** Fine-tune both reranker and generator. Use sparingly; costs compound.
- **Distilled Small Model + RAG:** Distill outputs from a strong model into a smaller, tuned model to reduce serving cost while keeping RAG for facts.

Start with Pattern 1 unless a retrieval weakness is clearly dominant.

> **Author's Note:** A set of simple diagrams would make these patterns much easier to compare. We should include a small flowchart for each of the four architectures described.

## 7.3 Data for Behavioral Fine-Tuning

**Sources**

- High-quality human-edited answers over your RAG context (keep the context with examples).
- “Gold” outputs from a stronger model post-critique (then reviewed).

**Example format (JSONL):**

```json
{"prompt":{"system":"...schema/policy...",
           "context":[{"id":"S1","text":"..."},{"id":"S2","text":"..."}],
           "user":"Summarize the policy changes..."},
 "output":{"status":"ok","answer":"...","citations":["S1","S2"]}}
```

**Curation rules**

- Balance intents and difficulty; include negative cases (insufficient context → refused).
- Strip PII/secrets; keep citation IDs accurate.
- Hold out a temporal slice to test generalization.

> **Pro Tip: Budget for Data Curation—It's a Project in Itself**
>
> Creating a high-quality dataset of 1,000–2,000 examples is the most critical—and often most underestimated—part of fine-tuning. This is not just a data export. It requires significant effort from subject matter experts (SMEs) to write, review, and edit examples to ensure they perfectly reflect the desired behavior. Budget for this as a sub-project with its own timeline and resource allocation. A great model cannot fix a mediocre dataset.

## 7.4 Training Strategy (PEFT/LoRA)

Objective: behavioral compliance (tone, format, refusal style), not memorizing facts.

**Base model:** same family as prod for smooth drop-in.

**LoRA config:** small rank (e.g., r=8–16), target attention/projection layers.

**Loss shaping:** weight schema fields; penalize hallucinated citations.

**Epochs/early stop:** prefer under-fit to avoid brittleness; watch evals.

**Safety**

- Include refusal examples; ensure tune doesn’t weaken guardrails.
- Validate with a jailbreak/safety set post-train.

A key risk in fine-tuning is catastrophic forgetting, where the model becomes highly specialized in your task but loses some of its general reasoning or safety capabilities. To mitigate this, consider including a small percentage (5-10%) of general-purpose and safety-oriented examples in your training data, even if they aren't specific to your core task. This helps the model retain its foundational abilities.

## 7.5 Serving the Hybrid

**Request path**

- Query normalization → Hybrid retrieve (BM25+vector + optional rerank).
- Context assembly (MMR, trimming, k≤6).
- Generator = tuned model in JSON/structured mode.
- Validator/repair → Refusal check → Emit with citations.

**Compatibility**

- Keep system/dev messages identical across tuned and base models so you can fall back instantly.
- Version and store the adapter weights; use immutable IDs.

## 7.6 Latency & Cost Modeling

Let:

- `L_r` = retrieval + rerank latency
- `L_g` = generation latency (TTFT + stream)
- `C_r` = retrieval cost (infra + rerank)
- `C_g` = tokens × price (input+output)

Hybrid total ≈ (`L_r`+`L_g`, `C_r`+`C_g`)

**Trade-offs**

- LoRA adds no runtime cost beyond using that model; gains show up as fewer repairs, shorter prompts, or shorter outputs.
- If tune allows leaner prompts (–15–30% tokens) or fewer self-consistency passes, you may break even or win on cost.

## 7.7 Caches That Matter

- Retrieval cache: `(query, filters) → [doc_ids]` with short TTL.
- Context hash → answer draft cache: safe only for deterministic tasks.
- Adapter pinning: keep tuned weights warm in memory; pre-load on deploy.

## 7.8 Evaluation Protocol (A/B/C)

Compare three arms on the same traffic or replay:

- **A:** RAG-only (base model)
- **B:** Tune-only (no RAG; same prompts)
- **C:** Hybrid (RAG + tuned model)

**Metrics:** Quality, Reliability, Perf/Cost, Safety.

Declare win if Hybrid (C) beats A on quality without violating p95/cost SLOs, and B underperforms on factuality (expected).

## 7.9 Rollout & Risk Controls

Canary 5–10% → 50% → 100%, with rollback switch to base model. Alert on: adherence drop, hallucination ↑, refusal ↓, p95 ↑, spend ↑.

## 7.10 Common Failure Modes

- Tune memorizes examples → answers without citing; fix with more refusal/uncertain cases, reduce epochs.
- Over-formatting: tune insists on style even when task changes; add diversity to training set and keep prompts authoritative.
- Citation drift: model cites wrong IDs; include citation accuracy loss.

## Hands-On Lab: “Behavioral LoRA over RAG”

**Goal:** Train a small LoRA adapter to enforce tone, schema, and citation style while RAG supplies facts; compare A/B/C.

### Step 1 — Dataset

Collect 1–2k examples with `{prompt, output}`.

### Step 2 — Train

Use PEFT/LoRA with a small rank for 1-3 epochs.

### Step 3 — Serve

Load adapter with a feature flag to switch `base|lora`.

### Step 4 — Evaluate (A/B/C)

Replay queries and record metrics.

### Step 5 — Report & Decision

Produce a table with deltas and a go/no-go recommendation.

## Acceptance Criteria

- Hybrid improves task success by ≥ +5 points over RAG-only
- Groundedness ≥ 90%, hallucinations ≤ 5%
- Schema adherence ≥ 99%, repair rate ≤ 3%
- p95 latency within SLO (no worse than +10%)

## Stretch Goals

- Distill Hybrid → a smaller model for low-cost serving.
- Train a tuned reranker and compare Hybrid+Rerank vs. Hybrid.

## Design Reviews & Anti‑Patterns

Tuning for facts: keep facts in RAG; tune behavior.  
Ignoring refusals in training: leads to over-confident outputs.  
No rollback: always keep base path hot and versioned.

## Checklists

### Training Readiness

- [ ] Curated, PII-safe dataset with context + outputs
- [ ] Balanced refusals/uncertain cases
- [ ] Holdout evals (incl. temporal)

### Deployment Readiness

- [ ] Adapter versioned; flag to switch base↔tuned
- [ ] Caches configured; index freshness gates
- [ ] Canary + rollback plan

## Quick Quiz

- Why is behavioral fine-tuning usually preferable to knowledge fine-tuning in dynamic domains?
- Name two metrics that prove hybrid beats RAG-only without busting SLOs.
- How do you prevent a tune from reducing refusal correctness?
- What cache keys make an answer cache safe in a RAG system?

## What’s Next

**Chapter 8 — Fine-Tuning Techniques:** We’ll go hands-on with PEFT/LoRA, dataset curation, eval-on-train vs. holdout, and drift management. You’ll ship a model card + test suite, and learn safe update practices for tuned models.
