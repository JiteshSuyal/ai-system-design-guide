# 04 --- Vector Databases: Detailed Interview Notes

> **Source document:** `04-vector-databases.md`
>
> This is a detailed explanation of the exact uploaded chapter, keeping
> its organization, terminology, examples, and interview framing. The
> goal is to understand **why each concept exists, how it works, and how
> to discuss it in an AI Engineer system-design interview**.

------------------------------------------------------------------------

# 1. What Is a Vector Database?

A vector database is a database designed to store and search
**embeddings**.

An embedding is a high-dimensional vector representing some object such
as:

``` text
Document
Image
Audio
Product
User
Chunk of text
```

For example:

``` text
"How do I reset my password?"
              ↓
       Embedding model
              ↓
[0.12, -0.43, 0.91, ...]
```

The vector database stores that vector and lets us find other vectors
that are close to it.

The source's key framing is:

``` text
Traditional database
    ↓
Exact attribute lookup

Vector database
    ↓
Similarity search
```

The chapter specifically asks us to think beyond:

> "Does this database support vector search?"

and instead ask:

> **Can it scale to 100M+ vectors with sub-100ms P99 and full metadata
> filtering?** fileciteturn10file14

That is an important **system-design mindset**.

------------------------------------------------------------------------

# 2. Traditional Database vs Vector Database

Suppose we have documents:

``` text
Document A:
"How to reset my password"

Document B:
"How to change my username"

Document C:
"How to configure Kafka"
```

A traditional query might be:

``` sql
SELECT *
FROM docs
WHERE category = 'account';
```

This is an exact/filter-based lookup.

A vector query is different:

``` text
User:
"I forgot my login password"

       ↓
Embedding

       ↓
[0.21, 0.71, ...]
```

We want:

``` text
Find documents whose embeddings
are closest to this query embedding.
```

So conceptually:

``` sql
SELECT *
FROM docs
ORDER BY similarity(embedding, query_embedding)
LIMIT 10;
```

The source uses this exact contrast to explain the role of vector
databases. fileciteturn10file14

------------------------------------------------------------------------

# 3. Core Capabilities

The source identifies five major capabilities.

## 3.1 Vector Storage

Store:

``` text
id
vector
metadata
```

Example:

``` json
{
  "id": "doc_123",
  "vector": [0.12, -0.44, 0.91],
  "metadata": {
    "tenant_id": "abc",
    "category": "technical"
  }
}
```

------------------------------------------------------------------------

## 3.2 Similarity Search

Given:

``` text
query vector
```

find:

``` text
nearest vectors
```

Usually we ask for:

``` text
Top K
```

For example:

``` text
Top 5
```

------------------------------------------------------------------------

## 3.3 Metadata Filtering

We might want:

``` text
semantic similarity
+
tenant_id = "abc"
+
category = "technical"
```

This is extremely important in production.

------------------------------------------------------------------------

## 3.4 CRUD

Vector databases are not just read-only search engines.

We need to:

``` text
Create
Read
Update
Delete
```

because source documents change.

------------------------------------------------------------------------

## 3.5 Scaling

The system may need to support:

``` text
1 million vectors
10 million
100 million
1 billion+
```

At that point, brute-force search becomes expensive.

------------------------------------------------------------------------

# 4. Why Not Just Use PostgreSQL?

This is an important interview question.

The answer is **not**:

> "PostgreSQL cannot store vectors."

It can.

For example:

``` text
PostgreSQL + pgvector
```

supports vector search.

The issue is scale and specialization.

Suppose:

``` text
n = number of vectors
d = vector dimensions
```

Brute-force search approximately costs:

``` text
O(n × d)
```

because the query vector has to be compared with every stored vector.

If:

``` text
n = 10,000,000
d = 1536
```

we have a huge amount of distance computation.

Dedicated ANN indexes reduce the amount of the database that must be
searched.

The source contrasts brute-force search with ANN indexing and describes
dedicated vector databases as suitable for much larger collections.
fileciteturn10file10

------------------------------------------------------------------------

# 5. Exact Search vs Approximate Search

This is the fundamental trade-off in vector databases.

## Exact Search

Suppose we have:

``` text
1,000,000 vectors
```

For every query:

``` text
Query
 ↓
Compare against vector 1
 ↓
vector 2
 ↓
vector 3
 ↓
...
 ↓
vector 1,000,000
```

You find the true nearest neighbors.

Advantages:

``` text
100% recall
Simple
No index complexity
```

Disadvantage:

``` text
Expensive at scale
```

The source describes this as:

``` text
O(n × d)
```

per query. fileciteturn10file1

------------------------------------------------------------------------

# 6. Approximate Nearest Neighbor --- ANN

ANN means:

> **Approximate Nearest Neighbor**

Instead of comparing against every vector:

``` text
Query
 ↓
Index
 ↓
Small candidate region
 ↓
Nearest candidates
```

The index tries to avoid obviously irrelevant vectors.

Therefore:

``` text
Search space ↓
Latency ↓
Compute ↓
```

but potentially:

``` text
Recall ↓ slightly
```

The source gives a typical recall range of roughly:

``` text
95–99%
```

for ANN configurations, while exact search has perfect recall.
fileciteturn10file1

------------------------------------------------------------------------

# 7. Recall vs Latency

This trade-off is critical.

Imagine:

``` text
Exact search
Recall: 100%
Latency: 100 ms

ANN
Recall: 98%
Latency: 10 ms
```

If the application requires:

``` text
< 20 ms
```

you may accept the 2% recall loss.

But if the application is:

``` text
Legal search
Medical retrieval
High-value compliance search
```

you may want higher recall.

So vector search is not:

> "Always maximize accuracy."

It is:

> **Choose the best recall/latency point for the application.**

The source explicitly presents this as a tuning trade-off.
fileciteturn10file1

------------------------------------------------------------------------

# 8. Distance Metrics

The source discusses three major metrics:

``` text
Cosine
Euclidean / L2
Dot product
```

------------------------------------------------------------------------

# 9. Cosine Similarity

Cosine measures the angle between vectors.

Formula:

``` text
cosine_similarity(a,b)
=
(a · b) / (||a|| ||b||)
```

The source expresses cosine distance as:

``` text
1 - cosine_similarity
```

So:

``` text
similar vectors
     ↓
small cosine distance
```

Cosine is commonly used for text embeddings according to the source.

------------------------------------------------------------------------

# 10. Intuition Behind Cosine

Consider:

``` text
A = [1, 1]
B = [2, 2]
```

These vectors point in the same direction.

Their magnitudes are different:

``` text
A length ≠ B length
```

but their direction is the same.

Cosine similarity focuses on direction.

This is useful for semantic embeddings because the orientation of the
embedding can matter more than raw magnitude.

------------------------------------------------------------------------

# 11. Euclidean Distance

Euclidean distance is ordinary geometric distance.

Formula:

``` text
sqrt(
  Σ(ai - bi)²
)
```

Example:

``` text
A = [1,2]
B = [4,6]
```

Distance:

``` text
sqrt(
  (1-4)² + (2-6)²
)

=
sqrt(9 + 16)

=
5
```

The source associates L2 with image embeddings. fileciteturn10file1

------------------------------------------------------------------------

# 12. Dot Product

Dot product:

``` text
a · b
=
Σ(ai × bi)
```

Example:

``` text
A = [1,2]
B = [3,4]

= 1×3 + 2×4
= 11
```

The source notes dot product as useful when vectors are already
normalized.

------------------------------------------------------------------------

# 13. Which Metric Should You Choose?

The source's practical guidance:

``` text
Text embeddings
    ↓
Cosine

Normalized vectors
    ↓
Dot product can be used

Image embeddings
    ↓
L2 can be useful
```

But in a real system, the correct choice ultimately depends on the
embedding model and how it was trained/evaluated.

### Interview answer

> I would use the similarity metric recommended by the embedding model's
> training setup and validate it empirically. For typical text
> embeddings, cosine similarity or dot product with normalized vectors
> is common.

------------------------------------------------------------------------

# 14. HNSW

The source calls HNSW:

> **Hierarchical Navigable Small World**

and describes it as one of the most popular algorithms for production
in-memory vector search. fileciteturn10file1

The basic idea:

> Build a graph connecting vectors to nearby vectors so that search can
> navigate toward the nearest region instead of checking every vector.

------------------------------------------------------------------------

# 15. HNSW --- Simple Mental Model

Imagine 1 million cities.

You want to find the city closest to a location.

Brute force:

``` text
Check city 1
Check city 2
Check city 3
...
Check city 1,000,000
```

HNSW instead creates a navigation network:

``` text
Top layer
    ↓
Large jumps

Middle layer
    ↓
Smaller jumps

Bottom layer
    ↓
Detailed neighborhood
```

So search becomes:

``` text
Start at top
 ↓
Move toward query
 ↓
Move down a layer
 ↓
Move toward query
 ↓
Move down
 ↓
Find nearest candidates
```

The source illustrates this with multiple graph layers.
fileciteturn10file1

------------------------------------------------------------------------

# 16. Why Is HNSW Hierarchical?

Imagine:

``` text
Layer 2:

A ----------------------- Z

Layer 1:

A ---- F ---- M ---- R --- Z

Layer 0:

A-B-C-D-E-F-G-H-I-J-K-L-M...
```

The upper layers let us make large jumps.

The bottom layer contains all vectors and allows precise navigation.

Therefore:

``` text
Upper layers
    ↓
Fast navigation

Lower layers
    ↓
Precision
```

This is the intuition behind the hierarchy.

------------------------------------------------------------------------

# 17. HNSW Search

At query time:

``` text
Query
 ↓
Enter high-level graph
 ↓
Find promising node
 ↓
Greedily move toward closer nodes
 ↓
Drop to next layer
 ↓
Repeat
 ↓
Bottom layer
 ↓
Top candidates
```

The word **greedy** is important.

At each stage, the search tries to move toward nodes that appear closer
to the query.

------------------------------------------------------------------------

# 18. HNSW Advantages

The source lists:

-   Excellent recall/latency trade-off
-   No training required
-   Supports updates natively

That makes HNSW attractive for dynamic production workloads.
fileciteturn10file1

------------------------------------------------------------------------

# 19. HNSW Disadvantages

The major disadvantage:

``` text
Memory
```

The graph itself consumes memory.

The source estimates:

``` text
Index size ≈ 1.5–2× vector data
```

depending on configuration. fileciteturn10file10

So HNSW can become expensive at very large scale.

------------------------------------------------------------------------

# 20. HNSW Parameters

Three parameters are particularly important.

## M

``` text
M = maximum connections per node
```

Typical range in the source:

``` text
16–64
```

Higher M:

``` text
More connections
 ↓
Potentially better recall
 ↓
More memory
```

------------------------------------------------------------------------

## ef_construction

Controls search breadth while building the graph.

Higher:

``` text
Better graph construction
Potentially better recall
More build time
```

------------------------------------------------------------------------

## ef_search

Controls search breadth at query time.

Higher:

``` text
More candidates explored
 ↓
Recall ↑
Latency ↑
```

This parameter is therefore an important runtime tuning knob.

------------------------------------------------------------------------

# 21. The HNSW Interview Question

### Question

> Explain HNSW.

### Strong answer

> HNSW builds a multi-layer graph over vectors. Upper layers contain
> fewer nodes and allow large jumps, while lower layers provide
> increasingly fine-grained navigation. At query time, we start at the
> top, greedily move toward the query, descend layers, and finally
> search the dense bottom layer. It provides an excellent recall/latency
> trade-off and supports updates, but its graph consumes substantial
> RAM.

------------------------------------------------------------------------

# 22. DiskANN

HNSW assumes that a large part of the index can stay in RAM.

DiskANN changes the storage model.

The source describes it as an SSD-based approach using the Vamana graph
algorithm. fileciteturn10file1

Conceptually:

``` text
HNSW

RAM
 └── Large graph


DiskANN

RAM
 └── Small critical index

NVMe SSD
 └── Large graph/data
```

------------------------------------------------------------------------

# 23. Why DiskANN?

Suppose:

``` text
100M vectors
1536 dimensions
```

The source states that an HNSW configuration can require nearly:

``` text
1 TB RAM
```

at this scale.

That becomes expensive.

DiskANN moves much of the storage burden to SSD.

The source describes:

``` text
90–95% RAM reduction
```

in its example while targeting sub-10ms query times.
fileciteturn10file10

------------------------------------------------------------------------

# 24. HNSW vs DiskANN

                 HNSW                    DiskANN
  -------------- ----------------------- ---------------------------------
  Main storage   RAM                     SSD
  Memory cost    High                    Lower
  Latency        Excellent               Slightly higher
  Scale          Large                   Very large
  Updates        Good                    Fair
  Best use       In-memory low latency   Huge datasets / cost efficiency

### Mental model

``` text
HNSW
→ Spend RAM to get speed

DiskANN
→ Spend SSD I/O to reduce RAM cost
```

------------------------------------------------------------------------

# 25. IVF

IVF means:

> **Inverted File Index**

The source explains it as clustering vectors into groups.

Suppose we have:

``` text
1,000,000 vectors
```

We can create:

``` text
1,000 clusters
```

using k-means.

Each vector gets assigned to its nearest centroid.

------------------------------------------------------------------------

# 26. IVF Index Construction

Step 1:

``` text
Run k-means
```

to produce:

``` text
Centroid 1
Centroid 2
...
Centroid 1000
```

Step 2:

``` text
Vector
 ↓
Nearest centroid
 ↓
Cluster
```

So:

``` text
Cluster 1 → vectors
Cluster 2 → vectors
...
```

------------------------------------------------------------------------

# 27. IVF Query

At query time:

``` text
Query
 ↓
Find nearest centroids
 ↓
Search only those clusters
 ↓
Return nearest vectors
```

Instead of searching:

``` text
1,000,000 vectors
```

we may search only:

``` text
10 clusters
```

or another configured number.

------------------------------------------------------------------------

# 28. IVF Parameters

The source highlights:

``` text
nlist
nprobe
```

## nlist

Number of clusters.

The source gives:

``` text
sqrt(n)
```

as a rule of thumb.

For:

``` text
n = 1,000,000
```

sqrt(n):

``` text
≈ 1,000
```

So an initial `nlist` around 1,000 may be a reasonable starting point.

But this is a starting heuristic, not a universal optimum.

------------------------------------------------------------------------

## nprobe

Number of clusters searched at query time.

Example:

``` text
nlist = 1000
nprobe = 10
```

means:

``` text
Find 10 promising clusters
 ↓
Search those clusters
```

Higher `nprobe`:

``` text
Recall ↑
Latency ↑
```

------------------------------------------------------------------------

# 29. IVF vs HNSW

### HNSW

``` text
Graph navigation
```

### IVF

``` text
Cluster selection
```

So:

``` text
HNSW → navigate graph
IVF  → choose clusters
```

IVF can have lower memory requirements and can be combined with PQ.

The source notes that IVF requires training and that updates can require
re-clustering or a hybrid approach. fileciteturn10file10

------------------------------------------------------------------------

# 30. Product Quantization --- PQ

PQ means:

> **Product Quantization**

The purpose is:

``` text
Compress vectors
 ↓
Reduce memory
 ↓
Speed up comparisons
```

Suppose:

``` text
1536-dimensional vector
```

Instead of storing all dimensions as full-precision floats, PQ divides
the vector into smaller subvectors.

------------------------------------------------------------------------

# 31. How PQ Works

Suppose:

``` text
Vector:

[x1 x2 x3 x4 x5 x6 x7 x8]
```

Split into:

``` text
[x1 x2]
[x3 x4]
[x5 x6]
[x7 x8]
```

Each subvector is quantized using a codebook.

Instead of storing the original floating-point values, we store compact
codes representing nearby centroids.

So:

``` text
Original vector
     ↓
Subvectors
     ↓
Quantization
     ↓
Compact codes
```

The source gives typical memory reduction of:

``` text
4–32×
```

with an accuracy trade-off. fileciteturn10file10

------------------------------------------------------------------------

# 32. Why Does PQ Reduce Memory?

Suppose float32:

``` text
4 bytes / value
```

For:

``` text
1536 dimensions
```

one vector requires:

``` text
1536 × 4
=
6144 bytes
```

≈ 6 KB before metadata/index overhead.

If you represent that vector using compact codes, storage can become
much smaller.

This matters when:

``` text
100M vectors
```

or:

``` text
1B vectors
```

are involved.

------------------------------------------------------------------------

# 33. PQ Trade-off

The compression is lossy.

Therefore:

``` text
Memory ↓↓↓
Latency ↓
Recall/accuracy ↓
```

So PQ is useful when:

``` text
Memory is the bottleneck
```

and a small loss in retrieval quality is acceptable.

------------------------------------------------------------------------

# 34. Flat Index

Flat means:

> Don't use an approximate index.

Store the vectors and compare directly.

``` text
Query
 ↓
Compare against every vector
 ↓
Sort/select top K
```

Advantages:

``` text
Exact
Simple
100% recall
```

Disadvantages:

``` text
Slow at large scale
```

The source recommends it for relatively small datasets, especially when
exact accuracy matters. fileciteturn10file10

------------------------------------------------------------------------

# 35. LSH

The source also mentions:

> **LSH --- Locality Sensitive Hashing**

as an alternative for very high-dimensional sparse vectors.

The key idea is to hash similar objects into the same or nearby buckets.

Conceptually:

``` text
Vector
 ↓
Hash function
 ↓
Bucket
```

Similar vectors have a higher probability of ending up in the same
bucket.

------------------------------------------------------------------------

# 36. Index Comparison

The source provides this comparison:

  Algorithm   Memory      Build    Query       Recall    Updates
  ----------- ----------- -------- ----------- --------- ---------
  HNSW        High        Medium   Very fast   95--99%   Good
  DiskANN     Low / SSD   Medium   Fast        95--99%   Fair
  IVF         Medium      Fast     Fast        90--98%   Fair
  IVF-PQ      Low         Fast     Fast        85--95%   Fair
  Flat        Low         None     Slow        100%      Instant

The exact values should be treated as **source-level indicative
ranges**, not guarantees for every workload. fileciteturn10file10

------------------------------------------------------------------------

# 37. How to Choose an Index

Think:

``` text
Small dataset?
    ↓
Flat

Need excellent in-memory retrieval?
    ↓
HNSW

Huge dataset + RAM expensive?
    ↓
DiskANN

Memory constrained?
    ↓
IVF-PQ

Need clustering-based search?
    ↓
IVF
```

But always benchmark against:

``` text
Your data
Your embedding model
Your filters
Your QPS
Your latency target
```

------------------------------------------------------------------------

# 38. Vector Database Landscape

The source divides databases into:

``` text
Vector-native dedicated databases
```

and:

``` text
General-purpose databases/search systems
```

------------------------------------------------------------------------

# 39. Pinecone

The source positions Pinecone as:

``` text
Managed cloud
Serverless
Easy to start
Managed SLAs
```

The important engineering trade-off:

``` text
Less infrastructure work
+
Faster development
-
More vendor dependency
-
Managed-service cost
```

------------------------------------------------------------------------

# 40. Qdrant

The source positions Qdrant as:

``` text
Open source
Cloud
Rust
High performance
Self-hosting
```

Useful when you want:

``` text
Control
+
Open-source option
+
High performance
```

------------------------------------------------------------------------

# 41. Weaviate

The source highlights:

``` text
Hybrid search
BM25 + dense vectors
Metadata
Multimodal
```

So it can be attractive when lexical and semantic retrieval need to
coexist naturally.

------------------------------------------------------------------------

# 42. Milvus

The source positions Milvus around:

``` text
Distributed scale
50M+ vectors
Heterogeneous node types
Tiered storage
```

This makes it more relevant when vector search itself is becoming a
major distributed infrastructure component.

------------------------------------------------------------------------

# 43. Chroma

The source positions Chroma mainly for:

``` text
Prototyping
Local development
Embedded use
```

This is different from selecting a database for a huge production
workload.

------------------------------------------------------------------------

# 44. pgvector

pgvector is particularly interesting for backend engineers.

Instead of introducing another database:

``` text
PostgreSQL
   +
pgvector
```

You can keep:

``` text
Relational data
+
Metadata
+
Vectors
```

in the same ecosystem.

The source describes pgvector as suitable for smaller-scale deployments
and notes HNSW and IVFFlat support. fileciteturn10file11

------------------------------------------------------------------------

# 45. Why pgvector Can Be Attractive

Suppose your application already uses:

``` text
PostgreSQL
```

and has:

``` text
1M vectors
```

You may not need:

``` text
Pinecone
+
Qdrant
+
Milvus
```

just for vector retrieval.

You could use:

``` text
PostgreSQL
 ├── relational tables
 ├── metadata
 └── pgvector
```

This reduces operational complexity.

------------------------------------------------------------------------

# 46. Metadata Filtering

This is one of the most important sections in the source.

Suppose a SaaS application has:

``` text
Tenant A
Tenant B
Tenant C
```

and all documents are stored in one vector index.

A query from Tenant A should search only:

``` text
tenant_id = A
```

not:

``` text
all tenants
```

------------------------------------------------------------------------

# 47. Semantic Search + Metadata

Instead of:

``` text
Find nearest documents
```

we want:

``` text
Find nearest documents
WHERE
tenant_id = A
AND
category = technical
```

Conceptually:

``` text
Query embedding
      ↓
Vector search
      +
Metadata constraint
      ↓
Top K valid results
```

The source provides Pinecone and Qdrant examples for exactly this
pattern. fileciteturn10file8

------------------------------------------------------------------------

# 48. Why Post-Filtering Is Dangerous

Bad architecture:

``` text
Query
 ↓
Vector search across all tenants
 ↓
Top 10
 ↓
Filter tenant
```

Suppose:

``` text
Top 10 results
```

contain:

``` text
8 documents from Tenant B
2 documents from Tenant A
```

After filtering:

``` text
Only 2 results
```

Worse, if the system accidentally exposes intermediate results or
mishandles filtering, cross-tenant data can leak.

The source explicitly says:

> Never post-filter.

Instead use filtering during retrieval. fileciteturn10file12

------------------------------------------------------------------------

# 49. Why Metadata Filtering Can Be a Bottleneck

This is a subtle but very good interview topic.

HNSW wants to do:

``` text
Graph navigation
 ↓
Short path toward nearest vector
```

But now suppose we say:

``` text
Only tenant A
```

The graph contains:

``` text
Tenant A
Tenant B
Tenant C
...
```

The search cannot freely traverse every promising node.

It must respect the filter.

That can make graph traversal more expensive.

The source describes specialized pre-filtering techniques using:

``` text
Boolean constraints
Bitmasks
SIMD / hardware acceleration
```

to keep this fast. fileciteturn10file8

------------------------------------------------------------------------

# 50. Multi-Tenancy Strategies

The source gives three strategies.

## Strategy 1 --- Metadata Filtering

``` python
filter={
    "tenant_id": current_tenant
}
```

Advantages:

``` text
Simple
One index
Cost effective
```

Disadvantages:

``` text
Shared resources
Potential isolation mistakes
```

------------------------------------------------------------------------

## Strategy 2 --- Collection per Tenant

``` text
tenant_A_collection
tenant_B_collection
tenant_C_collection
```

Advantages:

``` text
Strong isolation
Independent scaling
```

Disadvantages:

``` text
Many collections
Operational complexity
```

------------------------------------------------------------------------

## Strategy 3 --- Namespace per Tenant

The source gives Pinecone namespaces as the example.

Conceptually:

``` text
Index
 ├── tenant_A namespace
 ├── tenant_B namespace
 └── tenant_C namespace
```

Advantages:

``` text
Isolation
Single index
```

Disadvantage:

``` text
Vendor-specific behavior
```

------------------------------------------------------------------------

# 51. What Would I Choose?

The source's recommendation:

``` text
Most applications
    ↓
Metadata filtering

High-security isolation
    ↓
Separate collections

Never
    ↓
Post-filtering
```

This is a good interview answer because it considers both:

``` text
simplicity
```

and:

``` text
security/isolation
```

------------------------------------------------------------------------

# 52. Query Pattern 1 --- Semantic Search

The source's basic flow:

``` python
query_embedding = embed(query)

results = vector_db.search(
    query_embedding,
    top_k=5
)
```

Architecture:

``` text
User query
    ↓
Embedding model
    ↓
Query vector
    ↓
Vector DB
    ↓
Nearest neighbors
    ↓
Top K documents
```

This is the basic retrieval primitive. fileciteturn10file13

------------------------------------------------------------------------

# 53. Query Pattern 2 --- Filtered Search

Add:

``` text
metadata filters
```

Example:

``` text
tenant_id = abc
created_after = 2025-01-01
```

Architecture:

``` text
Query
 ↓
Embedding
 ↓
Vector DB
 ↓
Vector similarity
 +
Metadata filtering
 ↓
Top K
```

This is much closer to production RAG.

------------------------------------------------------------------------

# 54. Query Pattern 3 --- Hybrid Search

This is extremely important.

Dense retrieval:

``` text
semantic meaning
```

Sparse retrieval:

``` text
exact words / lexical matching
```

Why combine them?

Because semantic search can sometimes miss exact identifiers.

Example:

``` text
"error code EKS-429"
```

A keyword system may be excellent at finding:

``` text
EKS-429
```

while dense search is better at:

``` text
"I keep getting throttled by the service"
```

------------------------------------------------------------------------

# 55. Dense + Sparse

The source's hybrid flow:

``` text
                 Query
                   │
          ┌────────┴────────┐
          ↓                 ↓
      Dense search       BM25
          ↓                 ↓
      Results A          Results B
          └────────┬────────┘
                   ↓
             RRF / fusion
                   ↓
                Top K
```

The source uses:

> Reciprocal Rank Fusion --- RRF

to combine rankings. fileciteturn10file2

------------------------------------------------------------------------

# 56. What Is RRF?

Suppose:

``` text
Dense ranking:

A
B
C
D
```

and:

``` text
BM25 ranking:

C
A
E
B
```

Instead of comparing raw scores from two different systems, RRF combines
their rankings.

Conceptually:

``` text
rank 1 → high contribution
rank 2 → slightly lower
rank 3 → lower
...
```

A document appearing highly in multiple ranking systems receives a
strong combined score.

This is useful because:

``` text
Dense scores
```

and:

``` text
BM25 scores
```

are not necessarily directly comparable.

------------------------------------------------------------------------

# 57. Hybrid Search Parameter α

The source's Weaviate example uses:

``` text
alpha = 0.5
```

and explains:

``` text
alpha = 0
→ BM25 only

alpha = 1
→ vector only
```

So:

``` text
0.5
```

means a balanced combination.

The optimal value depends on the application.

------------------------------------------------------------------------

# 58. Query Pattern 4 --- Multi-Vector Query

Sometimes one query is insufficient.

For example:

``` text
"Compare refund policy and cancellation policy."
```

We may break this into:

``` text
Query 1:
refund policy

Query 2:
cancellation policy
```

Then:

``` text
Search each query
 ↓
Combine results
 ↓
Deduplicate
 ↓
Rerank
 ↓
Top K
```

The source gives this exact pattern for parent-child and multi-aspect
retrieval. fileciteturn10file2

------------------------------------------------------------------------

# 59. Why Rerank After Multi-Vector Retrieval?

Suppose:

``` text
Query A → documents 1,2,3
Query B → documents 3,4,5
```

Combined:

``` text
1,2,3,4,5
```

Document 3 appeared in both searches.

That may be a strong signal.

So:

``` text
Deduplicate
 ↓
Reranker
 ↓
Better ordering
```

The source uses a reranking step after deduplication.
fileciteturn10file15

------------------------------------------------------------------------

# 60. Capacity Planning

This section changes the conversation from:

> "Can vector search work?"

to:

> **"Can this system support our workload?"**

The source gives a resource estimation model.

------------------------------------------------------------------------

# 61. Vector Storage Calculation

Suppose:

``` text
dimensions = 1536
precision = float32
```

float32:

``` text
4 bytes
```

So:

``` text
Vector size
=
1536 × 4
=
6144 bytes
```

≈:

``` text
6 KB
```

per vector before index and metadata overhead.

------------------------------------------------------------------------

# 62. 10 Million Vector Example

Suppose:

``` text
10,000,000 vectors
1536 dimensions
float32
```

Raw vector storage:

``` text
10,000,000 × 1536 × 4
```

=:

``` text
61,440,000,000 bytes
```

≈:

``` text
61.44 GB
```

Then the source uses an approximate HNSW overhead factor:

``` text
1.5×
```

So index-related capacity becomes larger.

Then metadata is added.

This is why:

``` text
Raw vector size
```

is NOT:

``` text
Total database size
```

------------------------------------------------------------------------

# 63. Source's Capacity Planning Model

The source uses:

``` python
vector_size = dimensions * 4
total_vector_storage = num_vectors * vector_size

index_overhead = total_vector_storage * 1.5

metadata_storage = num_vectors * metadata_size_bytes

total_gb =
(
    total_vector_storage
    + index_overhead
    + metadata_storage
) / 1e9
```

This is an estimation model, not a universal capacity formula.

Actual usage depends on:

``` text
index implementation
metadata representation
replication
deleted/tombstoned records
segments
compression
storage engine
```

------------------------------------------------------------------------

# 64. QPS Estimation

The source uses:

``` python
qps_per_gb = 50
```

and:

``` python
estimated_qps =
    total_gb * qps_per_gb
```

Important:

> This is explicitly a rough estimate.

Real QPS depends heavily on:

``` text
dimensions
index
ef_search
filters
hardware
concurrency
query distribution
replicas
cache
network
```

So don't present:

``` text
50 QPS/GB
```

as a universal law in an interview.

Say:

> "The document uses a rough capacity heuristic; I would benchmark the
> actual workload."

That is a stronger engineering answer.

------------------------------------------------------------------------

# 65. Index Maintenance

A vector database is a living system.

Documents are:

``` text
Added
Updated
Deleted
```

The source therefore includes a maintenance class.

------------------------------------------------------------------------

# 66. Batch Upserts

The source uses:

``` python
batch_size = 100
```

Why batch?

Instead of:

``` text
Request
Request
Request
Request
...
```

we can do:

``` text
Batch
Batch
Batch
```

Benefits:

``` text
Fewer network round trips
Better throughput
Better embedding pipeline utilization
```

But the optimal batch size depends on:

``` text
API limits
payload size
memory
embedding model
database
network
```

------------------------------------------------------------------------

# 67. Embedding During Upsert

The source shows:

``` python
embeddings = embed_batch(
    [d.text for d in batch]
)
```

Then:

``` text
Document
 ↓
Embedding
 ↓
Vector
 ↓
Metadata
 ↓
Upsert
```

This is the normal ingestion pipeline.

------------------------------------------------------------------------

# 68. Updating Metadata Without Re-Embedding

This is a useful optimization.

Suppose:

``` json
{
  "text": "...",
  "tenant_id": "abc",
  "category": "tech"
}
```

changes only:

``` text
category
```

The actual text didn't change.

Therefore:

``` text
No need to regenerate embedding
```

We can update only metadata.

The source explicitly demonstrates this. fileciteturn10file15

------------------------------------------------------------------------

# 69. High Availability

The source gives this architecture:

``` text
                Load Balancer
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
   Replica 1      Replica 2      Replica 3
      Read           Read          Primary
                                      │
                                Replication
                                      ↓
                                   Storage
```

The key patterns are:

``` text
Leader-follower writes
Read replicas
Async replication
```

------------------------------------------------------------------------

# 70. Why Read Replicas?

Vector databases often have:

``` text
Many reads
Fewer writes
```

For example:

``` text
1000 search requests/sec
10 updates/sec
```

We can scale read capacity by adding replicas:

``` text
Query
 ↓
Load balancer
 ├── Replica 1
 ├── Replica 2
 └── Replica 3
```

Writes can continue through the primary/leader.

------------------------------------------------------------------------

# 71. Async Replication

The source mentions asynchronous replication.

Conceptually:

``` text
Primary
 ↓
Write
 ↓
Return / continue
 ↓
Replica synchronization
```

Benefit:

``` text
Lower write latency
```

Trade-off:

``` text
Replica may temporarily lag
```

This is a classic distributed-systems trade-off.

------------------------------------------------------------------------

# 72. Monitoring

The source recommends monitoring:

``` text
query_latency_p50
query_latency_p99
queries_per_second
index_size_gb
vector_count
filter_latency
upsert_latency
cache_hit_rate
```

These metrics answer different questions.

------------------------------------------------------------------------

# 73. P50 vs P99

P50:

> Median latency.

If:

``` text
P50 = 20 ms
```

half of requests are below approximately 20ms.

P99:

> 99th percentile latency.

If:

``` text
P99 = 500 ms
```

1% of requests are slower than roughly 500ms.

For production systems:

> P99 is often more important than average latency.

Because tail latency affects user experience and downstream LLM latency.

------------------------------------------------------------------------

# 74. Recall Monitoring

Latency alone isn't enough.

Imagine:

``` text
Latency = 5ms
Recall = 60%
```

The system is fast but retrieves poor documents.

So the source includes:

``` text
bench_recall
```

as an alertable metric.

This is a critical AI-system point:

``` text
Infrastructure health
+
Retrieval quality
```

must both be monitored.

------------------------------------------------------------------------

# 75. Managed vs Self-Hosted

This is a system-design decision.

## Managed

Examples:

``` text
Pinecone
Qdrant Cloud
Weaviate Cloud
Zilliz
```

You outsource infrastructure operations.

Advantages:

``` text
Low operational overhead
Easy scaling
Managed reliability
Faster development
```

Disadvantages:

``` text
Higher managed-service cost
Vendor lock-in
Less infrastructure control
```

------------------------------------------------------------------------

# 76. Self-Hosted

Examples:

``` text
Qdrant
Milvus
Weaviate
```

You operate the system.

Advantages:

``` text
Control
Potentially lower unit cost at scale
Data control
Customization
```

Disadvantages:

``` text
Kubernetes / infrastructure work
Monitoring
Upgrades
Backups
Scaling
Failure recovery
SRE burden
```

------------------------------------------------------------------------

# 77. TCO --- Total Cost of Ownership

Never compare only:

``` text
$100 managed service
```

vs:

``` text
$50 EC2 instance
```

Self-hosting also costs:

``` text
Engineer time
Monitoring
Backups
On-call
Networking
Storage
Replication
Upgrades
Failure handling
```

Therefore:

``` text
TCO
=
Infrastructure
+
Operations
+
Engineering
+
Reliability
+
Support
```

The source explicitly frames managed-vs-self-hosted as a TCO decision.
fileciteturn10file17

------------------------------------------------------------------------

# 78. Source's Managed vs Self-Hosted Recommendation

The source's decision table says:

``` text
Managed:
Ops overhead → Low
Control → Less
Vendor lock-in → Yes

Self-hosted:
Ops overhead → High
Control → Full
Vendor lock-in → Lower
```

Its stated verdict is:

> Start with serverless and consider self-hosting at very large scale or
> when there are strict on-prem/GPU-local requirements.
> fileciteturn11file5

Treat that as the **source's recommendation**, not an absolute industry
rule.

------------------------------------------------------------------------

# 79. Vector Database Selection

The source provides a decision tree.

## \< 100K vectors

If already using PostgreSQL:

``` text
pgvector
```

For prototyping:

``` text
Chroma
```

------------------------------------------------------------------------

## Larger than 100K

Ask:

``` text
Do I want managed?
```

If yes:

``` text
Cloud-first
   ↓
Pinecone
```

or:

``` text
Qdrant Cloud / Zilliz
```

------------------------------------------------------------------------

## Self-hosted

If enterprise/distributed requirements:

``` text
Milvus on Kubernetes
```

Otherwise:

``` text
Qdrant
Weaviate
```

The source's exact selection framework is shown in its decision tree.
fileciteturn11file5

------------------------------------------------------------------------

# 80. Evaluation Criteria

The source recommends evaluating:

### Scale

``` text
How many vectors today?
How many next year?
```

### Latency

``` text
What is P99 target?
```

### Operations

``` text
Can our team operate this?
```

### Cost

``` text
What is the budget?
```

### Features

``` text
Hybrid search?
Multimodal?
Metadata filtering?
```

### Lock-in

``` text
Do we need open source?
```

This is a better selection process than:

> "Which vector database is best?"

There is no universally best vector database.

------------------------------------------------------------------------

# 81. Proof of Concept

Before committing, the source recommends testing:

``` text
Representative data volume
Target QPS
Query latency
Metadata filtering
Update/delete
Failure recovery
Monitoring
TCO
```

This is extremely important.

You should not choose a vector database based only on:

``` text
benchmark website
```

or:

``` text
developer popularity
```

because workload characteristics matter.

------------------------------------------------------------------------

# 82. Interview Question: Pinecone vs Self-Hosted

A strong answer from the source's reasoning:

### Choose managed when:

``` text
Small team
+
Fast delivery
+
Limited ops capacity
+
Moderate scale
```

### Choose self-hosted when:

``` text
Strong Kubernetes/SRE team
+
Cost sensitivity
+
Compliance
+
Data control
+
Vendor lock-in concerns
```

The source gives these exact trade-offs. fileciteturn11file19

------------------------------------------------------------------------

# 83. Interview Question: When Not to Use HNSW?

The source gives several cases:

### Very small dataset

``` text
< 10K vectors
```

Brute force may be simpler.

### Memory constrained

HNSW uses additional graph memory.

### Exact search required

HNSW is approximate.

### Heavy updates + tight latency

Frequent index modifications can create operational/performance
challenges.

Alternatives listed by the source:

``` text
IVF-PQ
DiskANN
Flat
LSH
```

fileciteturn11file12

------------------------------------------------------------------------

# 84. Interview Question: HNSW vs DiskANN

A strong answer:

> I would choose DiskANN when the HNSW memory footprint becomes too
> expensive or doesn't fit comfortably on the available RAM. DiskANN
> keeps much of the graph on NVMe SSDs, reducing RAM requirements
> substantially. I would accept some additional latency in exchange for
> lower infrastructure cost, especially for very large datasets where
> pure in-memory HNSW becomes expensive.

This follows the source's stated reasoning. fileciteturn11file12

------------------------------------------------------------------------

# 85. Interview Question: Why Is Metadata Filtering Difficult?

Strong answer:

> Vector search indexes such as HNSW are optimized for geometric
> navigation. A restrictive metadata filter adds another constraint to
> the traversal. If we first retrieve nearest neighbors and then filter,
> we can lose recall or even return too few results. Pre-filtering
> solves the correctness issue but makes graph traversal more
> complicated because only nodes satisfying the filter can participate.
> Therefore metadata filtering can become a significant latency
> bottleneck.

This is directly aligned with the source. fileciteturn11file12

------------------------------------------------------------------------

# 86. Interview Question: Multi-Tenancy

Strong answer:

> I would normally start with metadata filtering using a tenant ID
> because it is operationally simple. For stronger isolation or
> high-security requirements, I would use separate collections or
> namespaces. I would never retrieve unrestricted results and
> post-filter them because that creates both retrieval-quality and
> potential data-isolation problems.

The source explicitly recommends these three approaches and warns
against post-filtering. fileciteturn11file12

------------------------------------------------------------------------

# 87. The Most Important System-Design Mental Model

Do not think:

``` text
Vector DB = database that stores embeddings
```

Think:

``` text
Embedding Model
      ↓
Vector
      ↓
Vector Index
      ↓
Candidate Retrieval
      ↓
Metadata Filtering
      ↓
Hybrid / Multi-Vector Retrieval
      ↓
Reranking
      ↓
Top K
```

Then production adds:

``` text
Scaling
Replication
Monitoring
Capacity Planning
Cost
Security
Multi-tenancy
```

That is the real role of a vector database in an AI system.

------------------------------------------------------------------------

# 88. How It Fits Into RAG

Since you are studying RAG, connect the chapters like this:

``` text
                    RAG

User Query
    ↓
Query Embedding
    ↓
Vector Database
    ↓
ANN Search
    ↓
Metadata Filtering
    ↓
Hybrid Retrieval
    ↓
Reranking
    ↓
Relevant Chunks
    ↓
LLM
    ↓
Answer
```

The vector database is therefore **one major subsystem of RAG**, not the
entire RAG architecture.

Your repository's RAG chapter explicitly points to Vector Databases as a
deeper supporting topic. fileciteturn10file16

------------------------------------------------------------------------

# 89. What You Should Actually Remember for Interviews

You do NOT need to memorize every database vendor.

Prioritize:

## Tier 1 --- Must Know

``` text
Vector
Embedding
Similarity
Cosine
ANN
HNSW
Recall vs latency
Metadata filtering
Multi-tenancy
```

## Tier 2 --- Important

``` text
IVF
PQ
DiskANN
Hybrid search
RRF
Capacity planning
P99
Replication
```

## Tier 3 --- Architecture Decision

``` text
Pinecone
Qdrant
Weaviate
Milvus
pgvector
Chroma
```

Know:

``` text
What they are
When you would choose them
Trade-offs
```

rather than memorizing marketing details.

------------------------------------------------------------------------

# 90. The Interview Chain You Should Practice

If the interviewer asks:

> "Design a RAG system."

You can walk through:

``` text
1. Documents
      ↓
2. Chunking
      ↓
3. Embedding model
      ↓
4. Vector storage
      ↓
5. ANN index
      ↓
6. Metadata
      ↓
7. Retrieval
      ↓
8. Hybrid search if needed
      ↓
9. Reranking
      ↓
10. LLM
```

Then the interviewer asks:

> "We now have 100M vectors."

You answer:

``` text
Brute force
    ↓
No longer ideal

Consider:
HNSW / DiskANN / IVF-PQ
```

Then:

> "RAM is too expensive."

You answer:

``` text
DiskANN
or
IVF-PQ
```

Then:

> "We have 10,000 tenants."

You answer:

``` text
Metadata filtering
or
namespaces / collections
```

Then:

> "Filtering makes latency bad."

You discuss:

``` text
Pre-filtering
index strategy
filter selectivity
bitmaps/SIMD
benchmarking
```

Then:

> "P99 is 500ms."

You investigate:

``` text
ef_search
index type
filter cost
dimensions
QPS
replicas
hardware
cache
```

That is **AI system-design thinking**.

------------------------------------------------------------------------

# 91. One-Page Revision Sheet

``` text
VECTOR DATABASE
──────────────────────────────────────

Stores:
    embeddings + metadata

Main operation:
    nearest-neighbor search


EXACT SEARCH
──────────────────────────────────────

Compare query with every vector

Complexity:
    O(n × d)

Recall:
    100%

Good for:
    small datasets


ANN
──────────────────────────────────────

Use an index to avoid searching everything

Benefits:
    latency ↓
    compute ↓

Trade-off:
    recall can ↓


METRICS
──────────────────────────────────────

Cosine
    → common for text

L2
    → geometric distance

Dot product
    → useful with normalized vectors


HNSW
──────────────────────────────────────

Graph + hierarchy

Search:
    top layer
      ↓
    greedy navigation
      ↓
    lower layer
      ↓
    bottom layer

Pros:
    fast
    high recall
    updates

Cons:
    RAM hungry

Parameters:
    M
    ef_construction
    ef_search


IVF
──────────────────────────────────────

k-means
   ↓
centroids
   ↓
clusters
   ↓
search selected clusters

Parameters:
    nlist
    nprobe


PQ
──────────────────────────────────────

Compress vectors

Memory ↓
Speed ↑
Recall may ↓


DiskANN
──────────────────────────────────────

Graph on SSD

RAM ↓↓↓
Cost ↓
Latency slightly ↑


FILTERING
──────────────────────────────────────

Vector similarity
       +
Metadata

Never blindly:
    retrieve everything
       ↓
    post-filter


HYBRID
──────────────────────────────────────

Dense semantic
       +
BM25 lexical
       ↓
RRF
       ↓
Top K


PRODUCTION
──────────────────────────────────────

Capacity
Replication
Monitoring
P50
P99
QPS
Recall
Upserts
Deletes
TCO


SELECTION
──────────────────────────────────────

Small + PostgreSQL
    → pgvector

Prototype
    → Chroma

Managed
    → Pinecone / Qdrant Cloud / Zilliz

Self-hosted
    → Qdrant / Weaviate / Milvus

Huge + RAM constrained
    → DiskANN / IVF-PQ
```

------------------------------------------------------------------------

# 92. Final Takeaway

The entire chapter can be reduced to one engineering problem:

> **How do I retrieve the most relevant vectors quickly, accurately,
> securely, and cheaply as the dataset and traffic grow?**

Everything in the chapter answers one part of that question:

``` text
Vector DB
    ↓
stores vectors

Cosine / L2 / Dot
    ↓
defines similarity

ANN
    ↓
avoids brute force

HNSW
    ↓
fast in-memory graph search

IVF
    ↓
cluster-based search

PQ
    ↓
compression

DiskANN
    ↓
SSD-based large-scale search

Metadata filtering
    ↓
correct constrained retrieval

Hybrid search
    ↓
semantic + keyword retrieval

Multi-vector
    ↓
multiple retrieval perspectives

Replication
    ↓
availability + read scaling

Monitoring
    ↓
operational visibility

Managed vs self-hosted
    ↓
TCO decision
```

And for your **AI Engineer interview**, the most important pattern is:

``` text
Requirement
    ↓
Choose retrieval strategy
    ↓
Choose index
    ↓
Choose filtering strategy
    ↓
Estimate memory / storage
    ↓
Estimate latency / QPS
    ↓
Add replicas
    ↓
Monitor recall + P99
    ↓
Re-evaluate cost
```

That is the level at which you should connect **Vector Databases → RAG →
AI System Design**.

------------------------------------------------------------------------

## Source Boundary

This note is based on the uploaded `04-vector-databases.md`. Claims
about HNSW, DiskANN, IVF/PQ, metadata filtering, query patterns,
production operations, vendor comparisons, TCO, and interview answers
above are grounded in that source.
fileciteturn10file14turn10file10turn10file8turn11file5

Where the source gives approximate values---such as recall percentages,
memory multipliers, QPS heuristics, example cloud pricing, or scale
thresholds---those values are presented here as **indicative examples
from the source**, not universal guarantees. The source itself labels
some pricing as indicative and says provider pricing should be verified.
fileciteturn11file17
