# 05 — PagedAttention: Detailed Interview Notes

## 1. What problem is PagedAttention solving?

The central problem is:

> How do we efficiently store and manage the KV Cache for many simultaneous LLM requests?

During generation, every request needs KV-cache memory. With many users generating responses simultaneously, the problem is not only whether VRAM is sufficient, but whether that VRAM is being used efficiently.

---

## 2. The Contiguous Memory Problem

Traditional serving approaches may allocate one large contiguous region for a request.

Suppose maximum sequence length is:

```text
8192 tokens
```

but a request actually generates only:

```text
10 tokens
```

Space may effectively be reserved for 8192 tokens even though only 10 are used.

This creates **internal fragmentation**.

---

## 3. Internal Fragmentation

Imagine reserving eight parking spaces for one car:

```text
🚗 🚗 🚗 🚗 🚗 🚗 🚗 🚗
```

but using only one:

```text
🚗
```

The rest is reserved but unused.

For LLM serving:

```text
Reserved: 8192-token capacity
Used:       10 tokens
```

So a large amount of reserved capacity is wasted.

---

## 4. External Fragmentation

Free GPU memory can also become split into small gaps:

```text
████   ██   █████   ██
```

You may have enough total free memory, but not one sufficiently large contiguous region for a new allocation.

This is **external fragmentation**.

---

## 5. Why This Is Bad for LLM Serving

Requests have different:

- prompt lengths
- output lengths
- completion times

For example:

```text
Request A → starts
Request B → starts
Request C → starts

Request A → finishes
Request D → starts

Request B → generates more tokens
Request E → starts
```

Memory requirements therefore change constantly.

Poor memory utilization can lead to:

```text
Fewer requests
    ↓
Smaller batches
    ↓
Lower GPU utilization
    ↓
Lower throughput
```

---

# 6. Core Idea of PagedAttention

PagedAttention takes inspiration from **virtual memory in operating systems**.

Instead of requiring one huge contiguous region, memory is divided into smaller pages/blocks.

For KV Cache:

```text
Tokens → Blocks
Logical memory → Physical memory
Block Table → Mapping
```

The model sees a logical sequence of blocks, while those blocks can be physically scattered in GPU memory.

---

# 7. KV Cache → Blocks

Suppose block size is:

```text
16 tokens
```

A 64-token sequence becomes:

```text
Block 0 → tokens 0–15
Block 1 → tokens 16–31
Block 2 → tokens 32–47
Block 3 → tokens 48–63
```

Therefore:

```text
64 tokens → 4 blocks
```

Each block has a fixed size.

---

# 8. Logical vs Physical Memory

The model can logically see:

```text
Block 0
Block 1
Block 2
Block 3
```

But physical VRAM may contain:

```text
Logical Block 0 → Physical Block 17
Logical Block 1 → Physical Block 4
Logical Block 2 → Physical Block 29
Logical Block 3 → Physical Block 8
```

So:

```text
Logical memory
      ↓
Block Table
      ↓
Physical memory
```

The blocks do not have to be physically adjacent.

---

# 9. Block Table

The **Block Table** maps logical KV-cache blocks to physical GPU-memory blocks.

Example:

```text
Logical Block       Physical Block

     0  ─────────────→   17
     1  ─────────────→    4
     2  ─────────────→   29
     3  ─────────────→    8
```

If the model needs logical Block 2:

```text
Block Table
     ↓
Physical Block 29
```

The Block Table is therefore the bridge between the model's logical contiguous view and scattered physical memory.

---

# 10. Dynamic Allocation

PagedAttention does not need to reserve maximum sequence length upfront.

Suppose:

```text
Block size = 16 tokens
```

and the request currently has 10 tokens.

It needs only the necessary block.

As the sequence grows:

```text
10 tokens
   ↓
16 tokens
   ↓
32 tokens
   ↓
48 tokens
```

additional blocks can be allocated as required.

Conceptually:

```text
Request starts
     ↓
Allocate Block 1
     ↓
Need more space
     ↓
Allocate Block 2
     ↓
Need more space
     ↓
Allocate Block 3
```

This is dynamic allocation.

---

# 11. LEGO Analogy

Traditional allocation:

```text
┌──────────────────────────────────┐
│          8192-token space        │
└──────────────────────────────────┘
```

PagedAttention:

```text
[Block 1]
```

Need more:

```text
[Block 1][Block 2]
```

Need more:

```text
[Block 1][Block 2][Block 3]
```

The physical blocks do not need to sit next to one another in VRAM.

The Block Table keeps track of their locations.

---

# 12. Block Manager

The **Block Manager** manages physical KV-cache blocks.

Think of it as a small memory-management system for GPU memory.

It is somewhat like a:

> mini operating system for GPU memory.

### Allocation

```text
Request
   ↓
Needs KV Cache
   ↓
Block Manager
   ↓
Find free physical blocks
   ↓
Assign blocks
```

For example:

```text
Free blocks:

[2] [5] [8] [11] [15]

Request A needs 3 blocks

Assign:

[2] [8] [15]
```

They do not have to be physically adjacent.

### Dynamic growth

As a request generates more tokens, the Block Manager allocates additional blocks.

Therefore:

```text
Memory usage ≈ actual KV-cache requirement
```

rather than:

```text
Memory usage ≈ maximum possible sequence length
```

### Freeing memory

When a request finishes:

```text
Request
   ↓
Completed
   ↓
KV Cache no longer needed
   ↓
Blocks released
   ↓
Blocks return to free pool
```

Those blocks can then be reused by new requests.

---

# 13. Paged Swap

If VRAM becomes full:

```text
GPU VRAM
████████████████████
        FULL
```

inactive KV blocks can potentially be moved to CPU RAM:

```text
GPU VRAM
   ↓
CPU RAM
```

and brought back when needed.

Conceptually:

```text
Active KV blocks
     ↓
GPU VRAM

Inactive KV blocks
     ↓
CPU RAM
```

This is another similarity to operating-system virtual memory.

---

# 14. KV Cache Sharing

Suppose 100 users share the same 5,000-token system prompt.

Without sharing:

```text
User 1 → 5000-token KV cache
User 2 → 5000-token KV cache
...
User 100 → 5000-token KV cache
```

That is:

```text
5000 × 100
=
500,000 tokens worth of KV cache
```

With prefix sharing:

```text
                 Shared 5000-token prefix
                         |
             +-----------+-----------+
             |           |           |
           User 1      User 2      User 3
             |           |           |
          Unique       Unique      Unique
          blocks       blocks      blocks
```

The shared prefix can be stored once, with individual continuations stored separately.

---

# 15. Copy-on-Write

If multiple requests share a block, one request cannot simply modify that shared block.

Instead, **Copy-on-Write (CoW)** allows a private copy to be created only when needed.

Before:

```text
        Shared Block
       /     |           U1     U2      U3
```

After User 1 needs a private version:

```text
        Shared Block
        /              U2          U3

      U1
       ↓
   Private Block
```

The shared block remains unchanged.

---

# 16. Why Prefix Sharing Matters

Many requests can share common prefixes:

### System prompts

```text
"You are an assistant..."
```

### Few-shot examples

```text
Example 1...
Example 2...
Example 3...
```

### Long common instructions

```text
Company policies...
Product documentation...
Agent instructions...
```

Instead of storing the same KV cache repeatedly, it can be shared.

---

# 17. Why Does PagedAttention Increase Throughput?

The key chain is:

```text
PagedAttention
      ↓
Less memory fragmentation
      ↓
More efficient VRAM usage
      ↓
More requests fit simultaneously
      ↓
Larger batch size
      ↓
Better GPU utilization
      ↓
Higher aggregate tokens/sec
```

This is the core interview explanation.

---

# 18. Why Does Batch Size Matter?

If traditional serving can fit only a few requests but PagedAttention allows many more requests to fit into the same VRAM, the GPU can process more requests together.

Therefore:

```text
Better memory utilization
        ↓
Larger batches
        ↓
Higher GPU utilization
        ↓
Higher throughput
```

---

# 19. PagedAttention Does NOT Make the Model Smaller

PagedAttention is primarily a **KV-cache memory-management technique**.

It does not compress the model weights.

Think of inference memory as:

```text
Model weights
      +
KV Cache
      +
Runtime memory
```

PagedAttention mainly addresses efficient management of the KV-cache portion.

It should not be described as a model-compression technique.

---

# 20. PagedAttention vs Quantization

### Quantization

Changes numerical representation:

```text
FP16
 ↓
INT8 / 4-bit
```

Goal:

```text
Reduce model memory / computation cost
```

### PagedAttention

Changes KV-cache memory allocation:

```text
Large contiguous allocation
       ↓
Small dynamically allocated blocks
```

Goal:

```text
Reduce memory fragmentation
+
Improve KV-cache utilization
```

They can be used together:

```text
Quantization
      +
PagedAttention
      ↓
More efficient inference
```

---

# 21. PagedAttention vs KV Cache

These are not the same thing.

### KV Cache

Stores previously computed:

```text
Keys
+
Values
```

so they do not have to be recomputed during every Decode step.

### PagedAttention

Provides an efficient way to organize and manage those KV-cache blocks in GPU memory.

Mental model:

```text
KV Cache
= WHAT is stored

PagedAttention
= HOW that cache is efficiently organized/accessed
```

---

# 22. PagedAttention vs GQA

### GQA

Changes the attention architecture:

```text
Many Query heads
      ↓
Fewer KV heads
```

Result:

```text
Smaller KV cache
```

### PagedAttention

Changes memory management:

```text
KV cache
      ↓
Small physical blocks
      ↓
Dynamic allocation
```

Therefore:

```text
GQA
→ reduce KV-cache size

PagedAttention
→ manage KV-cache memory efficiently
```

They solve different but complementary problems.

---

# 23. Connecting Your Previous Topics

Your recent concepts now fit together:

```text
GQA
 ↓
Fewer KV heads
 ↓
Smaller KV Cache
```

Then:

```text
PagedAttention
 ↓
KV Cache divided into blocks
 ↓
Efficient memory allocation
 ↓
Less fragmentation
```

Then:

```text
Better KV memory utilization
 ↓
More requests fit in VRAM
 ↓
Larger batches
 ↓
Better GPU utilization
 ↓
Higher throughput
```

Full picture:

```text
                 LLM Decode
                     |
              Uses KV Cache
                     |
           +---------+---------+
           |                   |
          GQA            PagedAttention
           |                   |
    Smaller KV Cache      Better memory
                          management
           |                   |
           +---------+---------+
                     |
               More efficient
                 inference
                     |
                Throughput ↑
```

---

# 24. Operating-System Analogy

This is one of the best ways to remember PagedAttention.

### Traditional memory allocation

```text
Process A
████████████████████
```

requires a contiguous region.

### Virtual memory

```text
Process A

Page 1 → physical location 10
Page 2 → physical location 3
Page 3 → physical location 17
Page 4 → physical location 8
```

The process sees a logical contiguous address space, while the OS maps it to scattered physical memory.

### PagedAttention

```text
Logical KV blocks
        ↓
Block Table
        ↓
Scattered physical VRAM blocks
```

This is why the name PagedAttention makes sense.

---

# 25. Interview Answer — What Is PagedAttention?

> PagedAttention is a KV-cache memory-management technique inspired by OS virtual memory. It divides the KV cache into fixed-size blocks and maps logical blocks to non-contiguous physical GPU-memory blocks using a Block Table. This reduces memory fragmentation, enables dynamic allocation and prefix sharing, allows more requests to fit in VRAM, and improves inference throughput.

---

# 26. Interview Answer — Explain the Block Table

> The Block Table maps logical KV-cache blocks to physical GPU-memory blocks. The model can treat the sequence as logically contiguous, while the underlying KV-cache blocks can be scattered throughout VRAM. This allows dynamic allocation and freeing of small blocks and enables features such as prefix sharing and efficient memory utilization.

Keywords:

```text
Logical blocks
Physical blocks
Mapping
Dynamic allocation
KV Cache
```

---

# 27. Interview Answer — Why Does PagedAttention Increase Throughput?

> PagedAttention reduces internal and external memory fragmentation by allocating KV Cache in small fixed-size blocks instead of reserving large contiguous regions. This allows more requests to fit into the same GPU VRAM, enabling larger batches and better GPU utilization. As a result, aggregate tokens-per-second throughput can increase significantly.

Remember:

```text
Fragmentation ↓
      ↓
Memory utilization ↑
      ↓
Batch size ↑
      ↓
GPU utilization ↑
      ↓
Throughput ↑
```

---

# 28. What You Should Memorize

### PagedAttention
Efficient KV-cache memory management using fixed-size blocks.

### Block
A small fixed-size unit of KV-cache memory, such as 16 tokens in the example.

### Logical memory
The model's view of the sequence as a contiguous set of blocks.

### Physical memory
The actual locations of those blocks in GPU VRAM.

### Block Table
Maps logical blocks to physical blocks.

### Block Manager
Allocates, manages, and frees physical KV-cache blocks.

### Copy-on-Write
Allows shared prefix blocks to remain shared until a request needs a private copy.

### Paged Swap
Moves inactive KV blocks between GPU VRAM and CPU RAM when needed.

---

# 29. Final Mental Model

```text
                    REQUEST
                       |
                       ↓
                    KV CACHE
                       |
                Split into blocks
                       |
        +--------------+--------------+
        |              |              |
     Block 0        Block 1        Block 2
        |              |              |
        +--------------+--------------+
                       |
                  Block Table
                       |
        +--------------+--------------+
        |              |              |
    Physical        Physical       Physical
     Block 17        Block 4        Block 29
        |              |              |
        +--------------+--------------+
                       ↓
                    GPU VRAM
```

The model thinks:

```text
Block 0 → Block 1 → Block 2
```

But VRAM can actually contain:

```text
Block 17 → Block 4 → Block 29
```

The **Block Table connects the two worlds**.

---

# 30. Most Important Connection

```text
                 LLM INFERENCE
                       |
                    DECODE
                       |
                    KV CACHE
                       |
          +------------+------------+
          |                         |
         GQA                 PagedAttention
          |                         |
   KV cache smaller          KV cache managed
                             in blocks
          |                         |
          +------------+------------+
                       |
               VRAM utilization ↑
                       |
               Batch size ↑
                       |
               GPU utilization ↑
                       |
                Throughput ↑
```

### One sentence to remember

> **PagedAttention treats the KV Cache like virtual memory: it splits the cache into small blocks, maps logical blocks to scattered physical VRAM blocks through a Block Table, and thereby reduces fragmentation so more requests can fit into GPU memory and be served in larger batches.**

---

## Quick Interview Revision

**What is PagedAttention?**

> A KV-cache memory-management technique inspired by OS virtual memory.

**Is PagedAttention the same as KV Cache?**

> No. KV Cache is the cached Key and Value tensors; PagedAttention is a technique for efficiently managing that cache in memory.

**Is PagedAttention quantization?**

> No. Quantization reduces numerical precision and model-weight memory. PagedAttention manages KV-cache memory allocation.

**Why does it improve throughput?**

> Better KV-cache memory utilization allows more concurrent requests and larger batches, improving GPU utilization and aggregate throughput.
