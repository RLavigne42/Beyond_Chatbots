# Chapter 15: Case Study — Enterprise Knowledge Assistant

**Synopsis:** We’ll turn ACME Assistant into a production-grade enterprise knowledge assistant that answers employee questions with citations, files helpdesk tickets when needed, and stays safe/compliant. You’ll build ingestion and RAG, wire hybrid retrieval + reranking, enforce citation-first answers, and integrate a helpdesk workflow with user confirmations and audit.

**Estimated time:** 90–120 min read · 90–120 min lab  
**Prereqs:** Parts I–III; especially Ch.5–6 (data & RAG), Ch.11 (systems integration), Ch.14 (safety).

## Learning Objectives

By the end of this chapter, you can:

- Design a docs→chunks→embeddings→indexes pipeline with lineage and access controls.
- Implement hybrid retrieval (BM25 + vector) with ML reranking and context assembly.
- Enforce citation coverage and groundedness checks in prompts and post-processing.
- Connect to a helpdesk API, propose actionable steps, and require user confirmation before creating tickets.
- Instrument cost/latency/quality dashboards and run a targeted eval suite for knowledge QA.

## 15.1 Problem & Scope

> **Design Review: Solving the "Cold Start" Problem**
>
> This case study assumes a rich set of documents is available for ingestion. In the real world, enterprise knowledge bases can be sparse or outdated. This is the "cold start" problem. A pragmatic strategy is to not boil the ocean. Instead:
>
> - Analyze the last 3 months of helpdesk tickets.
> - Identify the top 20-50 most frequently asked questions.
> - Work with content owners to ensure that high-quality, up-to-date documentation for just those topics exists before you launch.
>
> This ensures your assistant is immediately useful for the most common problems, buying you goodwill and time to progressively expand its knowledge base.

ACME employees ask: “How do I reset my VPN?”, “What’s the parental leave policy?”, “When does Q4 blackout start?” Our assistant must:

- answer only from approved sources with citations;
- recognize when the answer isn’t in docs and offer to file a helpdesk ticket;
- respect tenant/role access;
- keep PII out of logs and comply with retention rules.

## 15.2 Architecture (High Level)

A flowchart showing the end-to-end process from source documents through ingestion, RAG, and the helpdesk tool workflow.

## 15.3 Data Ingestion & Quality

Using the resilient pipeline architecture we designed in Chapter 5, we will now configure connectors for Confluence and PDFs.

- **Sources:** ACME Confluence/Notion spaces, HR PDFs, IT runbooks, prior solved tickets.
- **Cleaning:** remove boilerplate nav; fix headings; normalize Unicode; OCR for scanned PDFs.
- **Chunking:** structure-aware blocks ~300–800 tokens with overlap 10–15%.
- **Metadata (required):** `source_id`, `doc_acl`, `rev_timestamp`, `hash`.
- **Indexes:** BM25 for exact terms; Vector for semantics; Reranker to score relevance.
- **Freshness:** incremental jobs; purge on doc deletion.

## 15.4 Retrieval Orchestration

- Query normalization: lowercasing, acronym expansion.
- Hybrid retrieve: BM25@50 + Vector@50 → union → rerank to top 12–16.
- Context assembly: max tokens cap; group by source; include provenance.
- Safety filters: enforce ACLs; strip PII patterns.

## 15.5 Prompting for Grounded QA (Schema-First)

**System (excerpt):** “Answer only from supplied context. If answer not derivable, set `status:"insufficient"`. Cite sources with `supporting_citations[]`.”

**JSON Schema (knowledge_answer.v1):** requires `status`, `answer`, `supporting_citations`, `confidence`.

Citation coverage check: after model returns, verify claims in answer overlap with a cited span. If coverage is low, override status to `"insufficient"`.

> **Author's Note:** While this is a backend-focused book, a simple UI wireframe would be highly effective here. A visual mock-up showing how the citations are displayed in the answer and what the "Confirm Ticket Creation" modal looks like would make these crucial user-facing safety features much more tangible.

## 15.6 Helpdesk Workflow (Tool Use with Confirmation)

When `status:"insufficient"` or question is operational, assistant proposes creating a ticket.

Tool contract (`helpdesk.create_ticket`): Defines arguments like summary, priority, category.

Propose → Preview → Confirm → Commit: The model returns a proposal. The UI shows a preview. The user confirms, and only then does the tool execute.

This workflow is a direct implementation of the "Propose → Preview → Confirm → Commit" safety pattern we established for all side-effecting tools in Chapter 11.

> **Pro Tip: Close the Loop on Knowledge Gaps 🔁**
>
> When the assistant determines the context is insufficient (`status:"insufficient"`), filing a helpdesk ticket is a great fallback. A world-class system takes it one step further: it also logs the user's original question as a "knowledge gap."
>
> This creates a powerful feedback loop. You can send a weekly digest of these uncaught questions to the content owners (e.g., the HR or IT teams). This tells them exactly what new documentation they need to write to make the knowledge base—and therefore the assistant—more effective over time.

## 15.7 Metrics & Dashboards

- **Retrieval:** recall@k, reranker nDCG.
- **Answer quality:** groundedness score, citation coverage %, refusal correctness.
- **Ops:** p95 TTFT, p95 end-to-end latency, cost/request, fallback rate.
- **Workflow:** ticket proposal rate, confirm rate.

## Hands-On Lab: “ACME Knowledge Assistant”

**Goal:** Stand up ingestion + hybrid RAG + citation-first QA with a helpdesk ticket workflow and safety controls.

### Step 1 — Ingest & Index

Implement the data pipeline for mock HR/IT docs.

### Step 2 — Retrieval Orchestrator

Create the hybrid retrieval function with ABAC filtering.

### Step 3 — Prompt & Schema

Wire the `knowledge_qa` prompt package.

### Step 4 — Output Enforcement

Implement the `citation_check.py` post-processor.

### Step 5 — Helpdesk Integration

Add the `helpdesk.create_ticket` tool with confirmations.

### Step 6 — Evals & Dashboards

Create a golden set and safety probes for the knowledge use case.

## Acceptance Criteria

- ≥ 85% task success on goldens; citation coverage ≥ 0.8
- Zero cross-tenant leaks on safety set; correct refusals ≥ 99%
- p95 E2E ≤ 2.5s on warmed cache; cost/query within budget
- Ticket proposals only when `status:"insufficient"` or action-type request.

## Design Reviews & Anti‑Patterns

“Just vector search.” Always add BM25 for acronyms/IDs and a reranker for precision.  
“Answer then find a citation.” Reverse it: retrieve → answer from context → cite.  
“Trust the model on access.” Enforce ABAC before prompting.

## Checklists

### Data & Retrieval

- [ ] Sources deduped; lineage & ACLs attached
- [ ] Hybrid retrieval + reranking measured; k’s tuned

### Grounded QA

- [ ] Schema-first JSON; citation fields required
- [ ] Coverage checker enforces grounding

### Workflow & Safety

- [ ] Helpdesk tool with arg schema; confirmations & idempotency
- [ ] ABAC enforced pre-prompt; PII redaction in outputs/logs

## Quick Quiz

- Why is hybrid retrieval (BM25 + vector) typically better than either alone?
- What is citation coverage, and how do you enforce it?
- When should the assistant propose a helpdesk ticket vs. answering directly?

## What’s Next

**Chapter 16 — Case Study: Analytics Copilot.**
