# Chapter 8: Fine-Tuning Techniques

**Synopsis:** When prompts and patterns aren’t enough, fine-tuning adjusts the behavior of a model—tone, format discipline, refusal style, and domain phrasing—while keeping facts fresh via RAG. This chapter walks through instruction tuning (SFT), parameter-efficient methods (LoRA/QLoRA/Adapters), preference optimization (RLHF/RLAIF-style, DPO/ORPO-style), evaluation, drift management, and safe rollout. You’ll ship a small PEFT tune with a model card + test suite and wire it into ACME Assistant.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Ch. 1–7; basic PyTorch/Trainer familiarity helpful.

## Learning Objectives

- Decide what to tune (behavior) vs. what to leave to RAG (facts).
- Curate high-signal datasets for SFT and preference training.
- Run PEFT/LoRA/QLoRA fine-tunes; know when to consider full-model updates.
- Apply preference optimization to reduce refusals/hallucinations and improve alignment.
- Build evals that track schema adherence, tone, refusal quality, and drift.
- Package and roll out a tune with versioning, safety gates, and rollback.

## 8.1 When Fine-Tuning Helps (and When It Doesn’t)

**Use fine-tuning for**

- Style & tone (brand voice, reading level, legal phrasing).
- Output discipline (JSON fields, citations format, refusal templates).
- Task generalization within a stable domain (e.g., support macros, doc plans).

**Don’t tune for**

- Changing facts (product specs, policies) → keep in RAG.
- One-off formats that prompts & validators already handle.
- Rapidly evolving requirements → iterate prompts/flags first.

Rule of thumb: RAG for knowledge; fine-tuning for behavior.

## 8.2 Data Curation & Formatting

**SFT dataset schema (JSONL):**

```json
{"input":{
  "system":"… policy + schema brief …",
  "context":[{"id":"S1","text":"..."},{"id":"S2","text":"..."}],
  "user":"Write a compliant summary..."
 },
 "output":{
  "status":"ok",
  "answer":"…",
  "citations":["S1","S2"]
 }
}
```

**Curation checklist**

- Dedup near-identical examples; remove boilerplate.
- Balance intents (summarize/extract/classify/refuse).
- PII scrubbed; preserve citation IDs if present.
- Train/val/test split with temporal holdout.

## 8.3 Supervised Fine-Tuning (SFT) Basics

**Goal:** learn mapping from (system, context[], user) → structured output.

**Good practices**

- Keep system message short; let examples teach style.
- Up-weight loss for key fields (e.g., status, citations) using field masks.
- Set max target length to avoid rambling; early stop on val loss + adherence.

## 8.4 Parameter-Efficient Fine-Tuning (PEFT)

**LoRA / QLoRA / Adapters**

- **LoRA:** learn low-rank matrices on attention/MLP projections; small disk, fast load.
- **QLoRA:** quantize base weights (e.g., 4-bit) + LoRA adapters → big memory savings.
- **Adapters:** small MLP blocks inserted between layers; similar trade-offs.

**When to use**

- Most production cases → PEFT first.
- Full-model tuning reserved for niche, high-scale scenarios with strong infra.

**Config cheatsheet (starting points)**

- LoRA r=8–16, α default, dropout 0.05–0.1; target q_proj,k_proj,v_proj,o_proj,gate,up,down.
- LR 1e-4–2e-4 (SFT), batch 64–256 tokens/step effective; cosine decay; warmup 3–5%.
- Train 1–3 epochs; watch val adherence and refusal correctness.

> **Pro Tip: The Turbocharger Analogy for PEFT 🏎️**
>
> Why is PEFT so effective? Think of the base model as a massive, pre-built engine.
>
> Full Fine-Tuning is like re-machining the entire engine block. It's powerful but incredibly expensive, complex, and results in a whole new, heavy engine.
>
> PEFT (LoRA/QLoRA) is like adding a small, lightweight, highly-efficient turbocharger. You get a huge performance boost for your specific task (acceleration) without altering the core engine. It's cheaper, faster to install, and you can easily swap it out or turn it off. For most applications, this is a far more efficient way to achieve the desired performance.

## 8.5 Preference Optimization (Alignment)

If Supervised Fine-Tuning (SFT) teaches the model imitation (i.e., "here is a good output, learn to produce it"), Preference Optimization teaches the model judgment (i.e., "given two possible outputs, learn why this one is better"). This is a crucial step for encoding nuanced qualities like tone, safety, or brevity that are hard to capture in a single "correct" example.

**Why:** SFT matches targets but doesn’t encode preferences (e.g., brevity, cautious tone, refusal boundaries).

**Options**

- Pairwise preference datasets: (prompt, A preferred over B) from humans or AI feedback (RLAIF).
- Learning strategies:
  - RLHF-style (reward model + policy optimization).
  - Direct Preference Optimization (DPO) / Odds-Ratio Preference Optimization (ORPO): simpler training on pairs without an explicit reward model.

**What to optimize**

- Groundedness preference: cite accurately > eloquent but unsupported.
- Refusal correctness: prefer safe refusal when context missing or policy triggers.
- Brevity/format: prefer concise, schema-tight outputs.

## 8.6 Evals for Tuned Models

- **Behavioral:** Schema adherence %, repair rate, tone & style rubric.
- **Grounding & factual:** Citation accuracy, hallucination rate.
- **Performance:** TTFT, tokens/output, cost/request.
- **Stability:** Drift across locales, long inputs, or different k (chunks).

## 8.7 Drift Management & Versioning

- Immutable IDs: `model: base@X.Y`, `adapter: acme-behav-lora@v1`.
- Model card: data sources, licenses, PII policy, evals, known limits, safety posture.
- Canary by tenant/surface; shadow on read-only flows.
- Roll back by unmounting adapter; keep base path hot.

> **Design Review: The Model Card is Your Nutrition Label**
>
> Treat your model card as more than just documentation; it's a critical governance and risk management artifact. Like a nutrition label on food, it tells consumers of your model everything they need to know: its ingredients (training data), its intended use, its limitations (known failure modes), and its safety profile. A well-written model card builds trust and ensures your tuned model is used responsibly.

## 8.8 Safety, Security, and Compliance

- Include refusal/uncertain patterns in SFT + preference data.
- Red-team the tune: jailbreaks, PII bait, tool-misuse prompts.
- Data governance: consent/source tracking; license compatibility; region isolation.

## 8.9 Cost & Latency Considerations

PEFT adds negligible runtime overhead; the win comes from shorter prompts and fewer repairs or retakes. Keep adapter loading warm to avoid cold TTFT spikes.

## 8.10 Deployment Playbook

Pre-flight → Staging → Canary → Ramp → Rollback

## Hands-On Lab: “PEFT Tune with Model Card & Tests”

> **Pro Tip: Accelerate Your Lab with Optimized Libraries**
>
> Training PEFT adapters, especially with QLoRA, can still be memory-intensive. To get the most performance out of consumer hardware (like a single GPU), consider using highly optimized libraries built on top of the core ML frameworks. Libraries like unsloth can dramatically speed up training and reduce memory usage by 2-5x with minimal code changes, making these powerful techniques more accessible.

**Goal:** Train a small LoRA adapter that improves tone + schema discipline, generate a model card, and wire into ACME Assistant with flags and evals.

### Step 1 — Data

Assemble 1–2k SFT examples. Optional: 300–600 preference pairs.

### Step 2 — Train SFT (LoRA/QLoRA)

Train for 1-3 epochs.

### Step 3 — (Optional) Preference Optimization

Train DPO/ORPO on pairs.

### Step 4 — Evals

Run a full suite of behavioral, grounding, and performance tests.

### Step 5 — Integration

Add a feature flag `GENERATOR=base|lora|lora_pref`.

### Step 6 — Model Card

Fill a template with metadata, evals, and known limits.

## Acceptance Criteria

- Schema adherence ≥ 99%; repair rate ≤ 3%
- Refusal correctness ≥ 99% on probes
- Citation accuracy ≥ 95% on grounded tasks
- Tokens/output −10–25% vs. baseline (same task success)

## Design Reviews & Anti‑Patterns

Behavior vs. facts confusion: don’t encode facts in SFT.  
Over-polished outputs: tune that ignores `status:"uncertain"` harms trust.  
No eval gates: never ship a tune without adherence/refusal/grounding gates.

## Checklists

### Dataset Readiness

- [ ] Balanced intents; refusals included
- [ ] Deduped; PII scrubbed; licenses OK
- [ ] Train/val/test with temporal holdout

### Deployment Readiness

- [ ] Model card filled & stored
- [ ] Flags, dashboards, alerts in place
- [ ] Canary + rollback plan tested

## Quick Quiz

- Name two behaviors that are ideal targets for fine-tuning and two that should remain in RAG.
- Why might DPO-style training be preferable to RLHF-style for small teams?
- Which metrics must improve before promoting a tune from canary to 50%?
- How do you detect and respond to drift in a tuned model?

## What’s Next

**Part III — Systems in the Wild**

Chapter 9: State & Memory (coming up)
