# Chunking Strategies — My Notes (Part 1)

> Based on **02 - Chunking Strategies** from the AI System Design Guide with additional explanations, production notes, backend analogies, and interview insights.

---

# Table of Contents (Part 1)

1. Why Chunking Exists
2. The Retrieval–Context Tension
3. Why Fixed-size Chunking is a Bad Idea
4. Recursive Structure Splitting
5. Markdown-aware Chunking
6. Contextual Chunking
7. Interview Notes
8. Key Takeaways

---

# 1. Why Chunking Exists

Before understanding chunking, let's understand the problem.

Suppose your company uploads a PDF.

```
Employee Handbook

500 Pages
```

Can we directly generate one embedding for the entire document?

Technically yes.

Practically no.

Why?

Because embeddings represent **meaning**.

A 500-page document talks about

- Leave Policy
- Office Timing
- Holidays
- Insurance
- Salary
- Promotions
- Travel
- Security

Trying to represent all these topics with **one vector** is impossible.

Imagine summarizing an entire 500-page book into **one sentence**.

Most important details disappear.

Therefore,

instead of embedding

```
Entire Document
```

we split it into

```
Chunk 1

Chunk 2

Chunk 3

...

Chunk N
```

Each chunk now represents one meaningful topic.

---

# Why not make extremely tiny chunks?

Suppose we split every document into

```
10 words
```

Example

Original paragraph

```
Employees receive 20 annual paid leaves every year.
Unused leaves can be carried forward.
```

Tiny chunks become

```
Employees receive

20 annual

paid leaves

every year

Unused leaves

can be carried

forward
```

Now ask

> How many annual leaves are given?

The retrieved chunk might be

```
paid leaves
```

That's useless.

The complete meaning disappeared.

---

# Why not make one huge chunk?

Now suppose one chunk contains

```
50 pages
```

Question

```
Annual leave?
```

The LLM receives

```
50 pages

↓

Office Timing

↓

Insurance

↓

Salary

↓

Leave

↓

Travel

↓

Security
```

The important information becomes buried.

Search quality decreases.

Latency increases.

Cost increases.

---

Therefore,

Chunking is a balancing act.

Small enough for accurate retrieval.

Large enough for sufficient context.

This is exactly what the chapter calls

> **The Retrieval–Context Tension.** :contentReference[oaicite:0]{index=0}

---

# 2. The Retrieval–Context Tension

The chapter gives an excellent comparison.

Let's understand every row.

---

## Small Chunks (100 Tokens)

Advantages

```
Very precise
```

Suppose the query is

```
Leave Policy
```

The vector database can find the exact paragraph.

Very high precision.

---

Disadvantages

Context breaks.

Example

Original paragraph

```
Employees receive 20 annual paid leaves.

Unused leaves expire after one year.
```

Chunk

```
Employees receive
20 annual
paid leaves.
```

Second chunk

```
Unused leaves expire
after one year.
```

The second chunk doesn't explain

"What leaves?"

The relationship is broken.

---

## Large Chunks (1000 Tokens)

Advantages

Rich context.

The LLM understands

- previous paragraph
- next paragraph
- definitions
- examples

Reasoning becomes easier.

---

Disadvantages

Retrieval becomes less precise.

Suppose

1000 tokens contain

```
Office Timing

Insurance

Leave Policy

Payroll

Security

Travel

Laptop Policy
```

User asks

```
Leave Policy
```

The embedding now represents **all seven topics**.

Similarity search becomes weaker.

---

# Understanding the Trade-off

The chapter summarizes it beautifully.

> Smaller is better for **finding**.

> Larger is better for **thinking**. :contentReference[oaicite:1]{index=1}

Let's understand why.

---

Vector DB

Only performs

```
Finding
```

The LLM performs

```
Reasoning
```

Finding requires

```
Specific vectors
```

Reasoning requires

```
Rich context
```

These two goals conflict.

This conflict is called

```
Retrieval–Context Tension
```

---

# Backend Analogy

Imagine searching logs.

Would you rather search

```
Entire server logs

(2 GB)
```

or

```
Individual log entries
```

Individual log entries.

Finding becomes easier.

But after finding the log,

you still need surrounding logs

to understand what happened.

Exactly the same thing happens in RAG.

---

# Production Insight

The chapter gives an important rule.

> Small chunks are better for retrieval.

> Large chunks are better for reasoning.

Modern production systems solve this using

```
Hierarchical Chunking
```

We'll study that in Part 2.

---

# 3. Why Fixed-size Chunking is Bad

Early RAG systems used

```
Every 500 characters
```

Example

```
Employees receive 20 annual paid leaves every year.

Unused leaves...
```

might become

```
Employees receive 20 annual paid lea
```

Second chunk

```
ves every year.

Unused leaves...
```

This is terrible.

Why?

Because

- sentences break
- ideas break
- equations break
- code breaks
- tables break

The embedding no longer represents one complete idea.

The chapter calls this

> **Content-Blind Chunking.** :contentReference[oaicite:2]{index=2}

---

# Overlap

Older systems tried

```
Chunk 1

AAAA BBBB CCCC DDDD

Chunk 2

CCCC DDDD EEEE FFFF
```

Notice

```
CCCC DDDD
```

appears twice.

This is called

```
Overlap
```

Usually

```
10%
```

or

```
20%
```

---

## Why overlap?

Suppose

the sentence gets split.

Overlap ensures

part of the sentence appears in both chunks.

Better than nothing.

---

## Why isn't overlap enough?

Because the real problem still exists.

The document was split randomly.

Overlap only duplicates text.

It doesn't understand

meaning.

---

# 4. Recursive Structure Splitting

The chapter recommends

Logical Splitting.

Instead of

```
500 Characters
```

split according to

document structure.

Priority

```
Double Newline

↓

Single Newline

↓

Period

↓

Space
```

Exactly as described in the chapter. :contentReference[oaicite:3]{index=3}

---

Example

Original document

```
# Leave Policy

Employees receive 20 annual leaves.

Unused leaves expire after one year.

# Insurance

Medical insurance...
```

Recursive splitter first tries

```
Header
```

Then

```
Paragraph
```

Then

```
Sentence
```

Only if absolutely necessary

```
Words
```

This preserves meaning.

---

# Why is Recursive Splitting Better?

Because documents already contain

human-written structure.

Examples

```
Headers

Paragraphs

Bullet Lists

Tables

Code Blocks
```

Destroying that structure

reduces retrieval quality.

---

# 5. Markdown-aware Chunking

The chapter specifically recommends

Markdown-aware splitting. :contentReference[oaicite:4]{index=4}

Suppose

```
# Insurance

Medical insurance covers...

Hospitalization...

Dental...
```

If the chunk only contains

```
Hospitalization...

Dental...
```

The LLM doesn't know

what topic this belongs to.

Instead,

prepend

```
Insurance

↓

Hospitalization...

Dental...
```

Every child chunk keeps its parent heading.

This preserves context.

---

# Backend Analogy

Imagine a REST API response.

Without endpoint name

```
status

200
```

Meaningless.

With endpoint

```
GET /users

↓

status 200
```

Now the response has context.

Markdown headers work exactly like endpoint names.

---

# 6. Contextual Chunking

The chapter briefly introduces

Contextual Chunking. :contentReference[oaicite:5]{index=5}

Idea

Every chunk should carry

its parent context.

Example

Instead of storing

```
Battery lasts 12 hours.
```

Store

```
MacBook Air M4 Manual

↓

Battery lasts 12 hours.
```

Now

the embedding knows

this battery belongs to

MacBook,

not

Phone,

Drone,

or

Power Bank.

Retrieval quality improves significantly.

We'll revisit this idea in a later chapter on Contextual Retrieval.

---

# Interview Notes

### Q. Why don't production systems use fixed-size chunking?

Strong Answer

Fixed-size chunking is content-blind. It frequently splits sentences, headings, equations, tables, and code in unnatural places. Although overlap reduces some of the damage, it cannot preserve semantic boundaries. Modern production systems prefer logical or semantic chunking because each chunk represents a complete idea, improving retrieval precision and downstream LLM reasoning.

---

### Q. Explain Retrieval–Context Tension.

Strong Answer

Small chunks improve retrieval precision because each embedding represents a focused topic. Large chunks improve reasoning because they provide richer surrounding context to the LLM. Production systems balance these competing goals using techniques like Hierarchical Chunking.

---

# Key Takeaways

✅ Chunking exists because one embedding cannot accurately represent an entire document.

✅ Tiny chunks improve retrieval but lose context.

✅ Huge chunks preserve context but hurt retrieval.

✅ Fixed-size chunking is content-blind.

✅ Overlap is only a partial solution.

✅ Recursive splitting follows the natural structure of the document.

✅ Markdown-aware chunking preserves section headers.

✅ Contextual chunking attaches parent information to improve retrieval accuracy.


# Chunking Strategies — My Notes (Part 2)

> Based on **02 - Chunking Strategies** from the AI System Design Guide with additional explanations, backend analogies, production insights, and interview notes.

---

# Table of Contents (Part 2)

1. Semantic Chunking
2. Cross-Encoder Segmenters
3. Hierarchical (Parent-Child) Chunking
4. Why Parent-Child Retrieval is the Industry Standard
5. Production Architecture
6. Backend Analogies
7. Interview Questions
8. Key Takeaways

---

# 1. Semantic Chunking

The repository introduces **Semantic Chunking** after Recursive Splitting because even document structure isn't always enough. :contentReference[oaicite:0]{index=0}

Imagine this document:

```
Artificial Intelligence

AI is changing healthcare.

Hospitals are using AI to detect cancer.

-------------------------------------------------

Football World Cup

Argentina won the championship.
```

Notice that there is no heading.

A recursive splitter may keep everything together because there isn't an obvious structural boundary.

Semantically, however,

the topic clearly changes from

```
Healthcare

↓

Sports
```

This is where Semantic Chunking comes in.

---

# How Semantic Chunking Works

Instead of looking at

- Paragraphs
- Headers
- Line breaks

it looks at

```
Meaning
```

Pipeline

```
Document

↓

Split into Sentences

↓

Generate Embeddings

↓

Compare Similarity

↓

Group Similar Sentences
```

Exactly as described in the chapter. :contentReference[oaicite:1]{index=1}

---

# Example

Suppose we have

Sentence 1

```
Employees receive
20 annual paid leaves.
```

Sentence 2

```
Unused leaves
expire after one year.
```

Sentence 3

```
Medical insurance
covers hospitalization.
```

---

Embedding Similarity

```
S1 ↔ S2

0.95
```

Very similar.

Same topic.

---

```
S2 ↔ S3

0.34
```

Different topic.

Create a new chunk.

Final chunks become

```
Chunk 1

Leave Policy
```

```
Chunk 2

Insurance
```

Notice

No fixed size.

No fixed number of sentences.

Only topic boundaries matter.

---

# Why is this better?

Because embeddings represent

```
Meaning
```

instead of

```
Formatting
```

Even if the document has poor formatting,

Semantic Chunking still finds logical boundaries.

---

# Production Analogy

Imagine organizing your music library.

Would you group songs by

```
Exactly 100 songs per folder?
```

No.

You naturally group them by

```
Rock

Pop

Jazz

Classical
```

Semantic Chunking works the same way.

---

# Threshold

The chapter mentions

```
0.82 similarity
```

as an example threshold. :contentReference[oaicite:2]{index=2}

Meaning

```
Similarity > 0.82

↓

Same Chunk
```

Otherwise

```
Start New Chunk
```

Different applications may choose different thresholds.

---

# Limitation

Semantic Chunking requires

```
Embedding

↓

Similarity Calculation

↓

Decision
```

for every sentence.

This makes indexing slower than simple recursive splitting.

However,

indexing usually happens once,

while retrieval happens thousands of times.

Most production systems accept this trade-off.

---

# 2. Cross-Encoder Segmenters

This is one of the most interesting points in the chapter.

The author says that modern systems increasingly use

```
Cross-Encoder Segmenters
```

instead of simple cosine similarity thresholds. :contentReference[oaicite:3]{index=3}

---

# Traditional Semantic Chunking

Pipeline

```
Sentence

↓

Embedding

↓

Cosine Similarity

↓

Threshold
```

Problem

Cosine similarity is only a heuristic.

Sometimes

```
Topic changed

↓

Similarity still high
```

or

```
Same topic

↓

Similarity unexpectedly low
```

Mistakes happen.

---

# Cross-Encoder

Instead of comparing vectors,

use another AI model.

Pipeline

```
Sentence A

+

Sentence B

↓

Cross Encoder

↓

Should these stay together?

YES / NO
```

Think of it as an AI reviewer.

---

# Why is it more accurate?

Embeddings are generated independently.

Cross Encoders read

both sentences

together.

Example

```
Sentence A

Employees receive
20 annual leaves.

Sentence B

Unused leaves expire.
```

The Cross Encoder understands

these belong together.

---

# Repository Insight

The chapter says

these models predict

```
Separator Token
```

at semantic boundaries. :contentReference[oaicite:4]{index=4}

Instead of

```
Similarity = 0.82
```

they directly answer

```
Split Here
```

This is significantly more accurate.

---

# Backend Analogy

Imagine Git.

Cosine Similarity

```
Compare two files.

Guess.

```

Cross Encoder

```
Code Reviewer

↓

These changes belong together.

↓

These don't.
```

Much smarter.

---

# 3. Hierarchical (Parent-Child) Chunking

This is the most important section in the chapter.

The author calls it

> **The Industry Standard for Production RAG.** :contentReference[oaicite:5]{index=5}

If you remember only one chunking strategy,

remember this one.

---

# The Problem

Small chunks

↓

Easy to retrieve

Hard to reason

Large chunks

↓

Hard to retrieve

Easy to reason

Can we have both?

Yes.

Hierarchical Chunking.

---

# Parent-Child Architecture

Suppose we have

```
1500 Tokens
```

Instead of storing one huge chunk,

split it into

```
Parent

1500 Tokens
```

↓

```
Child 1

300 Tokens

Child 2

300 Tokens

Child 3

300 Tokens

Child 4

300 Tokens

Child 5

300 Tokens
```

Exactly as described in the repository. :contentReference[oaicite:6]{index=6}

---

# Which chunks are embedded?

This is the clever part.

The chapter says

```
Index ONLY the children.
```

Not the parent.

Why?

Small chunks produce

more precise embeddings.

---

# Retrieval

Suppose user asks

```
How many annual leaves?
```

Vector Search returns

```
Child 3
```

Not

Parent.

---

Now the system says

```
Child belongs to

Parent 1
```

Instead of sending

300 tokens

to GPT,

it sends

```
Entire Parent

1500 Tokens
```

Now GPT has enough context.

---

# Visual Pipeline

```
PDF

↓

Parent (1500 Tokens)

↓

Split

↓

Child Chunks (300)

↓

Embed Children

↓

Vector DB
```

Later

```
Question

↓

Retrieve Child

↓

Lookup Parent

↓

Send Parent to GPT
```

This architecture gives us

```
High Precision

+

Rich Context
```

Best of both worlds.

---

# Why does this work so well?

Suppose Child says

```
Unused leaves expire
after one year.
```

GPT asks

```
Unused leaves?

Which leaves?
```

Parent contains

```
Employees receive
20 annual paid leaves.

Unused leaves expire
after one year.
```

Now everything makes sense.

---

# Production Architecture

Real systems often look like this

```
PDF

↓

Parser

↓

Parent Generator

↓

Child Generator

↓

Embeddings

↓

Vector Database

↓

Question

↓

Child Retrieval

↓

Parent Lookup

↓

LLM
```

Notice

The Vector DB never returns

the final prompt.

It returns

an identifier.

Another service fetches

the parent chunk.

---

# Backend Analogy

Imagine YouTube.

Search

```
Specific Timestamp
```

↓

Result

```
12:43
```

Would YouTube play

```
Only one second?
```

No.

It opens

the whole video

at that timestamp.

Child

↓

Timestamp

Parent

↓

Entire video

Exactly the same idea.

---

# Why is this considered Industry Standard?

Because it solves

both problems simultaneously.

| Retrieval | Reasoning |
|-----------|-----------|
| Child | Parent |

The vector DB finds

small chunks.

The LLM receives

large chunks.

Everyone wins.

---

# Trade-offs

Advantages

✅ Better retrieval precision

✅ Rich context

✅ Fewer hallucinations

✅ Better reasoning

Disadvantages

❌ More metadata

❌ Parent-child mapping

❌ Slightly more complex pipeline

Almost every modern RAG framework supports this pattern because the quality improvement is worth the additional complexity.

---

# Interview Questions

## Q1

Why doesn't production RAG simply embed large chunks?

Strong Answer

Large chunks dilute semantic meaning because they combine many unrelated topics into one embedding. This reduces retrieval precision. Production systems therefore retrieve using small child chunks while sending larger parent chunks to the LLM for reasoning.

---

## Q2

Why is Hierarchical Chunking considered the industry standard?

Strong Answer

It balances the retrieval-context trade-off. Small child chunks provide highly accurate vector search, while large parent chunks preserve enough surrounding context for the LLM to reason correctly. This achieves better retrieval precision without sacrificing answer quality.

---

## Q3

Why are only child chunks indexed?

Strong Answer

Child chunks represent focused semantic units, producing more accurate embeddings. Parent chunks are intentionally not embedded because they mix multiple topics and reduce vector search precision. Parents are retrieved only after a matching child has been found.

---

# Key Takeaways

✅ Semantic Chunking groups text by meaning instead of formatting.

✅ Embedding similarity determines whether sentences stay together.

✅ Cross Encoders are more accurate than cosine-threshold approaches because they evaluate sentence pairs jointly.

✅ Parent-Child Chunking is the current production standard.

✅ Only child chunks are embedded and indexed.

✅ Parent chunks are retrieved after a child match to provide complete context.

✅ This architecture achieves both high retrieval precision and rich reasoning context.

# Chunking Strategies — My Notes (Part 3)

> Based on **02 - Chunking Strategies** from the AI System Design Guide with additional explanations, production insights, backend analogies, and interview notes.

---

# Table of Contents (Part 3)

1. Content-Specific Chunking
2. Code Chunking
3. Table Chunking
4. PDF/Layout Chunking
5. Contextual Retrieval (Anthropic Pattern)
6. Production Pipeline
7. Choosing the Right Chunking Strategy
8. Interview Questions
9. Complete Cheat Sheet
10. Final Takeaways

---

# 1. Content-Specific Chunking

The repository introduces an important production concept:

> **Not every document should be chunked the same way.** :contentReference[oaicite:0]{index=0}

Many beginners think

```
One Chunking Algorithm

↓

Everything
```

Production systems don't work like this.

Different document types require different chunking strategies.

Examples

| Document | Strategy |
|-----------|----------|
| Plain Text | Semantic |
| Markdown | Recursive |
| Source Code | AST |
| Tables | Table-aware |
| PDFs | Layout-aware |

Why?

Because every document stores information differently.

---

# 2. Code Chunking

The chapter recommends

> **AST (Abstract Syntax Tree) Parsing**. :contentReference[oaicite:1]{index=1}

This is one of the most common interview questions.

---

# Why not split code every 500 tokens?

Suppose we have

```javascript
function calculateTax(price){

   let tax = price * 0.18;

   return price + tax;

}
```

Imagine fixed-size chunking.

Chunk 1

```javascript
function calculateTax(price){

   let tax
```

Chunk 2

```javascript
= price * 0.18;

return...
```

The function is destroyed.

The embedding now represents

half a function.

Terrible retrieval.

---

# AST (Abstract Syntax Tree)

Instead,

parse code into its syntax tree.

```
File

↓

Class

↓

Function

↓

Statements
```

Now every function becomes

one chunk.

Example

```
UserController

↓

createUser()

↓

login()

↓

logout()

↓

deleteUser()
```

Each function becomes one semantic unit.

---

# Why AST?

Functions already represent

one logical task.

Examples

```
authenticate()

↓

uploadImage()

↓

calculateTax()

↓

sendEmail()
```

Much better than

```
500 Characters
```

---

# Repository Rule

The chapter explicitly says

> Never split a function in the middle.

Also keep

```
Imports

+

Class Definition

+

Methods
```

together whenever possible. :contentReference[oaicite:2]{index=2}

---

# Backend Analogy

Imagine GitHub.

Would you search

```
Random 300 Characters
```

or

```
Function Names
```

Every developer naturally thinks

in

```
Functions

Classes

Modules
```

Chunking should do the same.

---

# Production Example

Suppose user asks

```
How is JWT verified?
```

AST retrieval immediately finds

```
verifyJWT()

↓

authenticate()

↓

middleware.js
```

instead of random pieces of code.

---

# 3. Table Chunking

Tables are another difficult problem.

Example

| Year | Revenue |
|------|----------|
|2024|5M|
|2025|8M|

Traditional chunking destroys them.

Example

```
Year Revenue

2024

5M

2025
```

The relationship disappears.

---

# Repository Recommendation

Keep tables in

Markdown format. :contentReference[oaicite:3]{index=3}

Example

```
|Year|Revenue|

|2024|5M|

|2025|8M|
```

The structure remains.

---

# Modern Production Pattern

The chapter introduces

**Summarized Tables.** :contentReference[oaicite:4]{index=4}

This is brilliant.

Instead of embedding

the entire table,

store

```
Summary

+

Original Table
```

Example

Embedding

```
Annual revenue increased from
5M to 8M.
```

Store alongside

```
Full Markdown Table
```

Now

Search becomes easier

while GPT still receives

the complete table.

---

# Backend Analogy

Think of SQL.

Instead of searching

```
Entire table
```

you search

```
Indexed columns
```

Then fetch

the entire row.

Exactly the same principle.

---

# 4. PDF/Layout Chunking

This is one of the newest ideas.

Traditional PDF parsers read

```
Top

↓

Bottom

↓

Left

↓

Right
```

Problem

PDFs are visual.

Example

```
Chart

Sidebar

Image

Main Text
```

Plain text extraction mixes everything.

---

# Repository Recommendation

Use

**Vision Language Models (VLMs)**

such as

**ColPali**. :contentReference[oaicite:5]{index=5}

---

# What is ColPali?

The repository only mentions ColPali as an example.

It does **not** explain its internals.

The important takeaway is

Instead of embedding only text,

embed

```
Text

+

Layout

+

Position
```

This preserves

```
Charts

Tables

Sidebars

Headers
```

---

# Why does layout matter?

Suppose a PDF contains

```
Chart

↓

Explanation

↓

Footnote
```

Normal parsing may produce

```
Chart

Footnote

Explanation
```

Meaning gets destroyed.

Layout-aware chunking prevents this.

---

# Production Example

Suppose the user asks

```
Explain the sales chart on page 18.
```

Layout-aware retrieval knows

```
Chart

+

Caption

+

Nearby explanation
```

belong together.

A text-only parser may fail.

---

# 5. Contextual Retrieval (Anthropic Pattern)

This is one of the most useful ideas in the chapter.

The repository calls it

**Contextual Retrieval.** :contentReference[oaicite:6]{index=6}

---

# The Problem

Suppose one chunk says

```
Battery lasts
12 hours.
```

Question

Battery of what?

Phone?

Drone?

Laptop?

Car?

Impossible to know.

---

# Anthropic's Idea

Before embedding,

prepend

global context.

Instead of storing

```
Battery lasts
12 hours.
```

Store

```
[MacBook Air M4 Manual]

Battery lasts
12 hours.
```

Now

the embedding represents

```
MacBook Battery
```

instead of

generic battery.

---

# Why does this help?

Suppose someone searches

```
Phone battery
```

The MacBook chunk won't be retrieved.

Semantic confusion decreases.

Retrieval precision increases.

---

# Backend Analogy

Imagine logging.

Bad log

```
Connection failed
```

Good log

```
PaymentService

Connection failed
```

Adding context makes searching much easier.

Contextual Retrieval follows exactly the same idea.

---

# 6. Production Chunking Pipeline

A real production ingestion pipeline often looks like this.

```
Document Upload

↓

Parser

↓

Document Type Detection

↓

Choose Chunking Strategy

↓

Generate Chunks

↓

Context Injection

↓

Embeddings

↓

Vector Database
```

Notice

Chunking is no longer

one algorithm.

It becomes

a pipeline of decisions.

---

# Example

Suppose a company uploads

```
README.md

↓

Recursive Splitter
```

Upload

```
Node.js Source Code

↓

AST Chunker
```

Upload

```
Financial PDF

↓

Layout-aware Chunker
```

Every document takes

its own route.

---

# 7. Choosing the Right Strategy

| Scenario | Best Strategy |
|-----------|---------------|
| Markdown | Recursive |
| Long Reports | Semantic |
| Production RAG | Parent-Child |
| Source Code | AST |
| Financial Tables | Table-aware |
| Scientific PDFs | Layout-aware |
| Enterprise Manuals | Contextual Retrieval |

This is a practical interview summary based on the chapter's recommendations. :contentReference[oaicite:7]{index=7}

---

# Interview Questions

## Q1

Why shouldn't source code be chunked like plain text?

Strong Answer

Source code has syntactic structure. Functions, classes, imports, and methods represent complete semantic units. Splitting code arbitrarily destroys logical boundaries and reduces retrieval quality. AST parsing preserves the language structure.

---

## Q2

Why summarize tables before embedding?

Strong Answer

Embedding models struggle to capture the meaning of large tables directly. A natural-language summary improves semantic retrieval, while the full table is still returned to the LLM for accurate reasoning.

---

## Q3

Why is layout-aware chunking important?

Strong Answer

Many PDFs contain charts, sidebars, captions, and multi-column layouts. Plain text extraction often destroys these relationships. Layout-aware models preserve positional information, improving retrieval accuracy.

---

## Q4

Explain Contextual Retrieval.

Strong Answer

Contextual Retrieval prepends global document information before generating embeddings. This allows embeddings to capture both the local content and the broader document context, significantly reducing ambiguous retrievals.

---

# Complete Cheat Sheet

```
Chunking

↓

Goal

Split documents into meaningful retrieval units

-------------------------------------------------

Recursive

↓

Uses document structure

-------------------------------------------------

Semantic

↓

Uses embedding similarity

-------------------------------------------------

Cross Encoder

↓

Predicts semantic boundaries

-------------------------------------------------

Parent-Child

↓

Embed child

Return parent

-------------------------------------------------

AST

↓

Functions

Classes

Imports

-------------------------------------------------

Table

↓

Summary + Original Table

-------------------------------------------------

PDF

↓

Layout-aware

VLM

-------------------------------------------------

Contextual Retrieval

↓

Add parent context

before embedding
```

---

# Final Takeaways

✅ Chunking is one of the biggest factors affecting RAG quality.

✅ There is no universal chunking algorithm.

✅ Recursive chunking preserves document structure.

✅ Semantic chunking preserves meaning.

✅ Cross Encoders detect semantic boundaries better than simple cosine similarity.

✅ Parent-Child Chunking is the production standard.

✅ AST parsing is the correct strategy for source code.

✅ Tables should usually be summarized for embedding while preserving the original structure.

✅ PDFs benefit from layout-aware retrieval using Vision Language Models.

✅ Contextual Retrieval improves embeddings by adding document-level context before embedding.

---

# My Personal Revision Notes

If an interviewer asks me,

> "How would you design chunking for a production RAG system?"

My answer should be:

1. Detect document type.
2. Choose an appropriate chunking strategy (Recursive, AST, Table-aware, Layout-aware, etc.).
3. Preserve semantic boundaries instead of using fixed-size chunks.
4. Use Parent-Child Chunking to balance retrieval precision and reasoning context.
5. Add contextual information before embedding where appropriate.
6. Generate embeddings.
7. Store chunks and metadata in a vector database.
8. Retrieve child chunks but provide parent context to the LLM.
9. Continuously evaluate retrieval quality and adjust chunking strategies based on the data.

This is the architecture used by many modern production RAG systems and is exactly the level of understanding expected in AI Engineer interviews.
