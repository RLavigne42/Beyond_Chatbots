# Chapter 4: Advanced Prompt Patterns

**Synopsis:** Move beyond single-shot instructions. This chapter turns proven prompting strategies into a reusable patterns catalog that consistently improves reliability, factuality, and control—without always resorting to bigger models. You’ll implement decomposition, ReAct (reason + act with tools), critique-and-refine, and self-consistency, wrapped in schema-first outputs, safety, and tests.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Chapters 1–3 (contract-first app, model bake-off, prompt package), JSON Schema, basic tool calling.

## Learning Objectives

By the end of this chapter, you can:

- Select and implement the right pattern (decomposition, ReAct, critique-and-refine, self-consistency) for a task.
- Keep outputs structured and safe across multi-step prompting.
- Measure trade-offs (quality vs. latency vs. tokens) and set SLOs per pattern.
- Package patterns as versioned templates with goldens, safety probes, and rollbacks.

## 4.1 Pattern Selection Guide

| Goal | Pattern | Why | Watch-outs |
| --- | --- | --- | --- |
| Tame complex asks | Decomposition | Smaller subproblems; explicit acceptance rules | More tokens, added latency |
| Use tools for facts/math | ReAct | Interleave reasoning and API/tool calls | Tool caps, idempotency, confirmations |
| Improve correctness/style | Critique-and-Refine | Targeted second pass catches violations | Patch must be schema-constrained |
| Reduce variance | Self-Consistency | N candidates → aggregate | Costly; cap N and early-exit |

Patterns change the process; your schema stays the contract.

> **Pro Tip: Rules of Thumb for Choosing a Pattern 🧐**
>
> - **Use Decomposition when...** the task feels like writing an essay or a report. If you would first create an outline with sub-sections, Decomposition is a good fit. It excels at multi-part questions ("Compare X and Y, then recommend Z").
> - **Use ReAct when...** the answer requires up-to-the-minute information, calculations, or knowledge from a specific database. If a human would need to use a search engine, calculator, or query a CRM to answer, ReAct is the right choice.
> - **Use Critique-and-Refine when...** you have strict, non-negotiable output constraints (e.g., legal compliance, brand voice, specific formatting). It's an excellent pattern for adding a final layer of polish and safety.
> - **Use Self-Consistency when...** the task is ambiguous and has no single correct answer, but you need a robust, plausible result (e.g., creative generation, brainstorming). It reduces the chance of an unusual or "off-the-wall" response.
> - **Combine patterns when...** a single step in a larger plan requires its own tool. For example, you might use Decomposition to plan a report, where Step 2 requires a ReAct loop to gather the latest financial data.

## 4.2 Decomposition (Plan → Solve → Synthesize)

**System (excerpt):**

> You will return ONLY a final JSON matching the given schema.  
> Internally: (1) PLAN numbered steps and evidence needs, (2) produce DRAFT outputs per step using ONLY provided tools/context,  
> (3) SYNTHESIZE final JSON. Do not include PLAN or DRAFT in the final answer.  
> Abort with `status:"uncertain"` if evidence is insufficient.

**Developer:** `subtasks[]` with acceptance rules, per-step token budgets, final schema.  
**User:** delimited task + any snippets.

**Tips:** cap steps (3–5); enforce acceptance rules → safe refusal on missing inputs.  
**Pros:** Higher adherence, clearer failure modes.  
**Cons:** Token/latency overhead.

## 4.3 ReAct (Reason + Act with Tools)

**Tool contract (example):**

```json
{
  "name": "kb.search",
  "description": "Retrieve top-k snippets from the knowledge base",
  "parameters": {
    "type": "object",
    "required": ["query", "k"],
    "properties": {
      "query": {"type": "string"},
      "k": {"type": "integer", "minimum": 1, "maximum": 5}
    }
  }
}
```

**Rules:** prefer tools for math/lookup; hide chain-of-thought; cap tool calls; on cap hit without evidence → `status:"uncertain"`.  
**Safety:** confirmations for side-effects; idempotency keys; strict arg validation + one repair attempt.

## 4.4 Critique-and-Refine (Two-Pass)

Pass A: draft final JSON. Pass B: critic returns constrained JSON patch with violations & fixes (schema, grounding, style, safety). Apply → re-validate; if unverifiable claims, downgrade to `status:"uncertain"`.

## 4.5 Self-Consistency (N-Best + Aggregation)

Sample N candidates, aggregate by majority vote, reranker, or field-wise merge. Cap N (3–5); early-exit on consensus.

## 4.6 Guardrail & Refusal Patterns

**Refusal structure:**

```json
{"status":"refused","data":{},"notes":"Neutral reason + allowed alternatives."}
```

**Guards:** topic filters, context sufficiency checks, PII redaction. Consistent, A/B-able behavior.

## 4.7 JSON-First, Streaming-Friendly

Emit early header (status/title/outline), fill data later; buffer until valid; single repair attempt on invalid JSON; keep examples compact.

## 4.8 Pattern Packaging (Repo Layout)

```
/prompts/patterns/
  decomposition/ (system.j2, developer.j2, user.j2, rubric.yaml, tests/)
  react/         (system.j2, tools.json, tests/)
  critique_refine/ (system.draft.j2, system.critic.j2, patch_apply.py, tests/)
  self_consistency/ (system.j2, aggregate.py)
  guardrail/     (system.j2, policies.yaml)
catalog.json   # {id, semver, schema_id, caps, notes}
```

## 4.9 Instrumentation & SLOs

Track: task success, contradictions, adherence %, repair count, TTFT/total/tool time, tokens, tool costs, refusal correctness, PII redactions. Set SLOs (e.g., adherence ≥98%, contradiction ≤1%); alert on drift.

## 4.10 Cost/Latency Controls

Caps (steps/tool calls/N), timeouts, early-exit; cache retrieval & subresults; compress instructions/examples.

## 4.11 Debugging Multi-Step Prompts

When a simple prompt fails, the error is obvious. When a ReAct loop gets stuck or a Decomposition fails on step 3, debugging is harder. The key is maximum observability.

- **Log Every Step:** Your trace logs should not just show the final answer but the entire thought process: the plan, each tool call and its observation, the draft, the critic's patch, etc. This is essential for understanding where the logic went wrong.
- **Isolate the Failure:** In a ReAct loop, is the model choosing the wrong tool, providing bad arguments, or misinterpreting the tool's output? In a Decomposition, does each sub-step's output meet its acceptance criteria? Pinpoint which step in the chain is breaking.
- **Check for Context Drift:** In long chains, the model can sometimes "forget" the original user intent. Ensure the core task is re-injected or referenced at each step to keep the process on track.

## Hands-On Lab: “From Brittle to Patterned”

**Goal:** Replace a brittle prompt with Decomposition, ReAct, and Critique-and-Refine, then quantify gains.

- **Baseline:** record adherence, success, contradictions, TTFT, tokens.
- **Decomposition:** add 3–5 step scaffold; measure deltas.
- **ReAct:** define `kb.search` + `calculator.*`; test numeric correctness & arg validation.
- **Critique:** draft → critic → patch → validate; cut schema errors/contradictions ≥50%.
- **Report:** per-run JSONL; propose routing rules by task type.

## Acceptance Criteria

- +8 pts task success or 50% error reduction
- ≥98% adherence; repairs ≤5%
- ≥99% numeric accuracy on math set
- ≥99% refusal correctness on safety probes
- Token overhead ≤+35% or justified

## Stretch

Add Self-Consistency (N=3); early-exit heuristic; classifier to auto-pick pattern.

## Design Reviews & Anti‑Patterns

Pattern soup; unbounded loops; leaky thoughts; overengineering; missing tests. (Mitigate with caps, canaries, goldens.)

## Quick Quiz

- Decomposition vs. self-consistency—when and why?
- Two safety controls for ReAct?
- What lives in a critic rubric?
- How do you cap multi-step cost/latency?

## What’s Next

**Chapter 5 — Data Pipelines & Quality:** Build the ingestion, cleaning, dedup, and chunking pipeline your RAG layer depends on—complete with metadata/lineage, quality checks, and update workflows.
