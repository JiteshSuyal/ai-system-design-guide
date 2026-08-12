# RAG Data Ingestion & Automated Ingestion — Interview-Focused Notes

> Based on `06-retrieval-systems/15-data-engineering-for-ai.md` from the `JiteshSuyal/ai-system-design-guide` repository.
>
> Goal: understand the production data/ingestion side of RAG, especially automated and incremental ingestion.

## 1. RAG Has Two Pipelines

A basic RAG diagram is:

```text
User Query → Embedding → Vector DB → Context → LLM → Answer
```

That is only the **online/query path**.

A production RAG system also needs an **offline/ingestion path**:

```text
Data Sources
   ↓
Detect & Route
   ↓
Parse / Extract
   ↓
Clean / Normalize
   ↓
Deduplicate
   ↓
PII / Governance
   ↓
Quality Filter
   ↓
Chunk
   ↓
Enrich
   ↓
Embed
   ↓
Vector Store
```

The key interview insight is:

> RAG is not only a retrieval problem. It is also a data-engineering problem.

If ingestion is bad, retrieval will be bad regardless of how good the LLM or vector database is.

---

## 2. Why Ingestion Matters

Real knowledge bases contain heterogeneous data:

```text
PDFs
Scanned PDFs
Word / Office documents
HTML
Emails
Images
Databases
APIs
User uploads
Cloud storage
```

A raw document is not automatically RAG-ready.

You need:

```text
Raw data
   ↓
Structured + clean + governed data
   ↓
Chunks
   ↓
Embeddings
   ↓
Vector DB
```

The repository makes the broader point that data engineering is a standalone discipline that feeds both RAG and fine-tuning.

---

# 3. Shared Data Pipeline

The repository describes a shared pipeline with two consumers:

```text
                       RAW SOURCES
                            |
                            v
                   Detect & Route
                            |
                            v
                    Parse / Extract
                            |
                            v
                   Clean / Normalize
                            |
                            v
                      Deduplicate
                            |
                            v
                    Governance / PII
                            |
                            v
                     Quality Filter
                       /                             /                              v           v
                   RAG       Fine-tuning
                    |             |
             chunk/embed    curate/label
                    |             |
                Vector DB     Training data
```

The first stages are shared.

The branches diverge near the end.

For RAG:

```text
chunk → enrich → embed → vector store
```

For fine-tuning:

```text
curate/label → synthesize → decontaminate → balance → training records
```

---

# 4. Stage 1 — Detect & Route

## Problem

Never blindly trust:

```text
report.pdf
```

The extension can be wrong.

A PDF may actually contain scanned page images.

If you send a scanned PDF through a normal text extractor, you may get empty or incomplete text.

## Solution

Detect the actual content type using MIME/content sniffing.

Conceptually:

```text
File
 ↓
Content inspection
 ↓
Actual MIME type
 ↓
Correct processing route
```

A common approach is magic-number detection, for example through `libmagic`.

### Routing examples

```text
Text PDF
   ↓
Fast text extraction

Scanned PDF
   ↓
OCR

Complex table-heavy document
   ↓
Layout-aware / multimodal processing

Office document
   ↓
Format-specific parser

Email
   ↓
Headers + body
```

### Interview answer

> File extensions are not reliable enough for production ingestion. I would inspect the actual content/MIME type and route the document to the appropriate parser, OCR pipeline, or layout-aware processor.

---

# 5. Language Detection

Language can affect:

- parsing
- filtering
- embeddings
- retrieval
- generation

So the pipeline can perform:

```text
Document
   ↓
Language identification
   ↓
Route / filter
```

For example:

```text
English → English-capable processing
Hindi   → Hindi-capable processing
Marathi → Marathi-capable processing
```

---

# 6. Stage 2 — Parse / Extract

Extraction is not always:

```text
PDF → plain text
```

A document can contain:

```text
Title
Heading
Paragraph
Table
Image
Caption
List
Code
Footer
```

A useful parser preserves structure:

```text
Document
   ↓
Structured elements

Title(...)
Heading(...)
Paragraph(...)
Table(...)
Paragraph(...)
```

The repository mentions tools such as Unstructured, Docling, and LlamaParse. The important interview concept is:

> Different document types require different extraction strategies.

---

# 7. OCR vs Normal Text Extraction

### Normal PDF

```text
PDF
 ↓
Text extraction
 ↓
Text
```

### Scanned PDF

```text
Scanned PDF
 ↓
Page image
 ↓
OCR
 ↓
Text
```

### Complex document

```text
Complex layout
 ↓
Layout-aware / multimodal processing
 ↓
Structured content
```

Failure chain:

```text
Bad extraction
 ↓
Bad chunks
 ↓
Bad embeddings
 ↓
Bad retrieval
 ↓
Bad answer
```

This is why retrieval quality cannot compensate for bad ingestion.

---

# 8. Stage 3 — Cleaning & Normalization

Raw extracted content may contain:

```text
Navigation
Advertisements
Cookie banners
Headers
Footers
Broken encoding
Inconsistent Unicode
Duplicate whitespace
```

Cleaning should happen before downstream processing.

## Boilerplate removal

For web pages:

```text
Raw HTML
 ↓
Content extraction
 ↓
Main article/content
```

You don't want navigation and cookie banners embedded into the vector store.

## Encoding repair

Fix corrupted text such as:

```text
cafÃ©
```

instead of:

```text
café
```

## Unicode normalization

Visually identical strings can have different underlying representations.

Normalization such as NFC/NFKC helps make equivalent strings comparable.

This matters for:

```text
Deduplication
Matching
Search
```

---

# 9. Important Lesson: More Filtering Is Not Always Better

A common mistake is:

```text
More filtering
 ↓
Cleaner data
 ↓
Better RAG
```

Not necessarily.

Aggressive filtering can remove useful information.

Therefore:

```text
Filter
 ↓
A/B or ablation test
 ↓
Measure downstream retrieval/model quality
```

Keep a filter only when it actually helps the target system.

---

# 10. Stage 4 — Deduplication

Suppose your knowledge base contains:

```text
Document A
Document A duplicate
Document A duplicate
Slightly modified A
Paraphrased A
```

Embedding everything creates redundant vectors.

That causes:

- wasted compute
- wasted storage
- duplicate retrieval results
- less evidence diversity
- wasted context window

The repository describes a three-tier cascade.

---

# 11. Exact Deduplication

Hash documents or normalized text.

```text
Document
 ↓
Hash
 ↓
Fingerprint
```

If:

```text
hash(A) == hash(B)
```

they are exact duplicates.

Advantages:

- cheap
- fast
- precise

Limitation:

```text
"The automobile was parked outside."

"The car was left outside."
```

These are semantically similar but not exact.

---

# 12. Fuzzy / Near-Duplicate Deduplication

A common approach is:

```text
MinHash + LSH
```

It estimates lexical similarity using token shingles.

The reason LSH is useful is scale.

You do not want expensive all-pairs comparisons across millions of documents.

Conceptually:

```text
Millions of documents
       ↓
LSH buckets
       ↓
Likely-similar candidates
       ↓
Detailed comparison
```

MinHash is mainly lexical, so it can still miss paraphrases.

---

# 13. Semantic Deduplication

Semantic deduplication uses embeddings:

```text
Document
 ↓
Embedding
 ↓
Similarity
 ↓
Cluster similar items
 ↓
Remove near-duplicates
```

Example:

```text
"The automobile was parked outside."

"The car was left outside."
```

Different wording, similar meaning.

Semantic methods can catch cases that lexical methods miss.

### Production cascade

```text
Exact hash
   ↓
MinHash / LSH
   ↓
Semantic deduplication
```

The reason for this ordering:

```text
Exact       → cheapest
Fuzzy       → moderate cost
Semantic    → more expensive
```

---

# 14. Why Dedup Matters in RAG

Imagine top-5 retrieval returns:

```text
1. Document A
2. Document A duplicate
3. Document A duplicate
4. Document A duplicate
5. Document B
```

The LLM effectively received only two useful evidence sources.

Better:

```text
1. Document A
2. Document B
3. Document C
4. Document D
5. Document E
```

So deduplication improves evidence diversity.

---

# 15. Stage 5 — PII, Consent & Governance

Production documents may contain:

```text
Names
Emails
Phone numbers
Addresses
Government IDs
Financial information
Other sensitive data
```

The pipeline may need to:

```text
Detect
 ↓
Redact / mask / hash / encrypt
 ↓
Store provenance and policy metadata
```

The repository mentions Microsoft Presidio as a PII detection/redaction option.

---

# 16. Governance Metadata

A useful chunk record might contain:

```json
{
  "document_id": "doc_123",
  "version": 4,
  "source": "company_policy.pdf",
  "author": "HR",
  "timestamp": "2026-08-12T10:00:00Z",
  "license": "internal",
  "consent": true
}
```

This lets you answer:

- Where did this chunk come from?
- Which document version produced it?
- Can this user access it?
- When was it indexed?

Governance should therefore be treated as metadata carried through the pipeline, not only as a final gate.

---

# 17. Stage 6 — Quality Filtering

Quality filtering can use:

### Heuristics

```text
length
symbol-to-word ratio
repetition
formatting
```

### Model-based classification

```text
Document
 ↓
Quality classifier
 ↓
Quality score
 ↓
Keep / reject
```

But quality classifiers can be miscalibrated.

So again:

> Validate filters against downstream evaluation rather than trusting them by reputation.

---

# 18. Stage 7 — Chunking

Large documents need to be divided into retrieval units.

```text
Document
   ↓
Chunk 1
Chunk 2
Chunk 3
...
Chunk N
```

Possible strategies:

```text
Fixed-size
Recursive
Semantic
Structure-aware
```

There is no universally correct chunk size.

The correct strategy depends on:

- document structure
- query patterns
- embedding model
- retrieval evaluation
- downstream context limits

---

# 19. Stage 8 — Enrichment

Don't store only:

```text
embedding
```

Store useful metadata too:

```json
{
  "document_id": "doc_123",
  "chunk_id": "chunk_007",
  "title": "Refund Policy",
  "section": "Eligibility",
  "author": "Finance",
  "timestamp": "2026-08-12",
  "source": "policy.pdf"
}
```

Now retrieval can combine:

```text
semantic similarity
+
metadata filtering
```

For example:

```text
semantic search
WHERE department = "finance"
AND year >= 2025
```

---

# 20. Contextual Enrichment

A chunk can lose context after being separated from its document.

Original:

```text
Document: Employee Leave Policy
Section: Eligibility

"Employees become eligible after six months."
```

The chunk alone may not contain enough context.

Enrichment can turn it into something conceptually like:

```text
[Employee Leave Policy]
[Eligibility]

Employees become eligible after six months.
```

This gives the embedding more useful context.

---

# 21. Stage 9 — Embedding

Now convert each chunk into a vector:

```text
Chunk
 ↓
Embedding model
 ↓
Vector
```

Then store:

```text
Vector DB
 ├── embedding
 ├── chunk text
 ├── document ID
 ├── metadata
 └── version
```

This is where the ingestion pipeline connects to the vector database you have been studying.

---

# 22. Complete Basic Ingestion Pipeline

```text
                         DATA SOURCES
                              |
              +---------------+----------------+
              |               |                |
             PDF             DB               API
              |               |                |
              +---------------+----------------+
                              |
                              v
                       DETECT & ROUTE
                              |
                              v
                       PARSE / EXTRACT
                              |
                              v
                      CLEAN / NORMALIZE
                              |
                              v
                         DEDUPLICATE
                              |
                              v
                     PII / GOVERNANCE
                              |
                              v
                      QUALITY FILTER
                              |
                              v
                           CHUNK
                              |
                              v
                          ENRICH
                              |
                              v
                         EMBEDDING
                              |
                              v
                        VECTOR STORE
```

---

# 23. Now the Important Part — Automation

A demo application may do:

```text
Upload PDF
 ↓
Run script
 ↓
Chunk
 ↓
Embed
 ↓
Vector DB
```

Production systems cannot depend on a developer manually running the script.

Imagine:

```text
10,000 documents
500 new per day
100 updated per day
50 deleted per day
```

You need automatic ingestion.

---

# 24. Batch vs Incremental Ingestion

## Batch

```text
All documents
 ↓
Process all
 ↓
Build/rebuild index
```

Good for:

- initial indexing
- migrations
- large bulk imports

## Incremental

```text
Only changed documents
 ↓
Process delta
```

If you have:

```text
1,000,000 documents
```

and only:

```text
100 changed
```

you should process approximately the changed set, not the entire corpus.

This is the key production scaling idea.

---

# 25. Change Data Capture (CDC)

CDC detects source changes.

Suppose:

```text
doc_1
doc_2
doc_3
```

Then:

```text
doc_2 updated
doc_3 deleted
doc_4 created
```

CDC can produce events conceptually like:

```text
UPDATE doc_2
DELETE doc_3
INSERT doc_4
```

The ingestion pipeline reacts to those events.

---

# 26. CDC → RAG

```text
                 SOURCE SYSTEM
                      |
                      v
                     CDC
                      |
          +-----------+-----------+
          |           |           |
        INSERT      UPDATE      DELETE
          |           |           |
          v           v           v
       Process      Reprocess    Remove
          |           |           |
          v           v           v
       Embed       Re-embed    Delete vectors
          \           |           /
           +----------+----------+
                      |
                      v
                  VECTOR DB
```

This keeps the RAG index fresh.

---

# 27. New Document Automation

Suppose:

```text
company-policy.pdf
```

is uploaded to object storage.

A production flow could be:

```text
Object Storage
      ↓
Upload Event
      ↓
Queue
      ↓
Ingestion Worker
      ↓
Detect Type
      ↓
Parse / OCR
      ↓
Clean
      ↓
Dedup
      ↓
PII / Governance
      ↓
Quality Check
      ↓
Chunk
      ↓
Enrich
      ↓
Embed
      ↓
Vector DB
      ↓
Mark Indexed
```

The upload request does not need to wait for the entire process.

---

# 28. Why Use a Queue?

Suppose 10,000 documents arrive at once.

Without a queue:

```text
10,000 events
 ↓
10,000 simultaneous jobs
 ↓
overload
```

With a queue:

```text
10,000 events
      ↓
     QUEUE
      ↓
Controlled workers
      ↓
Vector DB
```

A queue provides:

- backpressure
- retries
- controlled concurrency
- asynchronous processing
- failure isolation

---

# 29. Idempotency

This is one of the most important production concepts.

Suppose the same event arrives twice:

```text
UPLOAD doc_123
UPLOAD doc_123
```

You don't want:

```text
duplicate chunks
duplicate vectors
```

Make ingestion idempotent.

A useful identity can be:

```text
document_id + version
```

For example:

```text
doc_123:v4
```

And deterministic chunk IDs:

```text
doc_123:v4:chunk_001
doc_123:v4:chunk_002
```

Then repeated processing can safely use upsert behavior.

---

# 30. Document Versioning

Suppose:

```text
policy.pdf v1
```

is indexed.

Later:

```text
policy.pdf v2
```

arrives.

Track:

```text
document_id = policy
version = 2
```

Then:

```text
new version
   ↓
process
   ↓
remove/retire old vectors
   ↓
upsert new vectors
   ↓
mark v2 active
```

Without versioning, old and new content can coexist and create conflicting retrieval results.

---

# 31. Updates and Stale Data

Suppose v1 says:

```text
Refund period = 30 days
```

and v2 says:

```text
Refund period = 15 days
```

If both remain active:

```text
Vector DB
 ├── old → 30 days
 └── new → 15 days
```

The LLM could receive conflicting context.

Therefore updates must correctly retire or replace old vectors.

---

# 32. Deleted Documents

Deletion is just as important as insertion.

If:

```text
Source document deleted
```

but:

```text
Vector DB still contains it
```

the RAG system can retrieve content that should no longer exist.

Correct flow:

```text
DELETE source
     ↓
DELETE corresponding vectors
```

This is why automated ingestion must handle:

```text
INSERT
UPDATE
DELETE
```

not just uploads.

---

# 33. Avoid Full Re-indexing

A common bad architecture is:

```text
One document changes
 ↓
Reprocess 1 million documents
```

Better:

```text
Document change
 ↓
CDC/event
 ↓
Identify affected document
 ↓
Process only that document
 ↓
Update vectors
```

This reduces:

- compute
- embedding API cost
- processing time
- vector DB load

---

# 34. Orchestration

As the pipeline grows:

```text
detect
→ parse
→ OCR
→ clean
→ dedup
→ PII
→ chunk
→ embed
→ index
```

you need orchestration.

The chapter mentions:

### Airflow

Mature and has a large ecosystem.

### Dagster

Asset-oriented and useful for dependency-aware/incremental workflows.

### Prefect

A lighter Python-oriented option.

Important distinction:

```text
Airflow / Dagster / Prefect
= orchestration
```

while:

```text
Spark / Ray
= distributed compute
```

Spark/Ray perform computation; an orchestrator schedules and coordinates workflows.

---

# 35. Retries and Dead-Letter Queues

A document can fail because of:

- corrupt file
- OCR failure
- parser failure
- embedding API failure
- vector DB outage

Don't simply crash the whole pipeline.

Use:

```text
Job
 ↓
Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
Dead Letter Queue
```

An engineer can inspect the failed document later.

---

# 36. Observability

Monitor the ingestion pipeline itself.

Useful metrics:

```text
documents_received
documents_processed
documents_failed
processing_latency
chunks_generated
duplicates_removed
PII_detections
embedding_failures
vector_db_failures
documents_pending
index_lag
```

A particularly important RAG metric is:

```text
Index freshness / lag
```

Example:

```text
Source updated:     10:00
Vector DB updated:  10:02

Index lag = 2 minutes
```

If your SLA is five minutes, this is healthy.

---

# 37. Lineage

You should be able to trace:

```text
Vector
 ↓
Chunk
 ↓
Document version
 ↓
Source
 ↓
Ingestion run
 ↓
Parser version
 ↓
Embedding model
```

Example:

```text
vector_123
   ↓
chunk_07
   ↓
policy.pdf:v4
   ↓
ingestion_run_987
   ↓
parser_v2
   ↓
embedding_model_X
```

Why?

If retrieval quality suddenly changes, you need to know what changed.

Without lineage, debugging becomes difficult.

---

# 38. Versioning

Track things such as:

```text
source version
parser version
chunking strategy
embedding model
pipeline version
```

Then you can reproduce how a vector was created.

This is important for:

- debugging
- rollback
- experiments
- auditing
- reproducibility

---

# 39. Scaling

Suppose:

```text
1,000,000 documents
```

Use parallel workers:

```text
                 QUEUE
              /    |                 /     |              Worker Worker Worker
            |      |      |
          Parse  Parse   Parse
             \     |     /
              \    |    /
                Embedding
                    |
                    v
                 Vector DB
```

Different stages can have different worker counts because their bottlenecks differ.

For example, OCR may require different scaling than embedding.

---

# 40. Cost Optimization

Major ingestion costs can include:

```text
OCR
Embedding APIs
LLM-based enrichment
Compute
Storage
Vector DB
```

Incremental ingestion saves money.

Example:

```text
Full re-index:
1,000,000 documents

Incremental:
1,000 changed documents
```

Other optimizations:

```text
deduplicate before embedding
batch embedding requests
cache embeddings
don't re-embed unchanged chunks
use deterministic IDs
```

---

# 41. Complete Production Architecture

A strong interview architecture:

```text
                         DATA SOURCES
                              |
               +--------------+--------------+
               |              |              |
              S3             DB             API
               |              |              |
               +--------------+--------------+
                              |
                         CDC / Events
                              |
                              v
                            Queue
                              |
                              v
                         Orchestrator
                              |
                              v
                       Detect / Route
                              |
                              v
                         Parse / OCR
                              |
                              v
                           Clean
                              |
                              v
                           Dedup
                              |
                              v
                      PII / Governance
                              |
                              v
                       Quality Filter
                              |
                              v
                           Chunk
                              |
                              v
                          Enrich
                              |
                              v
                         Embedding
                              |
                              v
                         Vector DB
                              |
                              v
                          Retrieval
                              |
                              v
                           Rerank
                              |
                              v
                          Context
                              |
                              v
                             LLM
                              |
                              v
                           Answer
```

Cross-cutting concerns:

```text
Monitoring
Retries
DLQ
Idempotency
Versioning
Lineage
Security
Access control
```

---

# 42. Important Interview Scenario: Document UPDATE

Question:

> A document already exists in your RAG system. The source document gets updated. What happens?

Strong answer:

```text
Source update
     ↓
CDC / event
     ↓
Identify document_id + new version
     ↓
Queue event
     ↓
Parse new version
     ↓
Clean
     ↓
Dedup
     ↓
Chunk
     ↓
Enrich
     ↓
Embed
     ↓
Delete/retire old vectors
     ↓
Upsert new vectors
     ↓
Mark new version active
```

The key point:

> Do not rebuild the entire vector index for one changed document.

---

# 43. Important Interview Scenario: DELETE

```text
Source DELETE
     ↓
CDC event
     ↓
document_id
     ↓
Find all chunks for document
     ↓
Delete vectors
     ↓
Update index state
```

This prevents orphaned vectors.

---

# 44. Important Interview Scenario: Duplicate Event

Question:

> What if the same event arrives twice?

Answer:

> Make the ingestion pipeline idempotent. Use a deterministic document/version/chunk ID, maintain processing state, and use safe upsert operations so repeated events don't create duplicate vectors.

---

# 45. Important Interview Scenario: Embedding Failure

Suppose:

```text
Document
 ↓
Parse
 ↓
Clean
 ↓
Chunk
 ↓
Embedding API fails
```

Don't lose the document.

Track processing state:

```text
RECEIVED
 ↓
PARSED
 ↓
CLEANED
 ↓
CHUNKED
 ↓
EMBEDDING_FAILED
```

Then retry from an appropriate checkpoint instead of starting everything from zero.

---

# 46. Why Modular Stages Matter

Prefer:

```text
detect()
parse()
clean()
dedup()
govern()
quality_check()
chunk()
enrich()
embed()
index()
```

rather than one giant:

```text
process_document()
```

Benefits:

- easier testing
- retries
- observability
- independent scaling
- component replacement

For example, you can change the PDF parser without redesigning the vector DB or retrieval system.

---

# 47. Connection to Vector Databases

You have been studying:

```text
HNSW
ANN
Metadata filtering
Hybrid search
Vector DBs
```

Now ask:

> Where did those vectors come from?

Answer:

```text
                    RAG SYSTEM

Documents
    |
    v
+----------------------+
| INGESTION            |
| parse                |
| clean                |
| dedup                |
| chunk                |
| enrich               |
| embed                |
+----------+-----------+
           |
           v
     +-----------+
     | Vector DB |
     +-----------+
           |
           v
       Retrieval
           |
           v
          LLM
```

The vector DB is the serving/index layer. It is not the whole RAG system.

---

# 48. The Most Important Mental Model

Think of RAG as two connected systems.

## System 1 — Knowledge preparation

```text
Messy world
   ↓
Reliable data
   ↓
Reliable chunks
   ↓
Reliable vectors
```

## System 2 — Knowledge retrieval

```text
User question
   ↓
Relevant vectors
   ↓
Relevant context
   ↓
LLM
   ↓
Answer
```

If System 1 is broken:

```text
Bad extraction
 ↓
Bad chunks
 ↓
Bad embeddings
 ↓
Bad retrieval
 ↓
Bad answer
```

A better LLM cannot reliably fix broken source data.

---

# 49. What You Should Memorize for the Interview

1. RAG ingestion is a separate pipeline from online query handling.
2. Detect file type from content, not just extension.
3. Route scanned PDFs to OCR.
4. Clean and normalize before embedding.
5. Deduplicate using a cascade: exact → fuzzy → semantic.
6. Carry PII, consent, licensing, provenance, and access metadata.
7. Chunking is dataset-specific; evaluate it.
8. Enrich chunks with metadata/context.
9. Use batch processing for initial/bulk indexing.
10. Use incremental ingestion for ongoing freshness.
11. CDC/events can detect inserts, updates, and deletes.
12. Don't full-reindex for every change.
13. Make ingestion idempotent.
14. Handle document versions and deletes explicitly.
15. Use queues/workers for asynchronous scalable ingestion.
16. Use retries and DLQs for failures.
17. Monitor index freshness/lag.
18. Track lineage.
19. Version important pipeline components.
20. Monitor the ingestion pipeline, not only the vector DB.

---

# 50. Strong Interview Answer

If asked:

> **"How would you design an automated ingestion pipeline for a production RAG system?"**

A strong answer is:

> I would build an event-driven ingestion pipeline. Source systems such as object storage, databases, and APIs emit events or CDC changes into a queue. Workers detect the actual content type and route documents to the appropriate parser or OCR path. Then I would clean and normalize the content, deduplicate it, apply PII and governance checks, run quality filtering, chunk and enrich the content, generate embeddings, and upsert the chunks with metadata into the vector store.
>
> For freshness, I would process inserts, updates, and deletes incrementally rather than rebuilding the entire index. I would make the pipeline idempotent using deterministic document/version/chunk IDs, maintain document versions, add retries and a DLQ, and monitor processing failures and index lag. I would also maintain lineage so that every vector can be traced back to its source document and pipeline/model versions.

---

# 51. One-Sentence Mental Model

> **Production RAG ingestion is an automated data pipeline that continuously converts messy, changing source data into clean, governed, deduplicated, enriched, embedded, versioned and searchable knowledge — without requiring a full re-index every time something changes.**
