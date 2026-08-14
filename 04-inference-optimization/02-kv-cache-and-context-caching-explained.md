# 02 — KV Cache and Context Caching: Detailed Notes

> These notes explain the attached `02-kv-cache-and-context-caching.md` in detail, following its structure and terminology. The numerical examples and provider/pricing claims below are reproduced from the source document; they are not independently verified here.

---

## 1. The Big Picture

The chapter starts with a very important observation:

> **The KV Cache is one of the most significant memory consumers in long-context AI systems.**

As context length grows, the amount of KV-cache data grows as well.

So an LLM serving system has to solve two related problems:

1. **How large is the KV Cache?**
2. **How can we avoid repeatedly storing or processing the same context?**

The chapter approaches this through:

```text
KV Cache
   ↓
GQA
   ↓
Smaller KV Cache

Context Caching
   ↓
Reuse already-computed context

RAD-O
   ↓
Compress long-context representations
```

---

# 2. The KV Cache Problem

During autoregressive generation, the model needs the **Key (K)** and **Value (V)** tensors corresponding to previous tokens.

Instead of recomputing those K/V representations for every generated token, the model stores them in the **KV Cache**.

Conceptually:

```text
Prompt tokens
     ↓
Compute K and V
     ↓
Store in KV Cache
     ↓
Generate next token
     ↓
Reuse previous K/V
     ↓
Generate next token
     ↓
Reuse previous K/V
     ↓
...
```

The benefit is reduced recomputation.

The cost is memory.

For long contexts and many simultaneous users, KV Cache can consume a very large amount of GPU memory.

---

# 3. Why Does KV Cache Grow?

For each previous token, the model stores Key and Value information across its transformer layers and KV heads.

A simplified relationship is:

```text
KV Cache size
∝
Number of layers
×
Number of KV heads
×
Head dimension
×
Number of tokens
×
2
×
Bytes per value
```

The `2` appears because we store:

```text
K + V
```

So increasing the context length directly increases KV-cache memory.

For example:

```text
10k tokens
   ↓
larger KV Cache

128k tokens
   ↓
much larger KV Cache

1M tokens
   ↓
extremely large KV Cache
```

This is why long-context inference creates a memory-management challenge.

---

# 4. The Source's Llama 4 70B Example

The document gives this calculation:

```text
Tokens       = 128,000
Precision    = BF16
Bytes/value  = 2
Layers       = 80
KV heads     = 8
Head dim     = 128
```

The formula given is:

```text
2 (K + V)
×
80 layers
×
128k tokens
×
8 KV heads
×
128 head dimension
×
2 bytes
```

Let's calculate it:

```text
2 × 80 × 128000 × 8 × 128 × 2
=
41,943,040,000 bytes
```

Approximately:

```text
41.94 GB
```

using decimal GB.

So the document rounds this to:

> **~42 GB per user at 128k context.**

### Important intuition

This is **per user**.

If the same memory requirement applied to 10 simultaneous users:

```text
~42 GB × 10
≈ 420 GB
```

That illustrates why KV-cache optimization is critical for serving long-context requests.

---

# 5. What Does "BF16 = 2 Bytes" Mean?

BF16 is a 16-bit floating-point representation.

```text
16 bits
÷
8 bits/byte
=
2 bytes
```

Therefore every stored K/V value requires approximately:

```text
2 bytes
```

in BF16.

For the source's calculation:

```text
Number of K/V values
×
2 bytes
```

gives the memory requirement.

---

# 6. Why Are There Two KV Components?

The attention mechanism uses:

```text
Query (Q)
Key   (K)
Value (V)
```

The KV Cache stores:

```text
K
+
V
```

It does not cache Q in the same way because Q is generated for the current token during each decode step.

Therefore the memory calculation contains:

```text
2 × ...
```

for:

```text
K + V
```

This is the same KV-cache concept you learned earlier.

---

# 7. Why Context Length Is So Important

Suppose everything else stays constant.

If:

```text
128k tokens → ~42 GB
```

then approximately:

```text
64k tokens → ~21 GB
256k tokens → ~84 GB
512k tokens → ~168 GB
```

The exact real-world result depends on architecture and implementation, but the key relationship is:

> **KV-cache memory grows approximately linearly with sequence length.**

Therefore:

```text
Context length ↑
        ↓
KV Cache ↑
        ↓
VRAM requirement ↑
```

This is one of the most important things to remember.

---

# 8. GQA — Grouped Query Attention

The chapter next introduces **GQA (Grouped Query Attention)**.

Its purpose is:

> **Reduce KV-cache size without requiring every Query head to have its own separate KV heads.**

The basic idea is:

```text
Many Query heads
       ↓
Fewer KV heads
```

Multiple Query heads can share the same K/V representations.

---

# 9. MHA vs GQA vs MQA

The document gives this comparison:

| Method | Ratio | KV Cache Reduction | Quality Loss |
|---|---:|---:|---:|
| MHA | 1:1 | 1× baseline | 0% |
| GQA | 8:1 | 8× | < 0.2% |
| MQA | All:1 | 64×–128× | 2–3% |

The key architectural difference is the number of KV heads.

---

# 10. MHA — Multi-Head Attention

In traditional Multi-Head Attention:

```text
Q1 → K1 → V1
Q2 → K2 → V2
Q3 → K3 → V3
...
Q64 → K64 → V64
```

If there are 64 Query heads, there can be 64 KV heads.

Therefore:

```text
64 Q heads
64 K heads
64 V heads
```

This gives maximum KV-head capacity among the three designs in the table.

But it also means a larger KV Cache.

---

# 11. GQA — Grouped Query Attention

Suppose we have:

```text
64 Query heads
8 KV heads
```

Then:

```text
64 / 8 = 8
```

So every KV head can serve a group of 8 Query heads.

Conceptually:

```text
Q1  ─┐
Q2  ─┤
Q3  ─┤
Q4  ─┤
Q5  ─┤──→ KV1
Q6  ─┤
Q7  ─┤
Q8  ─┘


Q9  ─┐
Q10 ─┤
...
Q16 ─┘──→ KV2
```

and so on.

Therefore:

```text
64 Query heads
        ↓
8 KV heads
```

This is the `8:1` grouping described in the document.

---

# 12. Why Does GQA Reduce KV Cache?

The KV Cache depends on the number of KV heads.

Compare:

```text
MHA:
64 KV heads
```

with:

```text
GQA:
8 KV heads
```

Everything else being equal:

```text
64 / 8
=
8
```

So the KV-cache portion associated with the heads can be approximately:

```text
8× smaller
```

This is why the document describes GQA as providing an **8× KV Cache reduction** for the example ratio.

---

# 13. MQA — Multi-Query Attention

MQA takes the idea even further.

Instead of:

```text
64 Query heads
8 KV heads
```

it can use:

```text
64 Query heads
1 KV head
```

Conceptually:

```text
Q1 ─┐
Q2 ─┤
Q3 ─┤
... ├──→ Shared K/V
Q64─┘
```

This makes the KV Cache extremely small.

But the source document notes a larger quality trade-off than GQA.

---

# 14. Why Not Always Use One KV Head?

This connects to the question you asked earlier.

It might seem:

> "If one KV head saves so much memory, why not always use MQA?"

Because KV heads are not just memory storage.

They are also part of the model's attention architecture.

With many KV heads:

```text
More independent K/V representations
        ↓
More representational flexibility
```

With one KV head:

```text
Much smaller KV memory
        ↓
Less independent K/V information
```

So there is a trade-off.

The source presents the simplified relationship:

```text
MHA
→ maximum KV capacity

GQA
→ large memory savings with small quality impact

MQA
→ maximum KV savings but larger quality trade-off
```

---

# 15. The Important GQA Nuance

The source says:

> GQA allows the model to attend to the same KV "memory" from multiple "reasoning" heads.

Think of:

```text
Many Query heads
       ↓
Shared KV groups
```

The Query heads can still represent different attention patterns, while several of them reuse the same K/V representations.

This is why GQA can reduce KV-cache memory without simply collapsing the entire attention mechanism into one head.

---

# 16. GQA and Decode Memory Bandwidth

The source makes another important point:

> GQA drastically reduces the memory bandwidth needed during Decode.

Why?

During Decode, the model repeatedly reads the KV Cache.

If the KV Cache is huge:

```text
Large KV Cache
     ↓
More data to read
     ↓
More memory bandwidth required
```

GQA reduces the amount of KV data:

```text
Fewer KV heads
     ↓
Smaller KV Cache
     ↓
Less data to read
     ↓
Lower memory bandwidth requirement
```

So GQA helps in two connected ways:

```text
KV Cache size ↓
Memory bandwidth requirement ↓
```

---

# 17. Context Caching

The next major topic is **Context Caching**.

The idea is:

> If many requests use the same context or prefix, don't repeatedly recompute/store everything independently.

For example, imagine a large shared knowledge base:

```text
100-page knowledge base
```

being used by:

```text
1000 users
```

If every request repeatedly processes the same content from scratch, that is wasteful.

Instead, a system can cache the computed representation/KV state for the shared prefix and reuse it.

---

# 18. Shared KV Cache

Conceptually:

```text
                Shared Context
                      |
               Shared KV Cache
                /     |     \
              U1      U2      U3
              |       |       |
           Unique   Unique   Unique
           context  context  context
```

The common part is computed once and reused.

This is particularly useful when many requests have:

- the same system prompt
- the same large document
- the same tool definitions
- the same instructions
- the same knowledge base prefix

---

# 19. Why Context Caching Helps

Without caching:

```text
Request 1
   ↓
Process common context

Request 2
   ↓
Process same common context again

Request 3
   ↓
Process same common context again
```

With caching:

```text
Common context
      ↓
Compute once
      ↓
Cache
      ↓
Reuse for Request 1
Reuse for Request 2
Reuse for Request 3
...
```

This can reduce repeated computation and, depending on the implementation/provider, reduce cost.

---

# 20. VRAM vs Disk/SSD Cache

The document distinguishes cache tiers.

### VRAM Cache

```text
Very fast
+
Limited capacity
```

This is useful for active/recent contexts.

### Disk/SSD Cache

```text
Slower
+
Much larger capacity
```

The document describes a tiered model used by frameworks such as SGLang:

```text
Most Recent
     ↓
VRAM

Frequent
     ↓
HBM

Occasional
     ↓
SSD
```

The important idea is:

> Put frequently needed data in faster memory and less frequently needed data in slower but larger storage.

---

# 21. Why Not Keep Everything in VRAM?

Because GPU memory is expensive and limited.

Suppose you have:

```text
Many users
+
Long contexts
```

Keeping every cached context in VRAM could consume enormous amounts of memory.

A tiered cache can instead look like:

```text
Hot data
  ↓
GPU memory

Warm data
  ↓
HBM / faster tier

Cold data
  ↓
SSD
```

When cold data is needed again, it can be promoted back into faster storage.

---

# 22. API-Level Context Caching

The chapter then moves from self-hosted systems to managed APIs.

Major model providers can offer **Prompt Caching / Context Caching**.

The basic idea is similar:

```text
Repeated prefix
      ↓
Cache it
      ↓
Reuse it
      ↓
Lower repeated input cost / processing
```

The attached document lists examples from:

- Anthropic
- OpenAI
- Google
- DeepSeek

The exact pricing in the source is provider/model/date-specific and should be rechecked before using it for a current cost decision.

---

# 23. Prompt Caching Example

Imagine a chatbot where every request contains:

```text
System instructions
+
Tool definitions
+
Large company policy
+
User question
```

Most of the request may be identical.

Without prompt caching:

```text
Request 1 → process full prefix
Request 2 → process full prefix
Request 3 → process full prefix
```

With caching:

```text
Common prefix
      ↓
Cached
      ↓
Request 1 → reuse
Request 2 → reuse
Request 3 → reuse
```

The changing part is mainly:

```text
User question
```

---

# 24. Cache Write vs Cache Read

An important concept in provider-level caching is that there can be different costs for:

```text
Cache Write
```

and:

```text
Cache Read / Cache Hit
```

Think of it like:

```text
First request
    ↓
Create cache
    ↓
Cache write

Later requests
    ↓
Reuse cache
    ↓
Cache reads
```

A provider may charge differently for these operations.

Therefore, caching economics depend on how many times the cached content is reused.

---

# 25. Break-Even Analysis

The source discusses a **break-even** idea.

Suppose caching has some setup/write cost but gives a cheaper price for later reads.

Then:

```text
Low reuse
   ↓
Caching may not be worthwhile

High reuse
   ↓
Caching becomes increasingly valuable
```

The source gives reuse thresholds and provider-specific pricing examples, but those figures are time-sensitive.

The general lesson is more important:

> **Context caching becomes economically attractive when the same prefix is reused enough times to recover the cache-write/storage overhead.**

---

# 26. Context Caching vs RAG

The document asks:

> Why can Context Caching be better than RAG for a 50k-token document?

The source gives three reasons.

### 1. Recall

With the whole document in context:

```text
Entire document
      ↓
Model can see everything
```

With RAG:

```text
Question
   ↓
Retriever
   ↓
Top-k chunks
   ↓
Model
```

RAG depends on the retriever finding the right chunks.

So the source argues that context caching can provide higher recall for a medium-sized document.

---

# 27. Coherence

If the entire document is available:

```text
Document
 ├── Section A
 ├── Section B
 ├── Section C
 └── Section D
```

the model can potentially reason across relationships between those sections.

With RAG:

```text
Question
   ↓
Retrieve selected chunks
   ↓
Model sees only retrieved context
```

Some relevant cross-references may not be retrieved.

Therefore the source argues that context caching can provide better whole-document coherence for suitable document sizes.

---

# 28. Economics

The source's argument is:

For a medium-sized document, if cached-context pricing is sufficiently cheap:

```text
50k-token document
       ↓
Context caching
```

may be simpler than building:

```text
Document ingestion
      ↓
Parsing
      ↓
Chunking
      ↓
Embedding
      ↓
Vector DB
      ↓
Retrieval
      ↓
Reranking
      ↓
LLM
```

This does **not** mean RAG is obsolete.

It means:

> For some document sizes and workloads, sending/caching the entire document can be simpler and potentially cheaper than maintaining a retrieval pipeline.

---

# 29. Context Caching vs RAG — Important Nuance

Do not memorize:

> "Context caching is always better than RAG."

The better mental model is:

```text
Small / medium context
+
High reuse
+
Affordable context window
      ↓
Context caching can be attractive
```

Whereas:

```text
Huge corpus
+
Need to search across millions of documents
+
Only small portions are relevant per query
      ↓
RAG becomes much more attractive
```

So the decision depends on:

- corpus size
- context-window capacity
- reuse rate
- latency
- token cost
- retrieval quality
- operational complexity

---

# 30. RAD-O — Retrieval Augmented Decoding

The final technical concept in the chapter is **RAD-O**.

The source describes it as a context-caching approach where the model compresses the KV cache of long documents into **latent tokens**.

The basic idea:

```text
Very long document
       ↓
Full KV representation
       ↓
Compression
       ↓
Latent tokens
       ↓
Much smaller representation
```

---

# 31. Why Compress the KV Cache?

Suppose a document has:

```text
1,000,000 tokens
```

Storing full KV representations for all those tokens can require enormous memory.

RAD-O, as described by the source, instead creates a compressed representation.

The source gives an illustrative claim of:

```text
Full KV representation
       ↓
~10× smaller compressed representation
```

The stated impact is enabling contexts of:

```text
2M+ tokens
```

on hardware that previously supported around:

```text
200k tokens
```

The important concept is:

> **Instead of storing every token's full KV representation, compress the long-context information into a smaller learned representation.**

---

# 32. How RAD-O Fits with the Other Techniques

Now the whole chapter can be viewed as several different optimization layers.

### GQA

Reduces the number of KV heads:

```text
Many KV heads
      ↓
Fewer KV heads
      ↓
Smaller KV Cache
```

### PagedAttention

Manages KV memory efficiently:

```text
KV Cache
      ↓
Fixed-size blocks
      ↓
Better memory utilization
```

### Context Caching

Avoids repeated processing of the same context:

```text
Common context
      ↓
Cache
      ↓
Reuse
```

### RAD-O

Compresses long-context representations:

```text
Huge KV representation
      ↓
Compressed latent representation
      ↓
Much smaller memory footprint
```

---

# 33. The Full Mental Model

```text
                    LONG-CONTEXT LLM
                           |
                     KV CACHE PROBLEM
                           |
          +----------------+----------------+
          |                |                |
         GQA       PagedAttention    Context Caching
          |                |                |
   Smaller KV cache   Better memory    Reuse common
                      allocation         prefixes
          |                |                |
          +----------------+----------------+
                           |
                     More efficient
                       inference
                           |
                         RAD-O
                           |
                  Compress long-context
                     representations
```

---

# 34. PagedAttention Connection

The chapter's interview question connects directly to your previous PagedAttention notes.

Traditional approach:

```text
One giant contiguous KV allocation
```

Problems:

```text
External fragmentation
+
Memory waste
```

PagedAttention:

```text
KV Cache
   ↓
Small fixed-size pages/blocks
   ↓
Non-contiguous physical memory
   ↓
Block Table
   ↓
Dynamic allocation
```

This lets the system allocate memory as needed and potentially share blocks for common prefixes.

---

# 35. PagedAttention vs Context Caching

These are easy to confuse.

### PagedAttention

Answers:

> **How should KV-cache memory be physically managed?**

```text
Blocks
+
Block Table
+
Dynamic allocation
```

### Context Caching

Answers:

> **Can I reuse already-computed context instead of processing it again?**

```text
Common prefix
     ↓
Cache
     ↓
Reuse
```

They can work together.

For example:

```text
Context caching
      ↓
Reuse shared KV state
      ↓
PagedAttention
      ↓
Store/manage those KV blocks efficiently
```

---

# 36. GQA vs PagedAttention vs Context Caching

Remember this table:

| Technique | Main Problem Solved |
|---|---|
| **GQA** | Reduces KV-cache size |
| **PagedAttention** | Manages KV-cache memory efficiently |
| **Context Caching** | Reuses previously computed context |
| **RAD-O** | Compresses long-context representations |

This is one of the most useful interview tables from the chapter.

---

# 37. Interview Question: How Does PagedAttention Help KV Cache Management?

Strong answer:

> PagedAttention breaks the KV Cache into small fixed-size pages or blocks instead of requiring one large contiguous allocation. A block table maps logical blocks to physical GPU-memory blocks. This reduces fragmentation, allows memory to be allocated as needed, and can enable sharing of common-prefix blocks between requests.

---

# 38. Interview Question: Why Can Context Caching Beat RAG for a 50k-Token Document?

According to the source:

> For an appropriately sized document and sufficiently cheap caching, context caching can be simpler and potentially more effective than introducing a retrieval pipeline.

Three arguments:

### Recall

```text
Context caching
→ entire document available
```

versus:

```text
RAG
→ only retrieved chunks available
```

### Coherence

```text
Entire document
→ cross-references remain visible
```

### Economics

```text
50k tokens
+
high reuse
+
cheap cached input
```

can potentially be cheaper/simpler than maintaining a complete vector-search pipeline.

---

# 39. But Context Caching Does Not Replace RAG Everywhere

This is an important system-design distinction.

Imagine:

```text
10 documents × 50k tokens
```

You may be able to put everything into context.

But imagine:

```text
10 million documents
×
50k tokens each
```

Putting everything into every prompt is impossible or economically unreasonable.

Then:

```text
Question
   ↓
Retriever
   ↓
Relevant documents/chunks
   ↓
LLM
```

becomes much more appropriate.

So:

> **Context caching is a reuse optimization; RAG is a retrieval strategy.**

---

# 40. A Practical Decision Framework

When deciding between the approaches, ask:

### Question 1 — How large is the corpus?

```text
Small
→ Context caching may work

Huge
→ RAG likely necessary
```

### Question 2 — How much is reused?

```text
Same prefix repeatedly
→ Caching is attractive
```

### Question 3 — How much context is relevant per query?

```text
Most of the document
→ Context caching can work

Tiny portion of huge corpus
→ RAG is better
```

### Question 4 — What matters more?

```text
Maximum recall
→ Full context has an advantage

Lower context processing
→ Retrieval may have an advantage
```

---

# 41. Key Terms to Memorize

### KV Cache

Stores previously computed Key and Value tensors so they can be reused during autoregressive Decode.

### GQA

Grouped Query Attention. Multiple Query heads share fewer KV heads.

### MHA

Multi-Head Attention. Typically each Query head has its own K/V heads.

### MQA

Multi-Query Attention. Many Query heads share a single KV head/group.

### Context Caching

Caching previously computed context so it can be reused by later requests.

### Prompt Caching

Provider/API-level context caching for repeated prompt prefixes.

### Prefix

The common beginning portion of multiple requests.

### Cache Hit

A request can reuse an already-created cached representation.

### Cache Write

Creating/storing the cached representation for the first time.

### RAD-O

Retrieval Augmented Decoding, described in the source as compressing long-document KV information into latent tokens.

---

# 42. The Most Important Formulas / Relationships

### KV Cache

```text
KV Cache
∝
layers
×
tokens
×
KV heads
×
head dimension
×
2 (K + V)
×
bytes per value
```

### GQA

```text
KV heads ↓
    ↓
KV Cache ↓
    ↓
Memory bandwidth ↓
```

### Context length

```text
Context length ↑
        ↓
KV Cache ↑
        ↓
VRAM requirement ↑
```

### Context caching

```text
Repeated prefix
      ↓
Cache once
      ↓
Reuse many times
      ↓
Repeated work/cost ↓
```

### PagedAttention

```text
KV Cache
      ↓
Blocks
      ↓
Block Table
      ↓
Non-contiguous physical memory
      ↓
Fragmentation ↓
```

---

# 43. One Diagram to Remember

```text
                    USER REQUEST
                         |
                         ↓
                    LONG PROMPT
                         |
                         ↓
                    KV CACHE
                         |
       +-----------------+------------------+
       |                 |                  |
      GQA          PagedAttention     Context Cache
       |                 |                  |
   Fewer KV         Block-based        Reuse shared
     heads           allocation          prefixes
       |                 |                  |
       ↓                 ↓                  ↓
   Cache size ↓    Fragmentation ↓    Repeated work ↓
       |                 |                  |
       +-----------------+------------------+
                         |
                         ↓
                 Efficient Inference
                         |
                         ↓
                    RAD-O (optional)
                         |
                         ↓
             Compress long-context KV
```

---

# 44. Final Interview Summary

If an interviewer asks:

### "Why is KV Cache important?"

> During autoregressive Decode, the model needs K and V representations for previous tokens. Caching them avoids recomputing them, but the cache can become a major memory consumer, especially for long contexts and many concurrent users.

### "How does GQA help?"

> GQA lets multiple Query heads share fewer KV heads, reducing KV-cache size and memory bandwidth requirements while retaining more flexibility than MQA.

### "What does Context Caching do?"

> It caches already-computed context, especially common prefixes, so subsequent requests can reuse the cached representation rather than repeatedly processing the same context.

### "What does PagedAttention do?"

> It manages KV Cache using small fixed-size blocks mapped from logical to physical memory, reducing fragmentation and improving GPU memory utilization.

### "What is RAD-O?"

> In the attached document, RAD-O is described as compressing long-document KV representations into smaller latent-token representations, reducing memory required for very long contexts.

---

# 45. Final Mental Model

Remember these four questions:

```text
GQA
"What if my KV Cache is too large?"
        ↓
Use fewer KV heads.

PagedAttention
"What if my KV memory is fragmented?"
        ↓
Use blocks/pages.

Context Caching
"What if I keep seeing the same context?"
        ↓
Cache and reuse it.

RAD-O
"What if the context itself is enormous?"
        ↓
Compress its KV representation.
```

### One sentence

> **GQA reduces the amount of KV Cache, PagedAttention manages that cache efficiently in memory, Context Caching avoids recomputing repeated context, and RAD-O—as described in the source—compresses very long-context KV representations.**

---

## Source Note

This document is based on the attached:

`02-kv-cache-and-context-caching.md`

The source contains provider-specific pricing, model names, performance numbers, and dated claims. Those values can change over time and should be independently verified before using them for production architecture or cost decisions.
