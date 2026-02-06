# Chapter 6: Vector Search & RAG Deep Dive

**Synopsis:** Retrieval-Augmented Generation (RAG) turns your model from “generally smart” into “specifically helpful.” In this chapter you’ll build a production-grade retrieval layer—embeddings, lexical + vector indexes, hybrid scoring, reranking, context assembly, and citation controls. You’ll quantify grounding and hallucinations, then wire the stack into ACME Assistant.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Chapters 1–5 (pipeline, prompt patterns), basic IR concepts.

## Learning Objectives

- Choose embeddings and indexes for your data and languages; understand cost/latency trade-offs.
- Implement hybrid retrieval (BM25 + vector) with optional reranking.
- Assemble answerable contexts with deduplication, diversity (MMR), and faithful citations.
- Measure RAG with recall@k, nDCG, groundedness, hallucination rate, and latency budgets.
- Operate RAG in production: freshness, ACL filtering, blue/green index swaps, eval gates.

## 6.1 What “Good Retrieval” Actually Means

RAG quality ≠ “the model wrote a nice paragraph.” It means:

- **Relevant:** the right passages are in the top-k (recall/precision).
- **Sufficient:** the retrieved set contains enough evidence to answer.
- **Faithful:** the answer cites those passages and avoids extra claims.
- **Performant:** p95 latency and cost fit your SLOs.

## 6.2 Embedding Model Choices

- **Dimensions & cost:** Higher-dimensional vectors can lift recall but raise memory/latency.
- **Domain fit:** Legal/biomed/code domains benefit from domain-tuned embeddings.
- **Multilingual:** Pick a model that preserves cross-lingual semantic proximity if you index mixed languages.
- **Normalization:** Lowercase, strip punctuation minimally, do not destroy domain tokens (IDs, codes).

**Checklist:**

- Cosine vs. dot-product consistency with index library
- Stable pre-processing (no “surprise” truncation)
- Embedding refresh policy tied to pipeline version

## 6.3 Indexing: Lexical, Vector, and Hybrid

- **Lexical (BM25/Okapi):** exact term matches; cheap & strong for keywords, IDs, and short queries.
- **Vector (HNSW / IVF-PQ):** semantic proximity; good for paraphrase and recall on fuzzy asks.
- **Hybrid:** score both and fuse (learned or heuristic). This is the practical default.

**Fusion options:**

- Weighted sum: `score = α * bm25_norm + (1-α) * vector_norm`
- Reciprocal rank fusion (RRF): stable under model drift; good baseline.
- Learning-to-rank: train small model on click/eval data (needs labels).

> **Pro Tip: Why Hybrid Search is a Superpower 💡**
>
> Lexical and vector search have complementary strengths. A hybrid approach lets you get the best of both worlds. Think about the types of queries they excel at:
>
> | Search Type | Excels At... | Example Query |
> | --- | --- | --- |
> | Lexical (BM25) | Keyword matching, IDs, codes, acronyms | "Find ticket INC-9403" |
> | Vector (HNSW) | Semantic meaning, concepts, paraphrasing | "How do I fix login problems?" |
>
> By combining them, your system can handle both a user searching for a specific document ID and another user describing their problem in conversational language.

**Operational notes:**

- Shard by tenant/source; keep per-shard stats.
- Use blue/green indexes for rebuilds; swap atomically.

## 6.4 Query Understanding

- Rewrite/expand: normalize synonyms and acronyms (domain glossary).
- Classifier routing: keyword-heavy → boost lexical; fuzzy → boost vector.
- Filters: time, author, product line; apply ACLs before scoring.
- Spelling: apply light correction; keep original tokens for IDs.

## 6.5 Reranking (Optional but Powerful)

Run a cross-encoder on the top-N retrieved candidates (e.g., N=50→re-rank to top-k=6). Gains: better ordering, fewer irrelevant chunks fed to the LLM.

**Costs:**

- latency (~2–10 ms/candidate on GPU)
- compute ($)

**Controls:**

- Gate by query length/complexity or only for critical surfaces (helpdesk, legal).

> **Pro Tip: Choosing a Reranker Model**
>
> You don't need to train a cross-encoder from scratch. Several powerful, open-source models are available that you can use directly. Look for lightweight but high-performing models on the MTEB (Massive Text Embedding Benchmark) leaderboard. Popular choices often include models from research groups like mixedbread-ai or providers like Cohere, which are designed to be dropped into your retrieval pipeline with minimal effort.

## 6.6 Chunking Recap (Why It Matters Here)

Your Chapter-5 chunking parameters directly shape retrieval:

- Target size (≈300–600 tokens) balances recall and waste.
- Overlap (10–20%) preserves context across boundaries.
- Structural first (headings), then windowing within large sections.

## 6.7 Context Assembly for Generation

**Goal:** a small, diverse, deduped set of passages that together answer the query.

> **Author's Note:** The process of retrieval, reranking, and assembly is a multi-stage funnel. A diagram here would significantly aid comprehension. The diagram should be a flowchart showing:
> 1. User Query enters.
> 2. Hybrid Retrieval fetches a wide set of candidates (e.g., N=50).
> 3. Reranker re-orders the set to promote relevance (Top 50 sorted).
> 4. MMR/Diversity selects a smaller, non-redundant set (e.g., k=6).
> 5. Trimming & Formatting creates the final Context Block for the LLM.

**Steps:**

1. Retrieve top-N (hybrid), optionally rerank to top-k (e.g., 6).
2. MMR (Max Marginal Relevance) or diversity re-rank to avoid near-duplicates.
3. Budget to window: trim each chunk to query-relevant spans (token saver).
4. Order by utility (most-essential first) and mark citations `doc_id#chunk_id`.
5. Build the prompt context block: provenance headers + text with boundaries.

Remember the "lost in the middle" problem: models pay the most attention to information at the very beginning and very end of the context window. For this reason, ensure your Order by utility step places the single most relevant chunk at the top of the context block you provide to the LLM.

**Anti-hallucination prompt bind:**

> Use ONLY the provided context. If insufficient, set `status:"uncertain"` and list what is missing.  
> Cite sources with their IDs like `[S1]`, `[S2]` next to claims they support.

## 6.8 Citations That Users Trust

- Pass stable IDs with human-readable titles: `[S3: “Reset MFA” (KB-2119) §2]`.
- Preserve offsets (char/token ranges) so UI can highlight the exact span.
- Reject answers without at least one citation if policy requires grounding.

## 6.9 Measuring RAG

**Retrieval metrics:**

- Recall@k: does the gold passage appear in the top-k?
- nDCG@k: graded relevance ranking quality.

**Answer metrics:**

- Groundedness/Attribution: % of claims supported by retrieved text.
- Hallucination rate: fraction of unsupported claims.

## 6.10 Cost & Latency Budgets

- Retrieval target: ≤50–80 ms p95 on warm caches.
- Rerank: ≤150 ms budgeted for critical paths only.
- Context tokens: cap at 1.5–2× the model’s expected answer length.

## 6.11 Safety & Privacy in RAG

- Pre-retrieval filters: tenant ACLs, document sensitivity classes.
- PII/Secrets redaction in chunks (done in Chapter-5 pipeline).
- Refusals when context is insufficient or restricted.

## 6.12 Failure Modes & Debugging

- High recall, low groundedness → context assembly or prompts are at fault.
- Low recall → chunking too coarse, poor embeddings, or domain terms missing.
- Latency spikes → index swaps, cold caches, or oversized k/N.

## Hands-On Lab: “Hybrid RAG with Reranking & Grounding”

**Goal:** Implement hybrid retrieval (BM25 + vector), optional reranking, and a context assembly step with citations. Measure recall@k, groundedness, hallucination rate, and latency; wire into ACME Assistant.

### Step 1 — Prep

Prepare indexes and evaluation queries.

### Step 2 — Hybrid Retrieve

Implement `retrieve_hybrid()` with lexical and vector components.

### Step 3 — Optional Rerank

Apply a cross-encoder to the retrieved set.

### Step 4 — Diversity & Trimming

Apply MMR and trim chunks.

### Step 5 — Prompt & Generate

Build the context block and generate the answer.

### Step 6 — Evaluate

Measure retrieval, answer, and performance metrics.

## Acceptance Criteria

- recall@5 ≥ baseline +10 points (or ≥0.75 absolute)
- Groundedness ≥ 90%; hallucination rate ≤ 5%
- p95 added latency (retrieval+rerank) ≤ 200 ms on warmed index

## Stretch Goals

- Train light learning-to-rank on eval clicks/labels
- Implement blue/green index swap with integrity check + canary

## Design Reviews & Anti‑Patterns

Vector-only or lexical-only dogma: hybrid usually wins.  
Stuffing 20 chunks: wastes tokens; pick k≤6 with trimming and diversity.  
“Retriever is perfect, model will cope”: retrieval errors amplify hallucinations.

## Checklists

### Retrieval Readiness

- [ ] Embedding model & pre-processing locked
- [ ] BM25 + vector indexes built and documented
- [ ] Fusion method & parameters versioned
- [ ] MMR/diversity + trimming parameters set

### Operational Readiness

- [ ] ACL filters enforced pre-ranking
- [ ] Blue/green index swap & rollback
- [ ] Metrics: recall@k, groundedness, hallucination rate, p95 latency

## Quick Quiz

- Why does hybrid (BM25 + vector) usually outperform either alone?
- What does MMR optimize for during context assembly?
- Two concrete ways to lower hallucination rate in RAG answers.
- When should you enable reranking—and how do you keep its cost in check?

## What’s Next

**Chapter 7 — Hybrid RAG + Fine-Tuning:** We’ll layer a small behavioral fine-tune over your RAG system—using RAG for facts and tuning for tone/format—then model the latency/cost trade-offs and compare against RAG-only or tune-only baselines.
