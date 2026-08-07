# LLM Internals — My Notes (Part 1)

> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, backend analogies, production notes, and interview explanations.

---

# Table of Contents (Part 1)

1. Why Study LLM Internals?
2. The Evolution of NLP Models
3. Why RNNs Failed
4. Why Transformers Changed Everything
5. High-Level Transformer Architecture
6. Encoder vs Decoder
7. Why GPT Uses Decoder Only
8. Why This Matters for AI Engineers
9. Interview Questions
10. Key Takeaways

---

# 1. Why Study LLM Internals?

When people first start learning AI, they often jump directly into:

- LangChain
- RAG
- Agents
- MCP
- Vector Databases

Those are important.

But they all assume one thing:

You already understand how an LLM works.

Think of it this way.

Learning RAG before Transformers is like

learning Redis

before understanding

RAM.

You can use it,

but you won't fully understand

why it exists.

---

The repository begins with the core building blocks of modern LLMs because almost every production decision depends on them. :contentReference[oaicite:0]{index=0}

Examples

When should you use

```
128K Context?
```

Depends on

```
RoPE
KV Cache
Attention
```

---

Why are some models

```
Fast
```

while others are

```
Slow
```

Depends on

```
MoE
Attention
KV Cache
```

---

Why can

```
Llama 70B

↓

Run on one GPU?
```

Depends on

```
Quantization
GQA
KV Cache
```

Understanding LLM Internals explains these decisions.

---

# 2. The Evolution of NLP Models

Before Transformers,

Natural Language Processing evolved through several generations.

Very simplified timeline

```
Rule Based Systems

↓

Statistical NLP

↓

RNN

↓

LSTM / GRU

↓

Transformer

↓

Large Language Models
```

The chapter focuses on the transition from

```
RNN

↓

Transformer
```

because this is the biggest breakthrough in modern AI. :contentReference[oaicite:1]{index=1}

---

# Rule-Based Systems

Very early NLP systems used

```
If

Else

Rules
```

Example

```
IF

contains "hello"

↓

Greeting
```

These systems worked only

for situations

developers had already predicted.

Problems

- Impossible to scale
- Hard to maintain
- No generalization

---

# Statistical Models

Later,

Machine Learning replaced

hand-written rules.

Instead of writing

```
if word == hello
```

models learned

probabilities

from data.

This improved flexibility

but still struggled with

long sentences

and complex language.

---

# RNNs

Then came

```
Recurrent Neural Networks
```

For the first time,

the model processed

language

word by word.

---

# 3. Why RNNs Failed

Understanding RNNs explains

why Transformers became necessary.

---

## How RNN Works

Imagine the sentence

```
The cat sat on the mat.
```

RNN processes it like this

```
The

↓

cat

↓

sat

↓

on

↓

the

↓

mat
```

Notice

Every word waits

for the previous word.

This is called

```
Sequential Processing
```

---

# Why is this slow?

Imagine

1000 tokens.

The GPU cannot process

Token 900

until

Token 899 finishes.

Pipeline

```
Token 1

↓

Token 2

↓

Token 3

↓

...

↓

Token 1000
```

No parallelism.

Modern GPUs hate this.

---

# Backend Analogy

Imagine an Express.js server.

Sequential processing

```
Request 1

↓

Finish

↓

Request 2

↓

Finish

↓

Request 3
```

Terrible throughput.

Now imagine

Node.js

handling many requests

simultaneously.

Much faster.

Transformers achieve a similar leap by processing tokens in parallel during training.

---

# Long-Term Memory Problem

Consider

```
The animal

that chased the mouse

while running through the garden

was actually

the cat.
```

When the model reaches

```
cat
```

it must still remember

```
animal
```

from many words earlier.

RNNs struggle to preserve

this long-range information.

This is commonly known as the

```
Long-Term Dependency Problem.
```

---

# Vanishing Gradient

The chapter also mentions one of the biggest training problems of RNNs. :contentReference[oaicite:2]{index=2}

Imagine passing a message

through

100 people.

Every person changes it

slightly.

By the end,

the original message

is almost gone.

Training signals in deep recurrent networks can fade in a similar way.

This is called

```
Vanishing Gradient
```

Result

The model struggles

to learn relationships

between distant words.

---

# LSTM Improved Things

LSTM

(Long Short-Term Memory)

introduced

```
Memory Cells

+

Gates
```

which helped retain information

for longer.

But

LSTMs still processed

tokens

one after another.

So they improved memory,

not parallelism.

---

# 4. Why Transformers Changed Everything

The paper

```
Attention Is All You Need
```

introduced

the Transformer architecture.

Instead of reading

one word at a time,

Transformers let every token

look at every other token

during training.

Conceptually

```
The

↓

looks at

↓

every word
```

```
cat

↓

looks at

↓

every word
```

```
sat

↓

looks at

↓

every word
```

All at once.

This is the foundation of

Self-Attention,

which we'll study in the next part.

---

# Massive Parallelism

Instead of

```
Token 1

↓

Token 2

↓

Token 3
```

Transformers process

```
All Tokens

↓

Together
```

GPUs excel at

large matrix operations,

making Transformers dramatically faster to train than RNNs.

---

# Better Long-Range Understanding

Suppose the sentence is

```
The book on the table near the window belongs to Alice.
```

The word

```
belongs
```

can directly attend to

```
Alice
```

without passing through

every intermediate word.

This is a huge advantage.

---

# 5. High-Level Transformer Architecture

The repository introduces the major building blocks of a Transformer before diving into each component. :contentReference[oaicite:3]{index=3}

High-level view

```
Input Tokens

↓

Embeddings

↓

Position Information

↓

Attention

↓

Feed Forward Network

↓

Repeat Many Layers

↓

Output
```

Don't worry about each block yet.

We'll study

every one

in detail

over the next parts.

---

# Mental Model

Think of a Transformer layer as

```
Read

↓

Think

↓

Refine
```

Where

Read

↓

Attention

Think

↓

FFN

Refine

↓

Residual + Normalization

After many layers,

the representation becomes

increasingly sophisticated.

---

# 6. Encoder vs Decoder

The chapter distinguishes

three Transformer families. :contentReference[oaicite:4]{index=4}

---

## Encoder

Job

```
Understand
```

Example

```
Sentence

↓

Meaning
```

Typical models

- BERT
- Modern embedding models

Best for

- Classification
- Retrieval
- Embeddings

---

## Decoder

Job

```
Generate
```

Example

```
Prompt

↓

Next Token

↓

Next Token

↓

Next Token
```

Typical models

- GPT
- Llama
- Qwen
- DeepSeek

Best for

- Chat
- Coding
- Reasoning
- Text Generation

---

## Encoder-Decoder

Job

```
Read

↓

Generate
```

Example

```
Translate English

↓

French
```

Typical models

- T5
- BART

Useful for

- Translation
- Summarization
- Sequence-to-sequence tasks

---

# Backend Analogy

Think of APIs.

Encoder

```
GET

↓

Read Data
```

Decoder

```
POST

↓

Create Response
```

Encoder-Decoder

```
GET

↓

Process

↓

POST
```

Not technically identical,

but a useful mental model.

---

# 7. Why GPT Uses Decoder Only

This is one of the most common interview questions.

GPT's goal is

```
Predict

the next token.
```

Example

Prompt

```
The capital of France is
```

Model predicts

```
Paris
```

Then predicts

the next token,

and so on.

Because the task is

generation,

a Decoder-only architecture

is sufficient.

The repository introduces this distinction here and expands on it later. :contentReference[oaicite:5]{index=5}

---

# 8. Why This Matters for AI Engineers

Understanding these architectural choices helps explain

why different models exist.

Examples

Need

```
Embeddings?
```

Choose

Encoder.

Need

```
Chatbot?
```

Choose

Decoder.

Need

```
Translation?
```

Choose

Encoder-Decoder.

Architecture should match

the task.

---

# Interview Questions

## Q1

Why did Transformers replace RNNs?

Strong Answer

Transformers solve two major problems of RNNs. They process tokens in parallel, making training dramatically faster on GPUs, and they capture long-range relationships using self-attention instead of relying on sequential hidden states.

---

## Q2

Why is parallelism important?

Strong Answer

Modern GPUs are optimized for large matrix computations. Transformers process many tokens simultaneously, allowing much higher hardware utilization than sequential RNNs.

---

## Q3

Difference between Encoder and Decoder?

Strong Answer

Encoders learn contextual representations of the entire input and are ideal for understanding tasks such as embeddings and classification. Decoders generate text autoregressively by predicting one token at a time, making them suitable for chatbots and code generation.

---

# Things to Memorize

✅ RNNs process sequentially.

✅ Transformers process tokens in parallel during training.

✅ Self-Attention solved long-range dependency problems.

✅ Encoder = Understanding.

✅ Decoder = Generation.

✅ Encoder-Decoder = Read + Generate.

---

# Key Takeaways

✅ Understanding Transformer architecture is the foundation for every modern AI Engineer.

✅ RNNs introduced sequential language modeling but suffered from slow training and difficulty modeling long-range dependencies.

✅ Transformers replaced recurrence with self-attention, enabling parallel training and stronger contextual understanding.

✅ Choosing the right Transformer architecture depends on the task—understanding, generation, or both.

---

## Next Part

We'll dive into the **heart of every LLM**:

- What are Queries (Q), Keys (K), and Values (V)?
- Why do we create Q, K, and V?
- What is Self-Attention?
- Why is attention computed as QKᵀ?
- Why divide by √dk?
- Why apply Softmax?
- How does one attention head actually work?
- The complete attention calculation with examples and intuition.

This is the single most important concept in modern AI systems.

# LLM Internals — Part 2
> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, backend analogies, production notes, and interview insights.

---

# Table of Contents

1. Why Self-Attention?
2. Queries, Keys and Values
3. Building Q, K and V
4. Attention Score (QKᵀ)
5. Why Divide by √dk?
6. Softmax
7. Weighted Sum with Values
8. Complete Attention Pipeline
9. Production Implications
10. Interview Notes
11. Key Takeaways

---

# 1. Why Self-Attention?

The repository introduces **Self-Attention** as the core operation that replaced recurrence in Transformers. :contentReference[oaicite:0]{index=0}

The biggest question is:

> How does a word know which other words are important?

Consider the sentence:

```
The animal didn't cross the road because it was tired.
```

What does **"it"** refer to?

```
Animal?
Road?
```

Humans immediately know it refers to **animal**.

A Transformer must learn this relationship automatically.

That's exactly what Self-Attention does.

Instead of reading words one by one, every token decides:

> "Which other tokens should I pay attention to?"

---

# 2. Queries, Keys and Values

Every input token is converted into **three different vectors**:

```
Input Embedding
       │
 ┌─────┼─────┐
 │     │     │
 Q     K     V
```

These are not three different words.

They are **three different representations of the same word**.

Think of them as different roles.

| Vector | Purpose |
|---------|---------|
| Query (Q) | What information am I looking for? |
| Key (K) | What information do I contain? |
| Value (V) | What information should I contribute? |

---

## An Interview Analogy

Imagine you're interviewing candidates.

You ask:

```
Looking for Node.js developers.
```

That's your **Query**.

Every resume contains skills.

Those are the **Keys**.

The actual resume content is the **Value**.

The better a resume matches your query,

the more attention you give to that candidate.

Exactly the same thing happens inside a Transformer.

---

# 3. How are Q, K and V Created?

The repository explains that Q, K and V are produced by applying **three different learned weight matrices** to the input embeddings. :contentReference[oaicite:1]{index=1}

Mathematically:

```
Q = XWQ

K = XWK

V = XWV
```

Where:

- **X** = Input embeddings
- **WQ** = Learned Query weights
- **WK** = Learned Key weights
- **WV** = Learned Value weights

Important point:

The input embedding is the same.

Only the learned projection matrices differ.

---

## Why Three Different Matrices?

Suppose the word is:

```
Apple
```

Depending on context,

it may represent:

- A fruit
- A company

When acting as a **Query**, the model asks:

> "What information do I need?"

When acting as a **Key**, it advertises:

> "What information do I have?"

When acting as a **Value**, it carries the information that will be passed to the next layer.

Keeping these roles separate makes attention much more expressive.

---

# 4. Computing Attention Scores

Now every Query compares itself with every Key.

This is done using:

```
QKᵀ
```

Conceptually:

```
Query
   │
Compare
   │
Every Key
```

The result is an **attention score**.

Example:

| Token | Score |
|--------|------:|
| The | 0.10 |
| Animal | 0.82 |
| Road | 0.14 |
| Tired | 0.91 |

Higher score means:

> "This token is more relevant."

Notice that every token compares itself with **every other token**.

This is why it's called **Self-Attention**.

---

## Why Matrix Multiplication?

Instead of comparing tokens one by one,

Transformers compare **all tokens simultaneously** using matrix multiplication.

```
Q Matrix

×

Kᵀ Matrix

↓

Attention Scores
```

This is one of the reasons GPUs are so effective for Transformer models.

---

# 5. Why Divide by √dk?

The repository includes the attention equation:

```
Softmax(QKᵀ / √dk)
```

Many people memorize this.

Interviewers want you to understand **why**.

---

## The Problem

As vector dimensions become larger,

the dot product also becomes larger.

Example:

```
64 dimensions

↓

Small scores

1024 dimensions

↓

Very large scores
```

Large scores cause Softmax to become extremely sharp.

Example:

```
Before

8
9
10

↓

After Softmax

0.00
0.01
0.99
```

The model becomes overconfident.

Learning becomes unstable.

---

## The Solution

Scale the scores by

```
√dk
```

This keeps values within a reasonable range before applying Softmax.

Think of it as **normalization** before converting scores into probabilities.

---

# 6. Softmax

After scaling,

the model applies Softmax.

Purpose:

Convert arbitrary scores into probabilities.

Example:

Raw Scores:

```
2.3
1.8
0.5
```

After Softmax:

```
0.55
0.33
0.12
```

Properties:

- Every value is between 0 and 1.
- All values sum to 1.

Now the model knows **how much attention** to give each token.

---

# 7. Weighted Sum of Values

Until now,

we have only calculated **attention weights**.

We haven't actually transferred information.

The final step is:

```
Attention Weights

×

Values

↓

Output
```

Suppose the attention weights are:

| Token | Weight |
|--------|--------|
| Animal | 0.70 |
| Road | 0.10 |
| Tired | 0.20 |

The output representation becomes mostly influenced by **Animal**, because it received the highest weight.

This is why the **Value** vectors matter.

Keys decide **who is important**.

Values decide **what information gets passed forward**.

---

# 8. Complete Attention Pipeline

Putting everything together:

```
Input Tokens
      ↓
Embeddings
      ↓
Generate Q, K, V
      ↓
QKᵀ
      ↓
Scale by √dk
      ↓
Softmax
      ↓
Attention Weights
      ↓
Multiply by V
      ↓
Output
```

This entire process happens inside every Transformer layer.

---

# Backend Analogy

Imagine a search engine.

```
User Search

↓

Query

↓

Search Index

↓

Keys

↓

Documents

↓

Values
```

The search engine first determines **which documents are relevant**.

Then it returns the **document contents**.

Attention works in a very similar way.

---

# Production Implications

Self-Attention provides several advantages:

- Every token can directly access information from every other token.
- Long-range dependencies become much easier to learn.
- Matrix operations allow efficient GPU execution.
- The same mechanism scales across many Transformer layers.

However, there is one important limitation.

Every token compares itself with every other token.

If there are **N** tokens,

the number of comparisons grows roughly as:

```
N × N
```

This is called **O(n²) complexity**.

It is one of the biggest reasons long-context models consume so much memory and compute.

The repository introduces this limitation before later discussing optimizations such as GQA and KV Cache. :contentReference[oaicite:2]{index=2}

---

# 🎯 Interview Notes

### What is Self-Attention?

Self-Attention allows each token to dynamically determine which other tokens are most relevant when building its contextual representation.

---

### What do Q, K and V represent?

- Query: What information am I looking for?
- Key: What information do I contain?
- Value: What information should I contribute?

---

### Why divide by √dk?

To prevent dot-product values from becoming too large, which would make Softmax overly confident and destabilize training.

---

### Why use Values instead of Keys?

Keys are only used to compute similarity scores. The actual information passed to the next layer comes from the Value vectors.

---

# 📝 Key Takeaways

- Self-Attention is the core computation in every Transformer.
- Every token produces a Query, Key and Value representation.
- Attention scores are computed using **QKᵀ**.
- Scaling by **√dk** stabilizes Softmax.
- Softmax converts scores into attention probabilities.
- Values carry the information forwarded to the next layer.
- Self-Attention enables rich contextual understanding but has **O(n²)** computational complexity.

---

## Next Part

We'll cover:

- Multi-Head Attention
- Why multiple heads are needed
- Head specialization
- Multi-Query Attention (MQA)
- Grouped Query Attention (GQA)
- Why Llama has **64 Query Heads but only 8 KV Heads**
- The memory savings and production implications

This section directly connects to KV Cache and modern LLM inference optimization.


# LLM Internals — Part 3
> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, production notes and interview insights.

---

# Table of Contents

1. Why One Attention Head Isn't Enough
2. Multi-Head Attention
3. Why Different Heads Learn Different Things
4. Output Projection (WO)
5. Grouped Query Attention (GQA)
6. MHA vs GQA vs MQA
7. Production Impact
8. Interview Notes
9. Key Takeaways

---

# 1. Why One Attention Head Isn't Enough?

A single attention head can only learn **one type of relationship** at a time.

Consider the sentence:

> "The animal didn't cross the street because it was tired."

Different relationships exist simultaneously:

- "it" → animal (coreference)
- "cross" → street (action)
- "animal" → tired (reason)

If one attention head tries to learn all of these patterns, its capacity becomes limited.

The repository introduces **Multi-Head Attention** to solve this problem. :contentReference[oaicite:2]{index=2}

---

# 2. Multi-Head Attention

Instead of one attention mechanism, Transformers run several attention mechanisms in parallel.

```
Input
   │
 ┌─┴──────────────────────┐
 │   │   │   │   │
H1  H2  H3  H4 ... Hn
 │   │   │   │
 └───┴───┴───┘
      │
Concatenate
      │
     WO
      │
   Output
```

Each head has its **own**:

- WQ
- WK
- WV

So every head learns different attention patterns. :contentReference[oaicite:3]{index=3}

---

# Example

Suppose a model has

```
d_model = 4096
Heads = 32
```

Then each head works on

```
4096 / 32 = 128 dimensions
```

instead of all 4096.

Every head processes a different "view" of the sentence.

---

# 3. Why Different Heads Learn Different Things?

During training, each head develops its own specialization.

Typical patterns include:

| Head | Learns |
|-------|---------|
| 1 | Grammar |
| 2 | Subject–Verb |
| 3 | Coreference ("it", "he") |
| 4 | Long-range dependencies |
| 5 | Punctuation |
| 6 | Semantic similarity |

The repository summarizes this idea as:

- Syntax
- Semantics
- Coreference
- Multiple perspectives improve robustness :contentReference[oaicite:4]{index=4}

Important:

Nobody explicitly tells Head 1 to learn grammar.

These patterns emerge automatically during training.

---

# Backend Analogy

Imagine reviewing a pull request.

Instead of asking one engineer,

you ask

- Backend Engineer
- Database Engineer
- Security Engineer
- DevOps Engineer

Each notices different issues.

You combine all feedback before merging.

Multi-Head Attention works similarly.

---

# 4. Output Projection (WO)

After every head produces an output,

the outputs are concatenated.

```
Head1 Output

+

Head2 Output

+

...

↓

Concatenate

↓

WO

↓

Final Output
```

WO (Output Projection Matrix)

mixes information from all heads into a single representation.

Without WO,

heads would remain independent.

WO allows the model to combine everything it has learned.

---

# 5. Grouped Query Attention (GQA)

The repository identifies GQA as a **critical production optimization**. :contentReference[oaicite:5]{index=5}

In standard Multi-Head Attention:

Every Query Head has its own

- Key
- Value

Example:

```
64 Query Heads

↓

64 Key Heads

↓

64 Value Heads
```

This creates a large KV Cache.

---

## GQA Idea

Instead of creating separate K and V for every Query Head,

multiple Query Heads share the same Key and Value heads.

Example:

```
64 Query Heads

↓

8 KV Heads
```

Each KV head serves multiple Query heads.

This dramatically reduces memory usage.

---

# Visual Example

### Standard Multi-Head Attention

```
Q1 → K1 V1

Q2 → K2 V2

Q3 → K3 V3

Q4 → K4 V4
```

Every Query has its own KV.

---

### GQA

```
Q1
Q2
Q3
Q4

↓

Shared K1 V1
```

Multiple Query heads reuse the same KV pair.

---

# Why Doesn't This Hurt Quality Much?

Queries ask

> "Who should I attend to?"

Keys and Values contain

the information.

Different Query heads can often use

the same Key and Value representations.

So we reduce memory

without losing much model quality.

This is why GQA has become common in modern LLMs. :contentReference[oaicite:6]{index=6}

---

# 6. MHA vs GQA vs MQA

The repository compares three variants. :contentReference[oaicite:7]{index=7}

| Method | KV Sharing | Memory | Quality |
|---------|------------|---------|----------|
| MHA | None | Highest | Best |
| GQA | Shared by groups | Low | Nearly identical |
| MQA | One KV for all heads | Lowest | Slight quality drop |

Think of it this way:

**MHA**

Everyone has their own notebook.

**GQA**

Small teams share notebooks.

**MQA**

The whole class shares one notebook.

Memory decreases,

but excessive sharing can reduce quality.

---

# Practical Example

The repository gives an example for **Llama 2 70B**:

```
64 Query Heads

↓

8 KV Heads
```

Result:

- Approximately **8× smaller KV Cache**
- Much higher serving throughput
- Minimal impact on model quality :contentReference[oaicite:8]{index=8}

This is one of the reasons Llama can serve long-context requests more efficiently.

---

# Production Impact

Why do companies care so much about GQA?

Because serving cost matters more than training cost.

Without GQA:

```
More KV Cache

↓

More GPU Memory

↓

Smaller Batch Size

↓

Lower Throughput
```

With GQA:

```
Smaller KV Cache

↓

More Requests Per GPU

↓

Lower Cost

↓

Higher Throughput
```

This is why the repository calls GQA a production-critical optimization.

---

# Backend Analogy

Imagine Redis.

Without sharing:

Every request creates its own cache.

```
100 Requests

↓

100 Cache Copies
```

Huge memory usage.

With sharing:

```
100 Requests

↓

Shared Cache
```

Less memory,

same information.

GQA follows a similar principle.

---

# Interview Notes

### Why use Multi-Head Attention?

Multiple heads learn different linguistic and semantic relationships simultaneously, making the model more expressive than a single attention head.

---

### Why is GQA important?

GQA reduces KV Cache memory by allowing multiple Query heads to share Key and Value heads. This significantly improves inference efficiency with very little quality loss.

---

### Difference between MHA, GQA and MQA?

- **MHA:** Every Query has its own Key and Value.
- **GQA:** Groups of Query heads share Key and Value.
- **MQA:** All Query heads share a single Key and Value.

---

# Key Takeaways

- Multi-Head Attention allows different heads to learn different patterns.
- Each head has its own Q, K and V projection matrices.
- Outputs from all heads are concatenated and projected using WO.
- GQA shares Key and Value heads across groups of Query heads.
- Modern models such as Llama and Mistral use GQA because it provides an excellent balance between memory usage and model quality.
- GQA is one of the most important optimizations for production LLM serving.

---

## Next Part

We'll cover **Position Encodings**, including:

- Why Transformers don't understand word order
- Sinusoidal Position Encoding
- Learned Position Embeddings
- Rotary Position Embeddings (RoPE)
- ALiBi
- Why RoPE is used in Llama
- How RoPE enables long-context models



# LLM Internals — Part 4
> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, production notes, and interview insights.

---

# Table of Contents

1. Why Positional Encoding?
2. Why Transformers Don't Understand Order
3. Absolute Position Embeddings
4. Sinusoidal Position Encoding
5. Learned Position Embeddings
6. Rotary Position Embeddings (RoPE)
7. Why Modern LLMs Prefer RoPE
8. Production Implications
9. Interview Notes
10. Key Takeaways

---

# 1. Why Positional Encoding?

Self-Attention treats input as a **set of tokens**, not a sequence.

Without positional information, these two sentences look identical:

```
Dog bites man
```

```
Man bites dog
```

Both contain the same words.

The only difference is **order**.

Humans understand this immediately.

A Transformer does not.

Therefore, we need to explicitly tell the model where each token appears.

This is the purpose of **Positional Encoding**. :contentReference[oaicite:0]{index=0}

---

# 2. Why Transformers Don't Understand Order

Consider these tokens:

```
["I", "love", "AI"]
```

After tokenization and embedding:

```
I      → Vector
love   → Vector
AI     → Vector
```

The model knows **what** each token is.

It does **not** know:

- Which token came first
- Which token came second
- Which token came last

Without positional information,

```
I love AI
```

and

```
AI love I
```

would appear almost identical.

This is why every Transformer adds positional information before attention.

---

# 3. Absolute Position Embeddings

The simplest solution is to assign every position its own embedding.

Example:

| Position | Position Embedding |
|----------|--------------------|
| 0 | P0 |
| 1 | P1 |
| 2 | P2 |

The final input becomes:

```
Token Embedding

+

Position Embedding

↓

Input to Transformer
```

Example

```
"I"

↓

Embedding(I)

+

P0
```

```
"love"

↓

Embedding(love)

+

P1
```

Simple and effective.

---

## Limitation

Absolute embeddings only learn positions seen during training.

Suppose training used

```
2048 Tokens
```

What happens if we ask the model to process

```
4096 Tokens?
```

There are no learned embeddings for positions beyond 2048.

Generalization becomes difficult.

---

# 4. Sinusoidal Position Encoding

The original Transformer paper introduced **Sinusoidal Position Encoding**.

Instead of learning one embedding per position,

positions are generated using mathematical functions:

```
sin()

cos()
```

Each position receives a unique pattern.

Conceptually:

```
Position 0

↓

[0.0, 1.0, ...]

Position 1

↓

[0.84, 0.54, ...]

Position 2

↓

...
```

Every position gets a deterministic encoding.

---

## Why Use Sine and Cosine?

Nearby positions produce similar patterns.

Far-apart positions produce different patterns.

This allows the model to reason about **relative distance** between tokens.

Advantages:

- No extra parameters
- Can generalize to longer sequences
- Works for unseen positions

---

# 5. Learned Position Embeddings

Many models later switched to **Learned Position Embeddings**.

Instead of mathematical functions,

the model learns position vectors during training.

Pipeline:

```
Position

↓

Embedding Table

↓

Learned Position Vector
```

Advantages:

- More flexible
- Often better accuracy on trained context lengths

Disadvantages:

- Doesn't extrapolate well beyond training length
- Requires additional parameters

---

# 6. Rotary Position Embeddings (RoPE)

The repository highlights **RoPE** because it is used by modern LLMs such as Llama. :contentReference[oaicite:1]{index=1}

Instead of adding positional information,

RoPE **rotates** the Query and Key vectors based on their position.

Conceptually:

```
Embedding

↓

Rotate(Q)

Rotate(K)

↓

Attention
```

The embeddings themselves stay the same.

Only Q and K are rotated before attention is computed.

---

## Why Rotate Instead of Add?

Attention compares

```
Query

↓

Key
```

If positional information is built directly into Q and K,

the attention mechanism naturally becomes position-aware.

This makes relative positions easier to learn.

---

# Example

Sentence:

```
The cat sat on the mat.
```

With RoPE,

the Query and Key vectors for each token are rotated according to their position.

When "sat" computes attention,

it automatically knows whether another token is nearby or far away.

This helps the model capture word order without relying on absolute position IDs.

---

# 7. Why Modern LLMs Prefer RoPE

RoPE has become the standard choice because it provides several advantages.

### Better Relative Position Understanding

The model naturally learns relationships like:

```
Previous Token

Next Token

Nearby Token

Far-away Token
```

instead of memorizing absolute positions.

---

### Better Long-Context Performance

RoPE generalizes better than learned position embeddings when context windows increase.

This is one reason modern models can support

```
32K

64K

128K
```

contexts.

Additional techniques are still required for extremely long contexts, but RoPE is a major building block.

---

### No Large Position Embedding Table

Since positions are encoded mathematically,

there is no need to store thousands of learned position vectors.

This keeps the architecture elegant.

---

# Backend Analogy

Think of GPS coordinates.

Instead of saying

```
Building Number 42
```

you say

```
200 meters north

50 meters east
```

The second description captures **relative position**.

RoPE works in a similar way.

It emphasizes relationships between positions instead of memorizing fixed IDs.

---

# Production Implications

Modern open-source LLMs commonly use RoPE because it offers:

- Better long-context behavior
- Relative position awareness
- Efficient integration into attention

RoPE is now a standard component in architectures such as Llama, Qwen, and DeepSeek.

---

# Interview Notes

### Why do Transformers need positional encoding?

Self-Attention has no inherent understanding of token order. Positional information tells the model where each token appears in the sequence.

---

### Difference between Absolute Position Embeddings and RoPE?

**Absolute Position Embeddings**

- Add learned vectors to token embeddings.
- Limited generalization beyond training length.

**RoPE**

- Rotates Query and Key vectors.
- Naturally captures relative positions.
- Better suited for long-context models.

---

### Why is RoPE popular?

Because it provides strong relative position modeling while scaling better to longer context windows than traditional learned position embeddings.

---

# Key Takeaways

- Self-Attention alone cannot understand word order.
- Positional information is required for every Transformer.
- Absolute Position Embeddings are simple but have context-length limitations.
- Sinusoidal Encoding generates deterministic position vectors using sine and cosine functions.
- RoPE rotates Query and Key vectors instead of adding position vectors.
- RoPE has become the standard positional encoding method for many modern LLMs because of its strong long-context performance.

---

## Next Part

We'll cover one of the most misunderstood components of a Transformer:

- Feed Forward Networks (FFN)
- Why FFNs contain most of the model's parameters
- GELU vs SwiGLU
- Why MoE replaces FFNs
- LayerNorm vs RMSNorm
- Residual Connections
- Building a complete Transformer block


# LLM Internals — Part 5
> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, production notes, and interview insights.

---

# Table of Contents

1. Feed Forward Networks (FFN)
2. Why FFNs Exist
3. FFN Architecture
4. Activation Functions (ReLU, GELU, SwiGLU)
5. Why FFNs Contain Most Parameters
6. Residual Connections
7. LayerNorm vs RMSNorm
8. Complete Transformer Block
9. Production Implications
10. Interview Notes
11. Key Takeaways

---

# 1. Feed Forward Networks (FFN)

Most beginners think **Attention** does all the work inside a Transformer.

It doesn't.

Attention helps a token gather information from other tokens.

The **Feed Forward Network (FFN)** is where the model performs most of its computation and stores a large portion of its knowledge.

A Transformer layer looks like this:

```
Input
  │
Attention
  │
Residual + Norm
  │
Feed Forward Network
  │
Residual + Norm
  │
Output
```

Every Transformer layer contains **both Attention and FFN**.

---

# 2. Why FFNs Exist

Attention answers one question:

> Which tokens are important?

It does **not** deeply transform the information.

Example:

```
"The capital of France is Paris."
```

Attention may determine that **France** should strongly attend to **Paris**.

But recognizing:

- Paris is a city
- Paris is a capital
- Capitals belong to countries

requires additional computation.

That computation happens inside the FFN.

Think of it as:

```
Attention

↓

Collect Information

↓

FFN

↓

Process Information
```

---

# 3. FFN Architecture

A standard FFN consists of two linear layers with a non-linear activation in between.

```
Input

↓

Linear Layer

↓

Activation

↓

Linear Layer

↓

Output
```

Mathematically:

```
FFN(x)

=

W₂(Activation(W₁x))
```

Where:

- W₁ expands the representation.
- Activation introduces non-linearity.
- W₂ projects it back to the original size.

---

## Why Expand First?

Suppose

```
Embedding Size

4096
```

The FFN first expands it to something like

```
11008
```

Then compresses it back.

```
4096

↓

11008

↓

4096
```

This larger hidden space allows the model to learn much richer patterns.

---

# 4. Activation Functions

Without an activation function,

the FFN would simply behave like another linear transformation.

Linear + Linear is still linear.

That would greatly reduce the model's expressive power.

---

## ReLU

```
f(x)=max(0,x)
```

Advantages

- Simple
- Fast

Problems

- Neurons can permanently become inactive ("dead neurons").

---

## GELU

Modern Transformers often replaced ReLU with **GELU (Gaussian Error Linear Unit)**.

Instead of abruptly removing negative values,

GELU smoothly weights them.

Benefits:

- Better gradient flow
- More stable training
- Better language modeling performance

Models such as BERT and GPT-2 use GELU.

---

## SwiGLU

The repository also introduces **SwiGLU**, which has become common in modern LLMs. :contentReference[oaicite:0]{index=0}

Instead of a simple activation,

SwiGLU adds a gating mechanism.

Conceptually:

```
Information

×

Gate

↓

Output
```

The gate learns

how much information should pass through.

Think of it like a valve controlling water flow.

---

## Why SwiGLU?

Advantages:

- Better parameter efficiency
- Better reasoning performance
- Stronger gradient flow
- Widely adopted in Llama, Qwen, DeepSeek and other recent models

---

# 5. Why FFNs Contain Most Parameters

This surprises many engineers.

Attention receives the most attention (pun intended),

but FFNs usually contain **more parameters**.

Example:

Suppose

```
Embedding Dimension = 4096
```

FFN expands to

```
11008
```

The weight matrices become very large:

```
4096 × 11008

+

11008 × 4096
```

These matrices dominate the parameter count.

In many modern Transformers,

roughly **60–70%** of all parameters belong to the FFNs.

This is also why **Mixture of Experts (MoE)** replaces FFNs rather than the Attention mechanism.

---

# 6. Residual Connections

Deep neural networks become difficult to train.

Information can degrade as it passes through many layers.

Residual connections solve this.

Instead of keeping only the new output,

the model adds the original input back.

```
Input
 │
Attention
 │
 +───────────┐
 │           │
 └────Add────┘
      │
Output
```

Mathematically:

```
Output = Input + Attention(Input)
```

The same idea is used after the FFN.

Benefits:

- Better gradient flow
- Easier optimization
- Enables extremely deep Transformers

Without residual connections,

training 80-layer Transformers would be extremely difficult.

---

# 7. LayerNorm vs RMSNorm

After residual addition,

the output is normalized.

Normalization stabilizes training by keeping activations within a healthy range.

---

## LayerNorm

LayerNorm normalizes both:

- Mean
- Variance

Pipeline:

```
Input

↓

Normalize

↓

Output
```

LayerNorm was used in the original Transformer architecture.

---

## RMSNorm

Modern LLMs often use **RMSNorm** instead.

The repository notes this transition because RMSNorm is computationally simpler and faster. :contentReference[oaicite:1]{index=1}

Instead of computing both mean and variance,

RMSNorm only uses the root mean square.

Advantages:

- Faster computation
- Lower inference cost
- Similar model quality
- Better hardware efficiency

This is why models such as Llama use RMSNorm.

---

# 8. Complete Transformer Block

Putting everything together,

a Transformer block looks like:

```
Input
  │
Multi-Head Attention
  │
Add (Residual)
  │
RMSNorm
  │
Feed Forward Network
  │
Add (Residual)
  │
RMSNorm
  │
Output
```

This block is repeated many times.

Examples:

| Model | Layers |
|--------|--------|
| GPT-2 | 12–48 |
| Llama 3 | 32–80 |
| Large MoE Models | 100+ |

Every layer gradually builds richer contextual representations.

---

# Backend Analogy

Imagine processing an API request.

```
Receive Request

↓

Read Database

↓

Business Logic

↓

Return Response
```

Transformer equivalent:

```
Attention

↓

Gather Information

↓

FFN

↓

Process Information
```

Attention collects.

FFN thinks.

---

# Production Implications

Understanding FFNs explains several production design choices.

### Why MoE Targets FFNs

Since FFNs contain most of the parameters,

replacing them with sparse experts dramatically reduces inference cost.

Attention remains shared,

while FFNs become expert-specific.

---

### Why RMSNorm Matters

Inference runs billions of times in production.

Even tiny computational savings per layer become significant at scale.

Replacing LayerNorm with RMSNorm reduces latency while maintaining quality.

---

# 🎯 Interview Notes

### What does the FFN do?

Attention gathers information from other tokens.

The FFN transforms that information into richer representations using non-linear computation.

---

### Why are FFNs so large?

Because the hidden dimension expands significantly before being projected back, resulting in very large weight matrices.

---

### Why use Residual Connections?

Residual connections improve gradient flow and allow Transformers to scale to dozens or even hundreds of layers.

---

### LayerNorm vs RMSNorm?

- **LayerNorm:** Normalizes mean and variance.
- **RMSNorm:** Uses only the root mean square, making it computationally cheaper while achieving similar performance.

---

# 📝 Key Takeaways

- Attention collects information; FFNs process it.
- FFNs usually contain the majority of Transformer parameters.
- Modern LLMs commonly use **SwiGLU** instead of simple activations.
- Residual connections enable very deep Transformer networks.
- RMSNorm has largely replaced LayerNorm in recent LLM architectures because it is simpler and more efficient.
- Understanding FFNs is essential before studying **Mixture of Experts (MoE)**, since MoE primarily replaces FFN layers rather than Attention.

---

## Next Part

We'll cover one of the most important topics in modern LLMs:

- Mixture of Experts (MoE)
- Why dense models don't scale efficiently
- Experts and Routers
- Top-K Routing
- Load Balancing Loss
- Active vs Total Parameters
- Why DeepSeek, Mixtral and GPT-5 use MoE
- Production trade-offs and interview questions

This is one of the hottest AI Engineer interview topics in 2026.


# LLM Internals — Part 6
> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, production notes, and interview insights.

---

# Table of Contents

1. Why Dense Models Don't Scale
2. What is Mixture of Experts (MoE)?
3. Dense Model vs MoE
4. Router (The Expert Selector)
5. Top-K Routing
6. Active vs Total Parameters
7. Load Balancing
8. Advantages and Trade-offs
9. Real-World Models
10. Interview Notes
11. Key Takeaways

---

# 1. Why Dense Models Don't Scale

Traditional Transformers are called **Dense Models**.

In a dense model:

> Every parameter participates in every forward pass.

Example:

```
70B Model

↓

Every token

↓

Uses all 70B parameters
```

Even if only a small portion of the model is actually needed to process a token, the entire network is activated.

This becomes expensive because:

- More computation
- More GPU time
- Higher latency
- Higher inference cost

As models grew from billions to trillions of parameters, this approach became impractical.

---

# 2. What is Mixture of Experts (MoE)?

Mixture of Experts solves this problem using **conditional computation**.

Instead of activating the entire model,

only a few specialized sub-networks (called **Experts**) are activated for each token.

```
Input Token

↓

Router

↓

Expert 7
Expert 12

↓

Combine Outputs

↓

Next Layer
```

The key idea is simple:

> **Large model capacity without activating the entire model.**

---

# 3. Dense Model vs MoE

### Dense Transformer

```
Input

↓

Entire FFN

↓

Output
```

Every token passes through the same FFN.

---

### MoE Transformer

```
Input

↓

Router

↓

Expert Selection

↓

Selected Experts Only

↓

Output
```

Different tokens may activate different experts.

---

## Analogy

Imagine a hospital.

### Dense Model

Every patient is examined by:

- Cardiologist
- Neurologist
- Orthopedic
- Dermatologist

Even if they only have a broken arm.

Very expensive.

---

### MoE

Patient arrives.

↓

Reception decides the correct department.

↓

Only relevant doctors examine the patient.

Same idea.

The **Router** acts like reception.

The **Experts** are specialists.

---

# 4. What is an Expert?

An Expert is usually **a Feed Forward Network (FFN)**.

This is important.

MoE does **not** replace Attention.

It replaces the FFN.

Instead of:

```
Attention

↓

One FFN
```

We get:

```
Attention

↓

Router

↓

Expert FFNs
```

This is why we studied FFNs before MoE.

---

# Example

Without MoE:

```
Layer

↓

One FFN
```

With MoE:

```
Layer

↓

Expert 1

Expert 2

Expert 3

...

Expert 64
```

The Router decides which experts should execute.

---

# 5. Router (The Brain of MoE)

The Router is a small neural network.

Its job is:

> Decide which experts should process each token.

Pipeline:

```
Token

↓

Router

↓

Scores

↓

Top Experts

↓

Inference
```

Example:

| Expert | Score |
|---------|------:|
| 1 | 0.03 |
| 2 | 0.82 |
| 3 | 0.11 |
| 4 | 0.91 |

The router selects:

```
Expert 4

Expert 2
```

because they received the highest scores.

---

# 6. Top-K Routing

Modern MoE models rarely activate only one expert.

Instead they activate the **Top-K experts**.

Example:

```
64 Experts

↓

Top-2
```

Only

```
Expert 8

Expert 29
```

run for this token.

Everything else remains inactive.

---

## Why Top-2 Instead of Top-1?

Top-1 is cheaper,

but risky.

If the router makes one bad decision,

the token loses important information.

Top-2 provides:

- Better robustness
- Better accuracy
- Smoother training

Many production MoE models use **Top-2 Routing**.

---

# Example

Sentence:

```
Python is a programming language.
```

Router Output:

| Expert | Specialty |
|---------|-----------|
| 4 | Programming |
| 17 | General Language |
| 23 | Biology |
| 30 | Finance |

Router chooses:

```
Programming

+

General Language
```

Biology and Finance experts are skipped.

---

# 7. Active vs Total Parameters

One of the most common interview topics.

Suppose a model advertises:

```
670B Parameters
```

This **does not** mean all 670B are used every token.

Instead:

```
Total Parameters

670B

↓

Active Parameters

37B
```

The model contains 670B parameters,

but only about 37B participate during one forward pass.

This is why MoE models can be enormous while remaining practical to serve.

---

# Why Companies Love This

Instead of paying inference cost for

```
670B
```

they pay roughly for

```
37B
```

while still benefiting from the larger overall capacity.

Think of it as:

Large brain.

Only relevant knowledge wakes up.

---

# 8. Load Balancing

One problem appears naturally.

Suppose one expert becomes extremely popular.

```
Expert 7

↓

90%

of tokens
```

Other experts receive almost no work.

```
Expert 2

↓

0.3%
```

This creates two problems:

- Some experts become overloaded.
- Other experts never learn.

---

## Load Balancing Loss

To solve this,

the router is trained using an additional objective.

Goal:

Distribute tokens more evenly across experts.

Ideal distribution:

```
Expert1

15%

Expert2

14%

Expert3

16%

...
```

instead of

```
Expert1

95%

Everyone Else

5%
```

This encourages all experts to learn useful specializations.

---

# 9. Advantages of MoE

## Huge Capacity

Large parameter counts become feasible.

---

## Lower Inference Cost

Only a few experts execute.

---

## Better Specialization

Different experts naturally learn different domains.

Examples:

- Code
- Mathematics
- Reasoning
- Multiple languages
- Scientific writing

Nobody manually assigns these tasks.

The specialization emerges during training.

---

## Better Scaling

Instead of making one FFN enormous,

we add more experts.

This often scales more efficiently.

---

# Trade-offs

MoE is not free.

Challenges include:

### Router Complexity

The router must make accurate expert selections.

---

### Load Balancing

Experts should receive similar workloads.

---

### Distributed Serving

Experts may live on different GPUs.

Routing tokens between GPUs introduces communication overhead.

Large-scale serving systems must optimize this carefully.

---

# 10. Real-World Models

The repository highlights MoE as the direction of modern frontier models.

Examples include:

- Mixtral
- DeepSeek
- Qwen MoE

The guide also notes that many recent large models have adopted MoE architectures because they provide much better inference efficiency than equally large dense models.

*(The repository intentionally focuses on the architectural trend rather than exhaustively listing every commercial model.)*

---

# Backend Analogy

Imagine a company with 100 engineers.

Dense Model:

Every engineer joins every meeting.

```
100 Engineers

↓

One Meeting
```

Very expensive.

MoE:

```
Bug in Payments

↓

Payments Team

+

Database Team
```

Only the relevant engineers participate.

The company still has 100 engineers,

but only a few work on each task.

Exactly how MoE operates.

---

# Production Implications

MoE has become one of the biggest architectural improvements in modern LLMs because it offers:

- Larger effective model capacity
- Lower inference cost
- Better GPU utilization
- Easier scaling toward trillion-parameter models

However, it also requires:

- Efficient routing
- Load balancing
- High-speed GPU communication

Building production MoE systems is significantly more complex than serving dense models.

---

# 🎯 Interview Notes

### What problem does MoE solve?

Dense models activate every parameter for every token, making inference expensive. MoE activates only a few experts, reducing computation while preserving model capacity.

---

### What is an Expert?

An Expert is usually a specialized Feed Forward Network (FFN). Attention remains shared across all tokens.

---

### What is the Router?

The Router scores all experts and selects the highest-ranked experts (Top-K) for each token.

---

### Active vs Total Parameters?

**Total Parameters** describe the model's full size.

**Active Parameters** are the subset actually used during one forward pass.

This is why a 600B+ MoE model may have inference costs similar to a much smaller dense model.

---

### Why is Load Balancing Necessary?

Without it, the router may repeatedly select the same experts, causing overload while other experts receive too little training.

---

# 📝 Key Takeaways

- Dense models activate every parameter; MoE activates only selected experts.
- MoE replaces FFNs, not the Attention mechanism.
- The Router decides which experts process each token.
- Top-K routing improves robustness over selecting a single expert.
- Active parameters are much smaller than total parameters.
- Load balancing ensures all experts learn and prevents routing bottlenecks.
- MoE is one of the most important architectural trends in modern LLMs.

---

## Next Part

We'll cover the final section of **LLM Internals**:

- Scaling Laws
- Chinchilla Scaling
- Training-Optimal vs Inference-Optimal Models
- Native Multimodality
- Why modern 8B models outperform older 70B models
- How all the components fit together into a complete Transformer

This will complete the **LLM Internals** chapter before moving to **Inference Optimization**.


# LLM Internals — Part 7 (Final)
> Based on **01 - LLM Internals** from the AI System Design Guide with additional intuition, production notes, and interview insights.

---

# Table of Contents

1. Scaling Laws
2. Chinchilla Scaling
3. Training-Optimal vs Inference-Optimal Models
4. Native Multimodality
5. Putting Everything Together
6. Complete Transformer Pipeline
7. Interview Notes
8. Key Takeaways
9. Final Revision Summary

---

# 1. Scaling Laws

One of the biggest discoveries in modern AI is that model performance follows predictable **Scaling Laws**.

As we increase:

- Model Parameters
- Training Data
- Compute

the model generally becomes better.

```
More Parameters
        +
More Data
        +
More Compute
        ↓
Better Performance
```

However,

performance doesn't improve forever.

Eventually,

returns begin to diminish.

This raises an important engineering question:

> **How should we spend our compute budget?**

Should we build:

- A bigger model?
- Train longer?
- Collect more data?

Scaling laws attempt to answer this.

---

# 2. Chinchilla Scaling Law

Earlier generations of LLMs assumed:

> Bigger models are always better.

DeepMind's **Chinchilla** research challenged this assumption.

The key finding was:

> Many large models were **undertrained**.

Instead of only increasing parameters,

we should also increase

- Training Tokens
- Training Duration

A well-trained smaller model can outperform a much larger undertrained model.

---

## Example

Imagine two students.

### Student A

- IQ: 180
- Studies: 2 hours

### Student B

- IQ: 140
- Studies: 500 hours

Student B may perform much better.

Exactly the same happens with language models.

---

## Practical Impact

This explains why

```
Modern 8B Models
```

can outperform

```
Older 70B Models
```

The newer model may simply have:

- Better architecture
- Better training data
- Better optimization
- More training tokens

Model size alone no longer determines quality.

---

# 3. Training-Optimal vs Inference-Optimal Models

The repository distinguishes between two different optimization goals.

---

## Training-Optimal

Goal:

Achieve the highest possible model quality during training.

Focus areas:

- Better accuracy
- Better convergence
- Better scaling

Training happens once,

so higher computational cost is acceptable.

---

## Inference-Optimal

Inference happens millions of times every day.

Now priorities change.

Companies care about:

- Lower latency
- Lower GPU memory
- Higher throughput
- Lower serving cost

This is why modern models adopt techniques such as:

- GQA
- MoE
- KV Cache (covered later)
- Quantization (covered later)

These improve serving efficiency without significantly reducing quality.

---

# Example

Training:

```
100 Days

↓

Train Best Model
```

Inference:

```
Millions of Requests

↓

Serve Cheaply
```

Training optimizes quality.

Inference optimizes cost.

---

# 4. Native Multimodality

Traditional LLMs understood only text.

Modern foundation models understand multiple modalities.

Example:

```
Text

Image

Audio

Video
```

Instead of treating each modality separately,

they learn a shared representation.

---

## Example

Prompt:

```
Describe this image.
```

The system processes:

```
Image

↓

Vision Encoder

↓

Shared Embedding Space

↓

LLM

↓

Text Response
```

Similarly,

for speech:

```
Audio

↓

Audio Encoder

↓

Shared Representation

↓

LLM
```

The language model now reasons across multiple input types.

---

# Why Shared Representations Matter

Imagine asking:

> "What animal is shown in this image?"

The Vision Encoder converts pixels into vectors.

The LLM doesn't receive pixels.

It receives embeddings that represent the visual content.

This allows the LLM to reason using the same mechanisms it uses for text.

---

# Modern Examples

Many recent frontier models support multiple modalities.

Examples include:

- GPT
- Gemini
- Claude
- Qwen
- Llama (multimodal variants)

The repository presents native multimodality as a major direction for future AI systems rather than a completely separate architecture.

---

# 5. Putting Everything Together

Let's revisit every major concept learned in this chapter.

---

## Step 1

Text enters the model.

```
Input Tokens
```

---

## Step 2

Tokens become vectors.

```
Embedding Layer
```

---

## Step 3

Positional information is added.

```
RoPE
```

so the model understands token order.

---

## Step 4

Attention begins.

Every token creates

```
Query

Key

Value
```

---

## Step 5

Attention scores are computed.

```
QKᵀ

↓

Scale

↓

Softmax

↓

Attention
```

Every token now knows

which other tokens are important.

---

## Step 6

Multiple attention heads learn different relationships.

```
Grammar

Reasoning

Coreference

Semantics
```

---

## Step 7

GQA reduces memory by allowing Query heads to share Key and Value heads.

---

## Step 8

FFNs perform deeper computation.

Modern models often replace

```
Dense FFN

↓

MoE FFN
```

to reduce inference cost.

---

## Step 9

Residual Connections

+

RMSNorm

stabilize training.

---

## Step 10

Repeat

```
32

48

80

100+

Layers
```

Every layer builds richer contextual representations.

---

## Step 11

Output Layer predicts

```
Next Token
```

The entire pipeline repeats until generation finishes.

---

# 6. Complete Transformer Pipeline

Putting everything together:

```
Input Text
      │
Tokenizer
      │
Embeddings
      │
RoPE
      │
─────────────────────────────
Transformer Layer
─────────────────────────────

Multi-Head Attention

↓

GQA

↓

Residual

↓

RMSNorm

↓

FFN / MoE

↓

Residual

↓

RMSNorm

─────────────────────────────

Repeat N Times

↓

Output Projection

↓

Softmax

↓

Next Token
```

This architecture powers nearly every modern LLM.

---

# Production Perspective

Different optimizations improve different parts of this pipeline.

| Optimization | Improves |
|--------------|----------|
| RoPE | Long-context understanding |
| Multi-Head Attention | Better representation learning |
| GQA | Lower KV memory |
| RMSNorm | Faster inference |
| MoE | Lower compute per token |
| KV Cache | Faster generation *(covered later)* |
| Quantization | Lower GPU memory *(covered later)* |

Notice that every optimization targets a specific bottleneck.

---

# 🎯 Interview Notes

### Why are newer 8B models sometimes better than older 70B models?

Because model quality depends on architecture, training data, optimization strategy, and training compute—not just parameter count.

---

### What are Scaling Laws?

Scaling Laws describe how model performance changes as parameters, training data, and compute increase.

---

### What is Chinchilla Scaling?

It argues that many large models were undertrained and that balancing parameters with sufficient training tokens leads to better performance.

---

### Difference between Training-Optimal and Inference-Optimal?

Training focuses on maximizing model quality.

Inference focuses on minimizing latency, memory usage, and serving cost.

---

### Why is Multimodality important?

Future AI systems need to reason across text, images, audio, and video using a shared representation.

---

# 📝 Key Takeaways

- Bigger models are not always better.
- Scaling Laws help determine how to allocate compute effectively.
- Chinchilla showed that many large models benefit from more training rather than simply more parameters.
- Modern LLMs optimize both architecture and training strategy.
- Native multimodality allows reasoning across different input types.
- Modern production models combine RoPE, GQA, RMSNorm, MoE, and other optimizations to balance quality with serving efficiency.

---

# Final Revision Summary

At this point, you should understand:

### Transformer Fundamentals

- Why RNNs failed
- Why Transformers succeeded
- Encoder vs Decoder
- Token Embeddings

### Attention

- Query
- Key
- Value
- Self-Attention
- Multi-Head Attention

### Efficiency

- GQA
- RMSNorm
- FFN
- MoE

### Position

- Positional Encoding
- RoPE

### Scaling

- Scaling Laws
- Chinchilla
- Training vs Inference Optimization

### Future Directions

- Native Multimodality

---

# What's Next?

The next chapter in the repository is **Inference Optimization**, where you'll study:

- KV Cache (deep dive)
- FlashAttention
- PagedAttention
- Continuous Batching
- Speculative Decoding
- Quantization
- VRAM calculations
- GPU serving architecture

This chapter explains **how companies like OpenAI, Anthropic, Google, and Meta serve LLMs efficiently in production**.

> **Important:** The repository intentionally introduced concepts like GQA and KV Cache at a high level in this chapter. The detailed implementation, memory calculations, and serving optimizations are covered in the next chapter, so your notes should follow that progression to avoid duplication.



