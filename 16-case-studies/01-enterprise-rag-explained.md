# Enterprise RAG System — Detailed Interview-Focused Notes

Source: `01-enterprise-rag.md`

This document explains the enterprise RAG case study as an actual system-design problem: what each component does, why it exists, how ingestion and query-time retrieval work, and what trade-offs matter in interviews.

---

## 1. Problem

The case study designs an internal document-search system for a financial company with:

- 500,000 documents
- 5,000 employees
- multiple departments
- daily document updates
- strict compliance/audit requirements
- answers that must include citations

The goal is natural-language Q&A over enterprise knowledge.

The key idea is:

```text
User question
   ↓
Find relevant enterprise evidence
   ↓
Rank the evidence
   ↓
Give evidence to the LLM
   ↓
Generate grounded answer + citations
```

The LLM is **not** the source of truth. The enterprise data is.

---

# 2. Requirements

## Functional

### Natural-language Q&A
Users can ask:

> What is our annual leave policy?

instead of searching exact keywords.

### Source citations
Answers must identify the source document/page. This is important for trust and compliance.

### Multi-document reasoning
Questions may require information from several documents:

```text
Compare the 2025 and 2026 compliance policies.
```

### Follow-up questions
Conversation history matters:

```text
User: What is the leave policy?
Assistant: ...
User: What about contractors?
```

### Summarization
The system should summarize long documents.

## Non-functional

| Requirement | Target |
|---|---:|
| P95 latency | < 5 seconds |
| Accuracy | > 90% |
| Availability | 99.9% |
| Concurrent users | 500 |
| Freshness | < 1 hour |

P95 means 95% of requests should finish within the target latency.

## Security

- RBAC
- query audit logging
- no data leaving company network
- PII handling

The most important security rule:

> Apply permissions **during retrieval**, not after retrieving sensitive documents.

---

# 3. High-level architecture

```text
User
 ↓
UI (Web / Slack / API)
 ↓
API Gateway
 ├─ Authentication
 ├─ Rate limiting
 └─ Routing
 ↓
Query Service
 ├─ Permission checks
 └─ Orchestration
 ↓
Hybrid Retrieval
 ├─ Qdrant / semantic
 └─ Elasticsearch / keyword
 ↓
RRF
 ↓
Reranker
 ↓
Context Builder
 ↓
LLM
 ↓
Output Guardrails
 ↓
Citations
 ↓
Answer
```

The data layer is split because each store has a different job:

```text
Qdrant        → vector/semantic search
Elasticsearch → keyword search
S3/doc store  → original/full document
Postgres      → metadata + ACL
```

---

# 4. Ingestion pipeline

The ingestion path runs when documents are uploaded or changed.

```text
Document
   ↓
Parse
   ↓
Extract metadata
   ↓
Chunk
   ↓
Embed
   ↓
Qdrant
   ↓
Full document store
   ↓
Metadata DB
   ↓
Elasticsearch
```

This is different from query processing.

## 4.1 Parsing

Example:

```text
company_policy.pdf
        ↓
PDF parser
        ↓
structured text + pages + tables + headings
```

Parsing quality is critical.

If a table is extracted incorrectly:

```text
bad parsing
   ↓
bad chunks
   ↓
bad embeddings
   ↓
bad retrieval
   ↓
bad answer
```

The case study specifically calls complex PDF tables a production challenge.

## 4.2 Metadata

Example:

```json
{
  "document_id": "doc123",
  "department": "HR",
  "access_level": "internal",
  "created_at": "2026-08-15"
}
```

Metadata enables filtering such as:

```text
public
OR user's department
OR explicitly shared with user
```

## 4.3 Chunking

A large document is split into searchable units:

```text
100-page PDF
 ↓
chunk 1
chunk 2
chunk 3
...
```

The example uses:

```text
chunk_size = 512
chunk_overlap = 50
```

Conceptually:

```text
chunk 1: tokens 1–512
chunk 2: tokens 463–974
chunk 3: tokens 925–1436
```

Overlap helps preserve context between boundaries.

Do **not** memorize 512 as a universal value. Chunk size depends on the document type, retrieval method, context window, and evaluation results.

## 4.4 Embeddings

Each chunk is converted into a vector:

```text
"Employees receive 20 days of annual leave."
              ↓
       embedding model
              ↓
[0.12, -0.31, 0.77, ...]
```

The example uses `text-embedding-3-large`.

All chunk vectors are indexed in Qdrant.

## 4.5 Qdrant point

A vector record conceptually contains:

```json
{
  "id": "doc123_14",
  "vector": [0.12, -0.44, 0.91],
  "payload": {
    "document_id": "doc123",
    "chunk_index": 14,
    "text": "Employees receive...",
    "department": "HR",
    "access_level": "internal"
  }
}
```

The vector finds similar content.

The payload lets the system recover the actual chunk and metadata.

## 4.6 Full document store

Store the original/full parsed document separately:

```text
Qdrant → chunks + vectors
S3     → full/original document
```

The full document is useful for:

- audit
- reprocessing
- re-chunking
- displaying the source
- rebuilding embeddings

## 4.7 PostgreSQL

Use relational storage for structured metadata:

```text
document_id
owner
department
ACL
created_at
updated_at
version
status
```

## 4.8 Elasticsearch

Elasticsearch gives lexical/keyword search.

This matters for exact terms such as:

```text
EMP-48291
POLICY-2026-17
```

while vector search is better for semantic questions.

---

# 5. Query-time pipeline

```text
User query
   ↓
Input guardrails
   ↓
Query understanding / rewrite
   ↓
Permission filter
   ↓
Parallel retrieval
   ├─ vector search
   └─ keyword search
   ↓
RRF
   ↓
Reranker
   ↓
Context builder
   ↓
LLM
   ↓
Output guardrails
   ↓
Citations
   ↓
Answer
```

---

# 6. Input guardrails

Before retrieval, the query can be checked for policy/safety issues.

If blocked:

```text
return safe fallback
```

Otherwise continue.

---

# 7. Query understanding

A query may be rewritten using conversation history.

Example:

```text
User: What is our leave policy?
Assistant: Employees receive 20 days...
User: What about contractors?
```

The second query can be transformed into:

```text
What is the leave policy for contractors?
```

This improves retrieval.

---

# 8. Permission filtering

Suppose:

```text
user.department = Finance
user.id = 123
```

The retriever should consider:

```text
public documents
OR Finance documents
OR documents explicitly shared with user 123
```

not the whole enterprise corpus.

Correct:

```text
permission filter
      ↓
retrieval
```

Riskier:

```text
retrieval
      ↓
retrieve private data
      ↓
filter it afterwards
```

This is one of the most important enterprise-RAG design decisions.

---

# 9. Hybrid retrieval

Two searches happen in parallel.

### Semantic search

```text
query
 ↓
embedding
 ↓
Qdrant
 ↓
top 100
```

### Keyword search

```text
query
 ↓
BM25 / Elasticsearch
 ↓
top 100
```

Why both?

Semantic search handles:

- meaning
- paraphrases
- concepts

Keyword search handles:

- exact terms
- IDs
- names
- rare words
- codes

---

# 10. RRF — Reciprocal Rank Fusion

The two result lists cannot simply be compared by raw scores because vector similarity and BM25 scores have different scales.

RRF combines **ranks**.

Simplified:

```text
RRF score = weight / (k + rank)
```

The example uses:

```text
semantic weight = 0.7
keyword weight = 0.3
k = 60
```

Example:

```text
Vector:
A rank 1
B rank 2
C rank 3

Keyword:
B rank 1
D rank 2
A rank 3
```

B benefits because it ranks highly in both systems.

So:

```text
Vector search + keyword search
             ↓
            RRF
             ↓
     combined candidates
```

RRF combines rankings, not raw similarity scores.

---

# 11. Reranking

After cheap candidate retrieval:

```text
large corpus
   ↓
retrieval
   ↓
50 candidates
```

the reranker evaluates query-document relevance more carefully:

```text
50 candidates
   ↓
cross-encoder reranker
   ↓
10 best
```

Why not rerank 500,000 documents?

Because reranking is more expensive.

This is a classic coarse-to-fine architecture:

```text
cheap + broad
      ↓
expensive + precise
```

Retriever = find candidates.

Reranker = rank candidates better.

---

# 12. Context building

The best documents/chunks are formatted for the LLM:

```text
[Employee Handbook]
[Page 17]

Employees receive 20 days...
```

The context can contain:

- chunk text
- document name
- page
- section
- citation ID

This makes grounded generation and citations possible.

---

# 13. Generation

The LLM receives something like:

```text
System instructions
+
retrieved context
+
conversation history
+
user question
```

Then:

```text
LLM
 ↓
answer
```

The model should be instructed to use the retrieved evidence and cite claims.

---

# 14. Long-context / Balanced Context RAG

The case study discusses a move away from blindly searching for a tiny “perfect” chunk.

With very large context windows, a system may retrieve larger document segments:

```text
10K–50K token segment
```

instead of only:

```text
512-token chunk
```

and allow the model to reason over more surrounding context.

But this does **not** mean chunking is obsolete.

The correct mental model is:

> Retrieval granularity should be chosen based on accuracy, context size, latency, and cost.

---

# 15. Output guardrails

After generation:

```text
LLM answer
   ↓
output guardrails
```

Possible checks:

- grounding
- policy compliance
- sensitive information leakage
- citation requirements

If the answer fails:

```text
fallback response
```

---

# 16. Caching

The system can cache answers.

## Exact cache

Same query + same permission context:

```text
"What is the leave policy?"
        ↓
exact cache hit
        ↓
return existing answer
```

This avoids another retrieval + reranking + LLM call.

## Semantic cache

Different wording, same meaning:

```text
"What is the annual leave policy?"

"How many vacation days do employees get?"
```

A semantic cache can recognize them as similar.

The example uses:

```text
threshold = 0.95
TTL = 1800 seconds
```

1800 seconds = 30 minutes.

Important:

> Cache keys must include permission/security context so one user cannot receive another user's restricted answer.

---

# 17. Scaling to 500K documents

The case study proposes a distributed Qdrant setup.

Conceptually:

```text
500K documents
      ↓
multiple shards
      ↓
multiple nodes
```

### Sharding

Distributes data.

### Replication

Keeps copies for availability.

Example:

```text
Shard 1
 ├─ Replica A
 └─ Replica B
```

If one node fails, another replica can serve requests.

---

# 18. Scaling 500 concurrent users

Use horizontal scaling:

```text
Load Balancer
 ├─ Query Service 1
 ├─ Query Service 2
 ├─ Query Service 3
 └─ Query Service 4
```

All instances use shared backend services:

```text
Qdrant cluster
Elasticsearch cluster
LLM API
cache
metadata DB
```

This avoids having a single query-service bottleneck.

---

# 19. LLM reliability

LLM API calls should have:

- timeout
- retry
- exponential backoff
- rate-limit handling
- fallback model

Conceptually:

```text
Primary LLM
    ↓
failure?
    ↓
Fallback LLM
```

---

# 20. Freshness and incremental ingestion

Requirement:

```text
freshness < 1 hour
```

If one document changes:

```text
Document changed
      ↓
Detect change
      ↓
Re-ingest only that document
      ↓
Parse
      ↓
Chunk
      ↓
Embed
      ↓
Upsert vectors
      ↓
Update keyword index
```

Do not reprocess all 500K documents for one change.

This is why production ingestion automation matters.

---

# 21. Long documents and hierarchical chunking

A 100+ page document may need structure:

```text
Document
  ↓
Section
  ↓
Subsection
  ↓
Chunk
```

This can preserve relationships that flat chunking loses.

---

# 22. Important production failure: fan-out writes

After parsing, the ingestion system writes to several places:

```text
             Parsed document
                   │
       ┌───────────┼────────────┐
       ↓           ↓            ↓
    Qdrant      Elastic         S3
       │
       ↓
    Postgres
```

These operations can fail independently.

Example:

```text
Qdrant succeeds
Elasticsearch fails
```

Now semantic search works but keyword search does not.

Therefore production ingestion needs:

- retries
- idempotency
- ingestion status
- reconciliation jobs
- dead-letter handling
- versioning
- observability

The source explicitly highlights this partial-failure problem.

---

# 23. Cost model

The example assumes:

```text
500 users
× 100 queries/user/day
× 30 days
= 1.5M queries/month
```

Illustrative monthly costs in the case study:

| Component | Approx. cost |
|---|---:|
| LLM | $20,250 |
| Embeddings | $200 |
| Reranking | $75 |
| Qdrant | $1,500 |
| Elasticsearch | $2,000 |
| Query compute | $1,000 |
| **Total** | **~$25,000/month** |

These figures are assumptions in the document, not universal/current vendor prices.

The biggest cost is the LLM.

---

# 24. Cost optimization

### Caching

30% cache hit rate can eliminate a significant amount of repeated LLM work.

### Model routing

```text
simple query → cheap model
complex query → powerful model
```

### Batch embeddings

Embed many chunks in batches instead of individual requests.

### Self-host reranker

Use an open-source reranker when infrastructure/compliance makes that sensible.

---

# 25. Lessons from the case study

## What worked

### Hybrid search
Semantic + keyword retrieval improved recall.

### Reranking
Improved top-5 precision.

### Citations
Improved user trust.

### Retrieval-time permissions
Avoided post-hoc security filtering.

## Challenges

### Tables
Complex PDFs require better parsing.

### Acronyms
Domain terminology can hurt retrieval.

### Freshness
Frequent changes require incremental/streaming ingestion.

### Long documents
May require hierarchical chunking.

---

# 26. What the team would do differently

1. Improve document parsing earlier.
2. Build evaluation before scaling.
3. Add query logging from day one.
4. Build a user-feedback loop early.

This gives a useful production principle:

> Do not scale a RAG system before measuring retrieval quality.

---

# 27. Evaluation

A wrong answer does not necessarily mean the LLM is bad.

Debug from the bottom upward:

```text
Wrong answer
   ↑
Did LLM receive correct evidence?
   ↑
Did reranker rank it correctly?
   ↑
Did retrieval find it?
   ↑
Were embeddings good?
   ↑
Was chunking good?
   ↑
Was parsing correct?
```

Useful retrieval metrics:

```text
Recall@K
Precision@K
MRR
NDCG
```

Useful system metrics:

```text
P50/P95/P99 latency
QPS
error rate
availability
cache hit rate
```

---

# 28. Interview walkthrough

A good interview structure:

## Opening

> I will design an enterprise RAG system for internal document search. First I will clarify scale, latency, accuracy, security, document types, and update frequency.

## Requirements

Mention:

- 500K documents
- 500 concurrent users
- <5s P95
- >90% accuracy
- 99.9% availability
- <1h freshness
- RBAC
- citations

## Architecture

Draw:

```text
UI
 ↓
API Gateway
 ↓
Query Service
 ↓
Hybrid Retrieval
 ↓
RRF
 ↓
Reranker
 ↓
LLM
 ↓
Citations
```

Then show:

```text
Qdrant
Elasticsearch
S3
Postgres
```

## Deep dive

Discuss:

1. ingestion
2. hybrid retrieval
3. RRF
4. reranking
5. permission filtering
6. generation
7. citations
8. caching

## Scaling

Discuss:

- sharding
- replicas
- horizontal query-service scaling
- caching
- LLM retries/fallbacks
- incremental ingestion

## Trade-offs

### Accuracy vs latency

Reranking improves accuracy but adds latency.

### Cost vs quality

More powerful LLMs cost more.

### Freshness vs simplicity

Streaming/incremental ingestion is more complex than batch ingestion.

### Managed vs self-hosted

Managed services reduce operational work; self-hosting gives more control.

---

# 29. The architecture you should remember

```text
                       USERS
                         │
                         ↓
                  API Gateway
                Auth + Rate Limit
                         │
                         ↓
                   Query Service
              Permission + Orchestration
                         │
               ┌─────────┴─────────┐
               ↓                   ↓
        Vector Search         Keyword Search
           Qdrant               Elastic
               │                   │
               └─────────┬─────────┘
                         ↓
                        RRF
                         ↓
                     Reranker
                         ↓
                  Context Builder
                         ↓
                        LLM
                         ↓
                  Output Guardrails
                         ↓
                     Citations
                         ↓
                       Answer


                INGESTION PIPELINE

Document
   ↓
Parse
   ↓
Metadata
   ↓
Chunk
   ↓
Embed
   ↓
 ┌─┴───────────┬───────────┐
 ↓             ↓           ↓
Qdrant       Elastic       S3
 ↓
Postgres metadata/ACL
```

---

# 30. Final mental model

> **Ingestion makes enterprise data searchable. Retrieval finds candidate evidence. RRF combines semantic and lexical search. Reranking selects the best evidence. The LLM converts that evidence into an answer. Permissions, guardrails, citations, caching, monitoring, and incremental ingestion turn the basic RAG demo into a production enterprise system.**

## Most important points to remember

1. Separate **ingestion** and **query-time retrieval**.
2. Parsing quality directly affects RAG quality.
3. Chunking is an evaluated engineering choice, not a fixed rule.
4. Embeddings enable semantic search.
5. Qdrant provides vector retrieval.
6. Elasticsearch provides lexical retrieval.
7. Hybrid search combines both.
8. RRF combines result rankings.
9. Reranking improves precision.
10. Apply ACL/permission filters during retrieval.
11. Store original documents separately from vectors.
12. Use metadata for filtering and auditability.
13. Cache repeated queries carefully with permission context.
14. Incremental ingestion keeps the corpus fresh.
15. Production systems need retries, idempotency, reconciliation, monitoring, and evaluation.
16. When RAG fails, investigate parsing → chunking → retrieval → reranking before blaming the LLM.
