# 01 — Inference Fundamentals: Detailed Interview-Focused Notes

> Based only on `01-inference-fundamentals.md`, expanded into easier-to-understand interview notes.

---

## 1. What is Inference?

Inference means **using a trained model to produce predictions/output**.

```text
Training
   ↓
Learn model parameters
   ↓
Trained model
   ↓
Inference
   ↓
Generate output
```

For an LLM, inference means generating tokens.

Example:

```text
Input:
"RAG stands for"

Output:
"Retrieval"
"Augmented"
"Generation"
```

The important point is that LLM inference has two major phases:

```text
LLM Inference
     |
     +-----------+-----------+
     |                       |
  Prefill                  Decode
     |                       |
Process input             Generate output
```

---

# 2. Prefill

**Prefill** is the phase where the model processes the entire input prompt before generating the first output token.

Example:

```text
User prompt:

"Explain how RAG works in detail."

        ↓

Tokenizer

        ↓

Input tokens

        ↓

Transformer

        ↓

Ready to generate
```

If the prompt contains 10,000 tokens, Prefill processes those 10,000 input tokens.

### Important characteristic

Prefill is **highly parallelizable**.

GPUs are very good at performing large matrix operations over many values simultaneously.

Conceptually:

```text
Token 1
Token 2
Token 3
...
Token N
   ↓
Large matrix operations
   ↓
GPU
```

Therefore, Prefill is generally **compute-bound**.

---

# 3. What Does Compute-Bound Mean?

A workload is compute-bound when the limiting factor is the amount of mathematical computation the hardware must perform.

Conceptually:

```text
Data is available
      ↓
GPU performs calculations
      ↓
Calculation takes time
```

For LLM Prefill:

```text
Large prompt
    ↓
Large matrix multiplications
    ↓
Lots of GPU computation
    ↓
Compute-bound
```

The GPU's arithmetic/computation capability is the important resource.

---

# 4. Why is Prefill Parallel?

Suppose the input contains many tokens:

```text
X1
X2
X3
...
X1000
```

Transformer operations can process these representations using large matrix operations.

Modern GPUs are designed to perform these operations in parallel.

Conceptually:

```text
X1  X2  X3  X4 ... X1000
          ↓
    Matrix operations
          ↓
         GPU
```

This is why Prefill can make high use of GPU compute resources.

---

# 5. Decode

After Prefill, the model starts generating the answer.

Unlike Prefill, output generation is **autoregressive**.

Example:

```text
Prompt
  ↓
Token 1
  ↓
Token 2
  ↓
Token 3
  ↓
Token 4
  ↓
...
```

Each generated token depends on the previous context.

Therefore:

```text
Decode
   ↓
Sequential token generation
```

This makes Decode fundamentally different from Prefill.

---

# 6. Why Can't the Model Generate All Output Tokens at Once?

Suppose the model generates:

```text
"The capital of India is New Delhi."
```

The probability of a later token depends on the previous tokens.

Conceptually:

```text
P(token₂ | token₁)

P(token₃ | token₁, token₂)

P(token₄ | token₁, token₂, token₃)
```

Therefore the model needs to generate tokens autoregressively.

```text
Token 1
   ↓
Token 2
   ↓
Token 3
   ↓
Token 4
```

This sequential nature is why Decode has much less parallelism than Prefill.

---

# 7. Why is Decode Memory-Bound?

This is one of the most important concepts in the chapter.

Consider a large LLM.

For example:

```text
70B parameters
```

The model's weights occupy a large amount of GPU memory.

During Decode, the model repeatedly needs to access the model weights and other relevant data to generate each next token.

The limiting factor can therefore become:

```text
How quickly can data move
between GPU memory and the compute units?
```

That is **memory bandwidth**.

Conceptually:

```text
GPU Memory
    ↓
GPU compute
    ↓
Calculate next token
    ↓
GPU Memory
    ↓
Next token
```

If data cannot be supplied to the compute units quickly enough, the compute hardware spends time waiting for data.

This is the **memory wall** problem.

---

# 8. What is Memory Bandwidth?

Memory bandwidth describes how quickly data can be transferred between memory and the compute hardware.

Think of GPU memory as a warehouse.

```text
             GPU
              ↑
              |
       Memory bandwidth
              |
              ↓
             VRAM
```

A GPU can have enormous computational capability, but if the required data cannot be moved quickly enough, computation cannot be fully utilized.

So for Decode:

```text
Large model
    ↓
Repeated memory access
    ↓
Large data movement
    ↓
Memory bandwidth becomes important
```

---

# 9. Simple Compute-Bound vs Memory-Bound Analogy

### Compute-bound

Imagine a chef who has all ingredients ready but is busy cooking.

```text
Ingredients → Chef → Cooking
```

The chef's ability to cook is the bottleneck.

### Memory-bound

Now imagine an extremely fast chef who keeps waiting for ingredients.

```text
Chef
 ↓
"Give me ingredients"
 ↓
Wait
 ↓
Ingredients arrive
 ↓
Cook
 ↓
"Give me more"
 ↓
Wait
```

The cooking ability isn't the main problem.

Getting the data to the chef is.

That is the intuition behind a memory-bound workload.

---

# 10. Prefill vs Decode

This is the most important table from the chapter.

| | Prefill | Decode |
|---|---|---|
| Purpose | Process input prompt | Generate output tokens |
| Parallelism | High | Low / sequential |
| Typical bottleneck | Compute | Memory bandwidth |
| Important metric | TTFT | TPOT |
| Important optimizations | FlashAttention, FP8, Tensor Parallelism, Prefix Caching | Quantization, GQA, Batching, Speculative Decoding |

Remember:

```text
Prefill → Compute
Decode  → Memory
```

This is a simplified but extremely useful interview mental model.

---

# 11. TTFT — Time To First Token

**TTFT** means:

> The time between sending the request and receiving the first generated token.

Example:

```text
User sends request
       ↓
     100 ms
       ↓
First token appears
```

Then:

```text
TTFT = 100 ms
```

TTFT is strongly influenced by the Prefill phase.

Therefore:

```text
Longer prompt
    ↓
More Prefill work
    ↓
Potentially higher TTFT
```

### Why TTFT matters

TTFT affects perceived responsiveness.

A system that produces the first token quickly feels responsive even if the entire answer takes longer to finish.

---

# 12. TPOT — Time Per Output Token

**TPOT** means the time required to generate each additional output token.

Example:

```text
Token 1 → 20 ms
Token 2 → 20 ms
Token 3 → 20 ms
Token 4 → 20 ms
```

Then approximately:

```text
TPOT ≈ 20 ms/token
```

Lower TPOT means faster streaming generation.

TPOT is mainly associated with the Decode phase.

Therefore:

```text
TPOT
  ↓
Decode performance
  ↓
Memory bandwidth becomes important
```

---

# 13. TTFT vs TPOT

Suppose:

```text
Request sent
    ↓
500 ms
    ↓
First token
    ↓
20 ms
    ↓
Next token
    ↓
20 ms
    ↓
Next token
```

Then:

```text
TTFT = 500 ms
TPOT ≈ 20 ms/token
```

These represent different parts of the user experience.

### If TTFT is high

Look at:

```text
Prefill
```

### If TPOT is high

Look at:

```text
Decode
```

This distinction is very important in system-design interviews.

---

# 14. Throughput

Throughput measures how much work the system can process per unit of time.

For LLM serving, an important form is:

```text
tokens / second
```

Example:

```text
10,000 tokens/sec
```

Throughput becomes especially important when serving many users simultaneously.

You are not only asking:

> "How fast is one request?"

You are also asking:

> "How many requests/tokens can my infrastructure serve?"

---

# 15. End-to-End Latency

Total request latency includes:

```text
Time to first token
+
Time required to generate the remaining tokens
```

Conceptually:

```text
Total latency
≈
TTFT + Decode time
```

Example:

```text
TTFT = 200 ms

100 output tokens
TPOT = 20 ms/token

Approximate total:
200 + (100 × 20)
= 2200 ms
```

The exact measurement can vary depending on how the serving system defines the metrics, but the relationship is the important part.

---

# 16. FP8

The chapter introduces **FP8**, an 8-bit floating-point representation.

You may already know:

```text
FP32
FP16
BF16
```

FP8 uses fewer bits per value.

Conceptually:

```text
FP16
████████████████

FP8
████████
```

Fewer bits can mean:

```text
Memory usage ↓
Data movement ↓
Potential compute efficiency ↑
```

The chapter discusses FP8 particularly in the context of modern NVIDIA inference hardware.

---

# 17. Why Does Lower Precision Help Inference?

Suppose some model data occupies:

```text
100 GB in FP16
```

Moving to an appropriate 8-bit representation can approximately halve the storage for those values:

```text
FP16 → ~100 GB
FP8  → ~50 GB
```

This can reduce:

```text
Memory footprint
+
Memory traffic
```

That is especially relevant for Decode, where memory bandwidth is an important bottleneck.

---

# 18. FP8 vs INT8

Both use 8 bits, but they represent numbers differently.

### INT8

Integer representation:

```text
-128 ... +127
```

### FP8

Floating-point representation containing:

```text
Sign
+
Exponent
+
Mantissa
```

FP8 therefore provides a floating-point representation with a different balance between range and precision.

The chapter highlights FP8 as useful for LLM workloads, particularly for activations.

---

# 19. Dynamic FP8 Scaling

Different parts of a model can have different activation ranges.

For example:

```text
Layer 1 → small values
Layer 2 → large values
Layer 3 → medium values
```

Using appropriate scaling can help represent those values effectively in lower precision.

Conceptually:

```text
Layer 1 → Scale A
Layer 2 → Scale B
Layer 3 → Scale C
```

The chapter describes dynamic/per-layer scaling as a more advanced inference optimization for handling varying activation ranges and outliers.

For an interview, understand the principle rather than memorizing hardware-specific implementation details.

---

# 20. Why is LLM Generation Slower Than Classification?

Classification typically looks like:

```text
Input
  ↓
Model
  ↓
Prediction
```

One forward pass can produce the classification result.

LLM generation looks like:

```text
Input
  ↓
Token 1
  ↓
Token 2
  ↓
Token 3
  ↓
Token 4
  ↓
...
```

Because generation is autoregressive:

```text
Classification
→ largely one forward computation

Generation
→ repeated Decode steps
```

And Decode is commonly memory-bandwidth-bound.

Therefore, generating a long answer can be substantially more expensive than producing a single classification output.

---

# 21. Optimizing TTFT

If TTFT is too high, focus primarily on the Prefill side.

The chapter mentions:

## FlashAttention

Optimizes attention computation and memory access.

Conceptually:

```text
Attention
   ↓
More efficient implementation
   ↓
Better memory behavior
   ↓
Faster Prefill
```

## Tensor Parallelism

Split model computation across multiple GPUs.

```text
             Model
               |
       +-------+-------+
       |       |       |
      GPU1   GPU2    GPU3
       |       |       |
       +-------+-------+
```

This can accelerate computation by distributing work.

## Prefix Caching

If many requests share the same prefix:

```text
System prompt
+
Common instructions
+
User-specific question
```

the shared prefix can potentially be cached.

Conceptually:

```text
Common prefix
     ↓
Compute once
     ↓
Cache
     ↓
Reuse
```

This can reduce repeated Prefill work and therefore improve TTFT.

---

# 22. Optimizing TPOT

If TPOT is too high, focus on Decode.

The chapter mentions:

## Quantization

Reduce model precision:

```text
FP16
  ↓
INT8 / lower-bit representation
```

Smaller representations can reduce memory traffic.

Conceptually:

```text
Weights smaller
      ↓
Less data moved
      ↓
Less memory pressure
      ↓
Potentially lower TPOT
```

## GQA — Grouped Query Attention

GQA uses fewer KV heads than Query heads.

Conceptually:

```text
Many Q heads
      ↓
Fewer KV heads
      ↓
Smaller KV representation
      ↓
Lower memory pressure
```

This connects directly to the KV-head concepts you have already studied.

## Batching

Multiple requests can be processed together to improve hardware utilization.

## Speculative Decoding

A smaller/draft model proposes multiple tokens:

```text
Draft model
    ↓
A B C D
```

The larger model then verifies those proposed tokens.

```text
Draft
  ↓
A B C D
  ↓
Large model verifies
  ↓
Accept valid tokens
```

If many proposed tokens are accepted, the expensive large model can generate more useful output per iteration.

---

# 23. Putting Everything Together

```text
                    LLM INFERENCE
                         |
              +----------+----------+
              |                     |
           PREFILL                DECODE
              |                     |
       Process prompt        Generate tokens
              |                     |
       Highly parallel        Sequential
              |                     |
       COMPUTE-BOUND          MEMORY-BOUND
              |                     |
       +------+-------+       +-----+------+
       |              |       |            |
 FlashAttention     FP8   Quantization    GQA
 Tensor Parallelism       Batching        Speculative
 Prefix Caching
```

---

# 24. Connection With KV Cache and GQA

The chapter's Decode optimization discussion connects directly with the LLM internals you've already studied.

### GQA

```text
Fewer KV heads
      ↓
Smaller KV cache
      ↓
Less memory required
      ↓
Less memory pressure during Decode
```

### Quantization

```text
Smaller weights
      ↓
Less data to move
      ↓
Lower memory bandwidth pressure
      ↓
Potentially faster Decode
```

This is why understanding model architecture helps with AI system design.

---

# 25. The Interview Mental Model

When an interviewer asks about inference performance, start by asking:

```text
Is the problem Prefill or Decode?
```

Then:

```text
                 Performance Problem
                         |
                +--------+--------+
                |                 |
             Prefill            Decode
                |                 |
          Compute-bound       Memory-bound
                |                 |
             TTFT              TPOT
                |                 |
        Optimize compute     Optimize memory
```

This is a much better answer than simply saying:

> "Use a faster GPU."

---

# 26. Interview Questions

## Q1. Why is LLM inference expensive?

**Answer:**

LLM inference has a Prefill phase and an autoregressive Decode phase. Prefill processes the prompt in parallel and is generally compute-bound. Decode generates tokens sequentially and is often memory-bandwidth-bound because the system repeatedly accesses large model weights and associated state.

---

## Q2. What is the difference between Prefill and Decode?

**Answer:**

Prefill processes the input prompt before generation and has high parallelism. Decode generates output tokens autoregressively, one token at a time, and therefore has much less parallelism.

---

## Q3. What is TTFT?

**Answer:**

Time To First Token — the time from request submission until the first generated token is returned.

---

## Q4. What is TPOT?

**Answer:**

Time Per Output Token — the time required to generate each additional output token during Decode.

---

## Q5. TTFT is high. What would you optimize?

Think:

```text
High TTFT
   ↓
Prefill problem
   ↓
FlashAttention
Tensor Parallelism
Prefix Caching
FP8
```

---

## Q6. TPOT is high. What would you optimize?

Think:

```text
High TPOT
   ↓
Decode problem
   ↓
Quantization
GQA
Batching
Speculative Decoding
```

---

## Q7. Why is Decode memory-bound?

**Answer:**

Decode generates tokens sequentially and repeatedly accesses model weights and other model state. The amount of data that needs to move through GPU memory can become the limiting factor, so memory bandwidth rather than raw arithmetic throughput often dominates.

---

## Q8. Why does quantization help Decode?

**Answer:**

Quantization reduces the size of model representations. Smaller representations require less memory and can reduce the amount of data that needs to move from GPU memory during Decode.

---

# 27. What You Should Memorize

### Core concepts

```text
Inference
= using a trained model

Prefill
= process input

Decode
= generate output tokens
```

### Bottlenecks

```text
Prefill
→ Compute-bound

Decode
→ Memory-bandwidth-bound
```

### Metrics

```text
TTFT
→ Time To First Token
→ mainly Prefill-related

TPOT
→ Time Per Output Token
→ mainly Decode-related
```

### Optimizations

```text
Prefill:
→ FlashAttention
→ Tensor Parallelism
→ Prefix Caching
→ FP8

Decode:
→ Quantization
→ GQA
→ Batching
→ Speculative Decoding
```

---

# 28. Final Cheat Sheet

```text
LLM INFERENCE
│
├── PREFILL
│   ├── Process input prompt
│   ├── Highly parallel
│   ├── Generally compute-bound
│   ├── Affects TTFT
│   └── Optimizations
│       ├── FlashAttention
│       ├── Tensor Parallelism
│       ├── Prefix Caching
│       └── FP8
│
└── DECODE
    ├── Generate output
    ├── Autoregressive
    ├── Sequential
    ├── Generally memory-bound
    ├── Affects TPOT
    └── Optimizations
        ├── Quantization
        ├── GQA
        ├── Batching
        └── Speculative Decoding
```

## One sentence to remember

> **Prefill processes the prompt and is generally compute-bound, while Decode generates tokens sequentially and is generally memory-bandwidth-bound; therefore TTFT and TPOT require different optimization strategies.**
