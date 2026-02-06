## Chapter 5: Data Pipelines & Quality

The Prodigy was now a brilliant reasoner, but there was a problem: they were still working from memory. When asked about ACME’s specific health insurance plans or the office move date, the Prodigy would confidently give the wrong answer because that information wasn't in their original training. The Manager realized it was time to build the **Corporate Library**.

In this chapter, we build the **Data Foundation**. This isn't just a folder of PDFs; it’s an industrial-grade pipeline that cleans, cuts, and organizes your company's knowledge so the Assistant can find the right fact in milliseconds.

---

### 5.1 The Data Contract: Garbage In, Garbage Out

The Manager knew that if she filled the library with messy, duplicate, or outdated memos, the Prodigy would give messy, duplicate, or outdated answers. We start by defining a **Document Contract**.

**What & Why: Normalization**

* **What it is:** Converting every file type (PDF, HTML, Word, Slack) into a clean, standard text format (usually Markdown).
* **Why we need it:** Models get confused by "invisible" noise like HTML tags or weird PDF formatting. Clean text leads to higher accuracy and lower token costs.

---

### 5.2 Chunking: Slicing the Knowledge

You can't hand the Prodigy a 200-page manual and ask them to "find the policy." They’ll get overwhelmed or forget the middle. We must perform **Chunking**.

**The Intuitive Analogy: The Index Cards**
Imagine taking a massive book and cutting it into small, 500-word index cards. Each card must be small enough for the Prodigy to read quickly, but big enough to contain a complete thought.

**Trade-offs & Decisions: Chunk Size vs. Context**

* **Small Chunks:** Very precise, but might miss the "big picture."
* **Large Chunks:** Great context, but expensive and might "distract" the model with irrelevant info.
* **Our Decision:** We use **Structural Chunking**. We split documents at natural boundaries (like headings) to ensure each "index card" is a logical unit of information.

---

### 5.3 Deduplication: Cleaning the Shelves

**The What & Why: Semantic Deduplication**

* **What it is:** Finding and removing chunks that say the same thing, even if the words are slightly different.
* **Why we need it:** If three different documents explain the same "Holiday Policy," the Assistant might get confused or provide redundant citations.

---

### 5.4 Metadata & Lineage: The Library Catalog

Every chunk in our library needs a "ID Tag" (Metadata). We must track:

* **Source:** Where did this come from? (e.g., `benefits_manual.pdf`)
* **Freshness:** When was this last updated?
* **Access Control:** Who is allowed to see this? (e.g., `Role: HR-Only`)

**The What & Why: Lineage**

* **What it is:** A record of every step the data took from the original file to the library shelf.
* **Why we need it:** If the Prodigy gives a wrong answer, we need to be able to trace it back to the exact version of the source document to fix it.

---

### 🛠️ Hands-On Lab: "The Ingestion Pipeline"

**Goal:** Build a script that takes a folder of mixed documents and turns them into a clean, chunked, and tagged JSON dataset.

**Step 1: The Normalizer**
Use a library (like `Unstructured` or `PyMuPDF`) to turn a PDF into Markdown text. Strip out the "boilerplate" (headers, footers, page numbers).

**Step 2: The Structural Chunker**
Write a function that splits the Markdown into chunks whenever it sees a `#` (Heading). Cap each chunk at 500 tokens.

**Step 3: Tagging**
Attach a JSON metadata object to every chunk including the `file_name` and a `hash` of the text (to detect changes later).

**Acceptance Criteria:**

* A folder of JSON files where each file represents a single, clean "index card."
* Metadata includes source URI and a timestamp.
* No chunks exceed the 500-token limit.

---

### 🚩 Anti-Patterns: Manager's Warnings

* **"Blind PDF Dumps":** Never just shove raw PDFs into your system. Tables and multi-column layouts will break the Prodigy's logic.
* **"The One-Time Load":** Data changes. If your pipeline doesn't have a way to *update* or *delete* old chunks, the Assistant will start lying as the world changes.

---

### Checklists: Data Readiness

* [ ] Source documents identified and permissions granted.
* [ ] Normalization logic handles your most common file types.
* [ ] Metadata schema defined (Source, Title, Date, ACL).

**What's Next:**
Our library shelves are full of index cards. Now we need a way for the Prodigy to find the right one instantly. In **Chapter 6: Vector Search & RAG Deep Dive**, we build the high-speed search engine that powers our Assistant's memory.
