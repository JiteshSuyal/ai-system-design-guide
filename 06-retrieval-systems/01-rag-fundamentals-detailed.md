
# RAG Fundamentals — My Notes

> Based on the **RAG Fundamentals** chapter from the AI System Design Guide with additional explanations and interview notes for better understanding.

---

# Table of Contents

1. Why RAG Exists
2. How RAG Works (Complete Pipeline)
3. Core Philosophy: Grounding vs Training
4. RAG Taxonomy
5. RAG vs Long Context (2M Context Window)
6. Retrieval Quality Gap
7. Interview Questions
8. Key Takeaways

---

# 1. Why RAG Exists

## The Problem

LLMs like GPT, Claude, Gemini, etc. only know what they were trained on.

If I ask:

> What is my company's internal leave policy?

The model **cannot answer** because that information was never part of training.

Similarly,

- Company Documents
- HR Policies
- Private PDFs
- Internal APIs
- Customer Tickets
- Financial Reports

are **not present** inside the model.

---

## Wrong Solution: Fine-tune the Model

One option is:

```
Company Documents
        │
        ▼
 Train the LLM again
        │
        ▼
  Updated Model
```

Problems

- Extremely expensive
- Takes days/weeks
- Requires GPUs
- Every document update requires retraining
- Difficult to remove knowledge later

Not practical.

---

## Correct Solution: RAG

Instead of storing knowledge inside model weights,

retrieve the relevant documents **at inference time**.

```
User Question
      │
      ▼
Retrieve Relevant Documents
      │
      ▼
Give Documents to LLM
      │
      ▼
Generate Answer
```

Hence the name

> **Retrieval + Augmented + Generation**

The LLM never memorizes company data.

It simply **reads the relevant documents** before answering.

This process is called **Grounding**.

---

# 2. How RAG Works

A production RAG system has **two pipelines**.

---

## Pipeline 1 — Offline (Indexing)

Runs once when documents are uploaded.

```
PDFs
 │
 ▼
Parse
 │
 ▼
Chunk
 │
 ▼
Generate Embeddings
 │
 ▼
Store in Vector Database
```

---

### Step 1 — Parse

Convert documents into plain text.

Example

```
Employee Handbook.pdf

↓

Raw Text
```

---

### Step 2 — Chunking

Don't embed an entire PDF.

Split it into smaller pieces.

Example

```
500 Page PDF

↓

Chunk 1

Chunk 2

Chunk 3

...
```

Typical chunk size

- 200–1000 tokens

Why?

Because only a small part of the document is usually relevant.

---

### Step 3 — Generate Embeddings

Each chunk becomes a vector.

Example

```
"The company offers 20 annual leaves."

↓

[0.23, -0.45, 0.81, ...]
```

Embeddings capture **meaning**, not keywords.

Example

```
Vacation

Annual Leave

Paid Time Off
```

These are different words but similar meanings, so their vectors are close.

---

### Step 4 — Store in Vector Database

Store

- Embedding
- Original Text
- Metadata

Example

```
Embedding

↓

Chunk Text

↓

Page Number

↓

Source PDF
```

Now the knowledge is searchable.

---

# Pipeline 2 — Online (Query Time)

Runs every time the user asks a question.

```
User Question
 │
 ▼
Generate Query Embedding
 │
 ▼
Vector Search
 │
 ▼
Retrieve Top-K Chunks
 │
 ▼
LLM
 │
 ▼
Answer
```

---

### Step 5 — Embed the User Question

Example

```
How many vacation days?

↓

Embedding
```

---

### Step 6 — Vector Search

Compare the query vector with all document vectors.

Retrieve the closest matches.

Example

```
Chunk A
Similarity = 0.97

Chunk B
Similarity = 0.93

Chunk C
Similarity = 0.82
```

---

### Step 7 — Send Context to LLM

Instead of sending the entire handbook,

send

```
Question

+

Top Retrieved Chunks
```

The LLM answers based on those chunks.

---

## Why RAG Reduces Hallucinations

Without RAG

```
Question

↓

LLM Memory

↓

Guess
```

With RAG

```
Question

↓

Retrieve Facts

↓

LLM

↓

Answer
```

The answer is grounded in real documents.

---

# 3. Core Philosophy — Grounding vs Training

The chapter compares Fine-tuning and RAG.

---

## Knowledge Type

### Fine-tuning

Knowledge becomes part of model weights.

```
Knowledge

↓

Model Weights
```

---

### RAG

Knowledge stays outside the model.

```
Knowledge

↓

Vector Database
```

The model only reads it when needed.

---

## Update Cycle

Fine-tuning

```
Document Changes

↓

Train Again
```

Very expensive.

---

RAG

```
Document Changes

↓

Re-embed

↓

Done
```

No retraining required.

---

## Attribution

Fine-tuned models

```
Answer

↓

No Source
```

---

RAG

```
Answer

↓

Source PDF

↓

Page Number
```

Enterprise systems prefer citations.

---

## Privacy

Suppose someone uploads

```
Salary.xlsx
```

Later asks

```
Delete it.
```

RAG

Delete the document.

Done.

Fine-tuning

Knowledge is inside model weights.

Very difficult to remove.

---

## Golden Interview Rule

> **Fine-tuning is for Form.**
>
> **RAG is for Fact.**

### Fine-tuning Examples

- Writing style
- Tone
- Personality
- Grammar
- Customer support behavior

### RAG Examples

- Company documents
- HR policies
- Stock prices
- Medical PDFs
- Internal APIs

---

# 4. RAG Taxonomy

Modern RAG systems are categorized by **Agentic Depth**.

```
Naive RAG

↓

Advanced RAG

↓

Agentic RAG

↓

GraphRAG
```

---

## 1. Naive RAG

Pipeline

```
User Question

↓

Vector Search

↓

Top-K

↓

LLM
```

Simple.

Easy.

Not production ready.

Problems

- Low precision
- Misses important documents
- Retrieval failures

The chapter explicitly says this is **deprecated for production**.

---

## 2. Advanced RAG

Pipeline

```
Query Rewrite

↓

Hybrid Search

↓

Reranking

↓

LLM
```

### Query Rewrite

User

```
Vacation days
```

System rewrites

```
Annual leave policy
```

Better search.

---

### Hybrid Search

Use

- Semantic Search

AND

- Keyword Search

instead of relying on only one method.

---

### RRF (Reciprocal Rank Fusion)

Vector Search Ranking

```
A

B

C
```

Keyword Search Ranking

```
C

D

A
```

RRF combines both rankings into one stronger ranking.

---

## 3. Agentic RAG

Pipeline

```
Agent

↓

Choose Tool

↓

Retrieve

↓

Enough?

↓

No

↓

Search Again
```

Unlike Advanced RAG,

this is **not a linear pipeline**.

The agent decides

- which database
- which tool
- whether retrieval was sufficient

Examples

- Self-RAG
- Corrective RAG (CRAG)

---

## 4. GraphRAG

Instead of storing isolated chunks,

store entities and relationships.

Example

```
Alice

↓

works at

↓

Google

↓

owns

↓

DeepMind

↓

created

↓

Gemini
```

Useful for

- Relationship discovery
- Cross-document reasoning
- Aggregative questions

Example

> Summarize legal risks across 50 contracts.

Graph traversal performs better than similarity search.

---

# 5. RAG vs Long Context (2M Context)

Modern models support

- 1M tokens
- 2M tokens

Does this kill RAG?

**No.**

---

## In-Context RAG

If your dataset is small

(< 50k tokens)

don't use a Vector DB.

Simply put everything into the prompt.

```
Entire Dataset

↓

Prompt

↓

LLM
```

---

## Prompt Caching

Instead of repeatedly processing the same background context,

cache it.

Benefits

- Lower latency
- Much cheaper inference

---

## Architectural Decision

According to the chapter

### Under 50k Tokens

Use

```
In-Context RAG
```

---

### Over 100k Tokens

Use

```
Standard RAG
```

---

### Cross-document Aggregation

Use

```
GraphRAG
```

---

# 6. Retrieval Quality Gap

The chapter says

> Most RAG failures are retrieval failures, not generation failures.

There are three major gaps.

---

## Gap 1 — Semantic Mismatch

User asks

```
Fast Cars
```

Database contains

```
Porsche 911
```

Different words.

Same meaning.

Solution

- Better embeddings
- Embedding rerankers

---

## Gap 2 — Missing Context

Relevant document exists.

Retriever fails to return it.

Solution

- Hybrid Search

---

## Gap 3 — Lost in the Middle

Correct information is already inside the prompt.

The LLM ignores it because it sits in the middle of a huge context window.

Solution

- Context Compression

---

# 7. Interview Questions

## Q1

Why use RAG when models support 1M–2M tokens?

### Strong Answer

Three reasons

### 1. Cost & Latency

Reading

2M tokens

every request is expensive.

Retrieving

2–5 relevant chunks

is much cheaper.

---

### 2. Freshness

RAG can retrieve

- Stock Prices
- News
- Database Records
- APIs

Long-context prompts cannot magically stay updated.

---

### 3. Scale

Enterprise data

- SharePoint
- Logs
- Wikis
- Internal Docs

often exceeds millions of tokens.

RAG retrieves only the relevant 0.01%.

---

## Q2

Difference between Advanced RAG and Agentic RAG?

### Advanced RAG

```
Rewrite

↓

Search

↓

Rerank

↓

LLM
```

Deterministic pipeline.

---

### Agentic RAG

```
Think

↓

Choose Tool

↓

Retrieve

↓

Evaluate

↓

Retrieve Again

↓

LLM
```

Reasoning loop.

---

# 8. Key Takeaways

✅ Fine-tuning changes model behavior.

✅ RAG supplies external knowledge.

✅ Fine-tuning is for **Form**.

✅ RAG is for **Fact**.

✅ Naive RAG is no longer production-ready.

✅ Advanced RAG (Hybrid + RRF + Reranking) is today's baseline.

✅ Agentic RAG adds reasoning and iterative retrieval.

✅ GraphRAG solves relationship-heavy and aggregative queries.

✅ Long-context models reduce—but do not eliminate—the need for RAG.

✅ Most production failures happen because retrieval failed, not because the LLM generated poorly.
