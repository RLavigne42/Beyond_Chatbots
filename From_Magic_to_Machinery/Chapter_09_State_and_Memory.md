# Chapter 9: State & Memory

**Synopsis:** LLMs are stateless; products aren’t. This chapter adds request, session, user, and global memory to your application—safely. You’ll design data models, summarization buffers, vector memory, and privacy/retention policies. You’ll implement consent, TTL, redaction, cross-tenant isolation, and context pruning so the assistant remembers what it should—and forgets what it must.

**Estimated time:** 120–150 min read · 90–120 min lab  
**Prereqs:** Ch. 1–8; basic DB + queue familiarity.

## Learning Objectives

- Map use cases to memory scopes (request/session/user/global) and choose storage backends.
- Build summarization buffers and vector memory with conflict resolution and pruning.
- Enforce privacy: consent UX, PII redaction, TTL, encryption, audit logs, tenant isolation.
- Prevent memory poisoning (prompt injection, untrusted RAG snippets) and leakage.
- Measure memory’s UX impact (task success, re-ask rate) and cost/latency overhead.

## 9.1 Why Memory?

LLMs reset each call; users expect continuity. Memory reduces repetition (“my timezone is…”, “preferred tone is…”) and enables multi-step work. Done wrong, it stores secrets forever or amplifies bad context. Memory is a product decision, not just a cache.

## 9.2 Memory Scopes & Responsibilities

| Scope | Lifetime | Examples | Risks | Store |
| --- | --- | --- | --- | --- |
| Request | single call | tool results, scratch vars | leakage to user | in-process, ephemeral |
| Session | conversation | running summary, open tasks | context bloat | KV store + transcript DB |
| User | across sessions | profile, preferences, stable facts | privacy, consent, drift | relational DB + vector index |
| Global | all users | product glossary, FAQs | bias, leakage | curated KB (RAG) |

Rule: lowest viable scope. Don’t put user data into global; don’t persist request-scope data.

> **Pro Tip: Choosing the Right Database for the Job 🗄️**
>
> The "Store" column in the table isn't arbitrary. Each database type is chosen for its specific strengths:
>
> - **KV Store (e.g., Redis, Dragonfly) for Session Memory:** These are perfect for session data because they offer extremely low-latency reads/writes and have built-in support for Time-to-Live (TTL), automatically expiring old sessions.
> - **Relational DB (e.g., PostgreSQL) for User Profiles:** User data is structured, requires strong consistency, and has clear relationships (a user has many sessions). A relational database is the classic, correct choice for this.
> - **Vector Index (e.g., in Pinecone, OpenSearch) for User Facts:** When you need to find memory items based on semantic similarity ("you mentioned something about a project budget..."), a vector index is the only tool that can efficiently perform that kind of search.

## 9.3 Data Model (Contract-First)

**Core tables/collections**

- `sessions(session_id, user_id, started_at, last_active_at, locale, device)`
- `messages(msg_id, session_id, role, text, tokens_in, tokens_out, created_at)`
- `session_memory(session_id, summary, salience_score, ttl_at, redaction_state)`
- `user_profile(user_id, fields JSONB, consent_version, pii_hashes, updated_at)`
- `user_memory_vectors(user_id, vector, payload {title, text, tags, source_id, ttl_at})`

**Metadata fields**

- `lineage`, `visibility`, `acl`, `retention`

> **Author's Note:** A simple Entity-Relationship Diagram (ERD) would be highly effective here to visually represent the relationships between the sessions, messages, user_profile, and memory tables. This would help readers grasp the overall schema at a glance.

## 9.4 What to Remember (and What Not)

**Good candidates**

- Stable preferences: tone, reading level, units, language.
- Long-lived context: team/org names, current project, CRM account IDs.
- Open loops: tasks awaiting follow-up, drafts in progress.

**Avoid**

- Secrets: API keys, passwords, access tokens.
- Sensitive PII unless strictly necessary and consented.
- Volatile state better re-derived from source-of-truth APIs.

## 9.5 Summarization Buffers (Session Memory)

Pattern: rolling summary + pointers instead of raw transcripts.

- **Trigger:** every N messages or M tokens.
- **Method:** summarize by roles (user intents, system constraints, decisions), keep UUID pointers to original messages.
- **Policy:** cap summary at K tokens; salience weighting (recent + importance tags).
- **System hint to model:** “Prefer session summary over full transcript. If in doubt, ask.”

## 9.6 User Memory (Profile + Facts)

- **Profile fields (typed):** name, locale, timezone, tone, units, industry.
- **Facts store (vector memory):** short atomic notes (“prefers examples with code”), tagged and timestamped.
- **Upserts:** new fact → dedupe by semantic similarity + rule-based merge; ask to confirm for ambiguous merges.
- **Confidence:** store source (user, inferred, org-admin) and confidence ∈ [0,1]; use low-confidence facts only as hints.

For example, if the system has a stored fact `{"key": "project_deadline", "value": "October 5th"}` and the user later says, "the deadline for the project is now October 12th," the system should handle the conflict gracefully. The extractor would identify the new fact, a similarity search would find the existing one, and the orchestrator would trigger a clarifying question: "I have the project deadline as October 5th. Should I update it to October 12th? [Yes/No]". This confirmation loop is essential for maintaining accurate and trusted user memory.

## 9.7 Privacy, Consent, and Compliance

- **Just-in-time consent:** “May I remember your timezone for next time? [Yes/No]”
- **Granular toggles:** per-field memory on/off; “forget this” per item.
- **Retention:** per-scope TTLs (e.g., session = 30d, user fact = 180d unless refreshed).
- **Encryption:** at rest (column/field-level for PII); in transit.
- **Deletion:** hard delete cascades (vectors, logs, caches) + tombstone.

## 9.8 Preventing Memory Poisoning & Leakage

**Inputs you must distrust**

- User-provided text (“ignore the system”, jailbreaks).
- Retrieved snippets (RAG) with hostile instructions.

**Defenses**

- Delimiters & role separation (already in Ch. 3).
- Write contract: only specific extractors can write to memory; generation prompts cannot.
- Quarantine: store unverified suggestions in scratch; require confirmation to promote.

## 9.9 Context Assembly with Memory

**Order of operations**

1. Request features (task type, locale)
2. Session summary (≤ K tokens)
3. User profile (only requested or relevant fields)
4. User facts from vector memory (k≤3, MMR diversity)
5. Task-specific RAG context

**Budgeting:** reserve strict token quotas per source; if over budget → prune in this order: user facts (low confidence) → session summary tail → RAG tail.

> **Design Review: Memory Isn't Free**
>
> While memory dramatically improves user experience, it introduces three distinct costs you must manage:
>
> - **Infrastructure Cost:** The databases for storing profiles, sessions, and vectors have a direct hosting cost.
> - **Latency Cost:** Every memory lookup is a network roundtrip that adds milliseconds to your end-to-end latency. This is why low-latency stores and tight p95 targets (like the lab's 25ms) are critical.
> - **Token Cost:** Every piece of retrieved memory is added to the prompt, consuming valuable context window space and increasing the token cost of every API call. The aggressive pruning in your context assembler isn't just for performance; it's a direct cost control measure.

## 9.10 Measuring Memory’s Value

**KPIs**

- Re-ask rate (how often we ask for data we should know)
- Task success uplift on multi-turn tasks
- Tokens saved per session (via summaries)
- Latency delta (with/without memory fetch)

## Hands-On Lab: “Safe Session & User Memory”

**Goal:** Add session summaries + user profile/fact memory with consent, TTL, redaction, and vector recall; integrate into ACME Assistant.

### Step 1 — Schema & Stores

Create relational tables and a vector store.

### Step 2 — Consent & Redaction

Implement middleware for consent and a PII redactor.

### Step 3 — Session Summary Buffer

Run a summarizer every N messages.

### Step 4 — User Facts (Vector Memory)

Extract facts and dedupe before writing.

### Step 5 — Context Assembler

Merge memory into the context with pruning.

### Step 6 — Evals & Metrics

A/B test memory vs. no-memory.

## Acceptance Criteria

- Memory fetch p95 ≤ 25 ms; added tokens ≤ 400 on average
- Re-ask rate ↓ by ≥ 30% on multi-turn tasks
- 0 writes without consent; deletion cascade verified

## Design Reviews & Anti‑Patterns

“Save everything.” Collect minimally; ask permission; set TTLs.  
Letting generator prompts write memory. Only dedicated extractors may write.  
Unlimited session context. Summarize early; prune aggressively.

## Checklists

### Design & Policy

- [ ] Scope mapping done; minimal persistence chosen
- [ ] Consent UX (granular) and policy text approved
- [ ] TTLs by scope; deletion cascade tested

### Implementation

- [ ] Role-aware session summary with token caps
- [ ] Vector memory for user facts with dedupe + confidence
- [ ] Context assembler with strict budgets & prune order

## Quick Quiz

- Name one item for session memory and one for user memory—and why.
- What’s the safest mechanism to prevent model-generated hallucinations from contaminating memory?
- How would you prune context when you exceed token budgets?

## What’s Next

**Chapter 10 — Agents & Tool Use**
