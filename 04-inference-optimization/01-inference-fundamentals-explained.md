# PEFT, LoRA & QLoRA — Detailed Interview Notes

## 1. PEFT

**PEFT = Parameter-Efficient Fine-Tuning.**

PEFT is a family of techniques for adapting a pretrained LLM while updating only a small fraction of its parameters.

Common approaches include:

- LoRA
- QLoRA
- Prefix Tuning
- Prompt Tuning
- Adapter-based methods

The most important for practical AI Engineer interviews are **LoRA and QLoRA**.

---

## 2. Why PEFT?

Suppose a pretrained model has:

```text
70B parameters
```

Traditional fine-tuning updates the model weights:

```text
Pretrained Model
      ↓
Training Data
      ↓
Update ALL weights
      ↓
Fine-tuned Model
```

For very large models, this requires substantial GPU memory, compute, training time, and checkpoint storage.

The core PEFT idea is:

> Keep most of the pretrained model frozen and train only a small number of additional parameters.

```text
Base Model
    ↓
Freeze almost everything
    +
Train small adapter
```

---

# 3. Full Fine-Tuning

Let the original weights be:

```text
W
```

After training:

```text
W' = W + ΔW
```

where:

- `W` = pretrained weights
- `ΔW` = learned change
- `W'` = final weights

Full fine-tuning learns the complete `ΔW`.

With billions of parameters, this becomes expensive.

---

# 4. LoRA

**LoRA = Low-Rank Adaptation.**

LoRA avoids learning the entire `ΔW`.

Instead:

```text
ΔW ≈ BA
```

Therefore:

```text
W' = W + BA
```

`A` and `B` are much smaller matrices than `W`.

The original `W` remains frozen.

---

# 5. Why "Low Rank"?

Suppose:

```text
W = 4096 × 4096
```

Full matrix:

```text
4096 × 4096
= 16,777,216 parameters
```

Choose LoRA rank:

```text
r = 8
```

Then:

```text
A = 8 × 4096
B = 4096 × 8
```

Trainable parameters:

```text
8 × 4096 + 4096 × 8
= 65,536
```

Compare:

```text
Full update: 16,777,216
LoRA update:     65,536
```

This is the main reason LoRA is parameter-efficient.

---

# 6. LoRA Forward Pass

Normal layer:

```text
Y = XW
```

LoRA:

```text
Y = X(W + BA)
```

or:

```text
Y = XW + XBA
```

In practical LoRA, the adapter is commonly scaled:

```text
Y = X(W + (α/r)BA)
```

where:

- `r` = rank
- `α` = scaling factor

---

# 7. What Gets Trained?

With LoRA:

```text
Base Model Weights
        ↓
      FROZEN
```

Only the adapter matrices are trained:

```text
A
B
```

So the training process is:

```text
Loss
 ↓
Gradients
 ↓
LoRA parameters only
```

The original model weights are not updated.

---

# 8. Where Is LoRA Applied?

LoRA is commonly applied to transformer linear projections, especially attention projections:

```text
Wq
Wk
Wv
Wo
```

For example:

```text
Q = X(Wq + BqAq)
K = X(Wk + BkAk)
V = X(Wv + BvAv)
```

Some configurations target only `q_proj` and `v_proj`; others target more attention or MLP projections.

The choice depends on the model, task, memory budget, and desired adaptation.

---

# 9. Multiple LoRA Adapters

A major practical advantage:

```text
                 Base LLM
                /    |                   /     |               Finance   Legal   Support
            LoRA     LoRA     LoRA
```

The large base model is shared.

Only small adapters differ.

This is much more storage-efficient than keeping a separate full model for every task.

---

# 10. LoRA Inference

You can conceptually use:

```text
Base model + LoRA adapter
```

or merge the adapter into the base weights:

```text
W' = W + (α/r)BA
```

Keeping adapters separate is useful when switching between multiple specialized adapters.

---

# 11. QLoRA

**QLoRA = Quantized LoRA.**

QLoRA combines:

```text
Quantization
+
LoRA
```

The base model is loaded in a low-bit quantized representation, commonly 4-bit, and kept frozen.

Then LoRA adapters are trained:

```text
Base LLM
   ↓
4-bit quantized
   ↓
Frozen
   +
LoRA adapter
   ↓
Train adapter
```

---

# 12. Why QLoRA?

Consider a 70B model.

At approximately BF16:

```text
70B × 2 bytes
≈ 140 GB
```

That is only raw weight memory. Training additionally requires memory for activations, optimizer state, gradients for trainable parameters, and runtime overhead.

At approximately 4 bits:

```text
4 bits = 0.5 bytes

70B × 0.5 bytes
≈ 35 GB
```

This is an approximate raw-weight calculation. Actual memory is higher because of quantization metadata and runtime/training memory.

The important idea:

> Quantization dramatically reduces the memory needed to hold the frozen base model.

---

# 13. QLoRA Is Not "Everything Is 4-bit"

A common mistake is to say:

> QLoRA means the entire training process is 4-bit.

Better:

```text
Base model
→ quantized
→ frozen

LoRA adapters
→ trainable
```

QLoRA is designed to make large-model fine-tuning much more memory-efficient.

---

# 14. NF4

**NF4 = NormalFloat 4-bit.**

It is a 4-bit quantization format designed for neural-network weights and is strongly associated with QLoRA.

Interview takeaway:

> NF4 provides a compact 4-bit representation designed to preserve useful information from normally distributed model weights.

---

# 15. Double Quantization

QLoRA can also quantize the constants/statistical information used during quantization.

Conceptually:

```text
Weights
  ↓
Quantize
  ↓
Quantization constants
  ↓
Quantize those constants too
```

This is called **double quantization** and provides additional memory savings.

---

# 16. Paged Optimizers

Training can produce memory spikes.

QLoRA uses paged optimizer techniques to manage optimizer memory more efficiently and reduce memory pressure during training.

Interview takeaway:

> Paged optimizers help make memory usage more manageable during large-model QLoRA training.

---

# 17. LoRA vs QLoRA

| Feature | LoRA | QLoRA |
|---|---|---|
| PEFT | Yes | Yes |
| Base model frozen | Yes | Yes |
| LoRA adapters trained | Yes | Yes |
| Base model quantized | Not necessarily | Yes |
| Main goal | Efficient adaptation | Even lower memory usage |
| Core idea | Low-rank adapters | Quantized base + LoRA |

---

# 18. Full Fine-Tuning vs LoRA vs QLoRA

```text
Fine-Tuning
     |
     +------------------+------------------+
     |                  |                  |
    Full               LoRA              QLoRA
     |                  |                  |
Train all            Freeze base       Freeze base
weights              Train adapters    Train adapters
                                        +
                                      Quantize
                                      base model
```

---

# 19. RAG vs PEFT

These solve different problems.

### RAG

RAG provides external knowledge at inference time:

```text
Documents
   ↓
Chunk
   ↓
Embedding
   ↓
Vector DB
   ↓
Retrieve
   ↓
LLM
```

Use RAG when:

- knowledge changes frequently
- the model needs private/company documents
- answers need source grounding
- you need citations

### PEFT / Fine-Tuning

Fine-tuning adapts model behavior:

```text
Training examples
       ↓
    LoRA/QLoRA
       ↓
Adapted model
```

Useful for:

- response style
- specialized behavior
- task adaptation
- consistent output formats
- domain-specific patterns

---

# 20. RAG + Fine-Tuning Can Coexist

They are not mutually exclusive.

```text
User
 ↓
Retriever
 ↓
Relevant Context
 ↓
Fine-tuned LLM
 ↓
Answer
```

RAG provides the knowledge.

Fine-tuning provides behavior adaptation.

---

# 21. Interview Question: Why LoRA Instead of Full Fine-Tuning?

Strong answer:

> Full fine-tuning updates all model parameters, which is expensive in GPU memory, compute, training time, and checkpoint storage. LoRA freezes the pretrained model and learns a small low-rank update using adapter matrices. This dramatically reduces the number of trainable parameters while still allowing the model to adapt to a task.

---

# 22. Interview Question: What Is QLoRA?

Strong answer:

> QLoRA is a parameter-efficient fine-tuning method that combines LoRA with quantization. The pretrained base model is loaded in a low-bit representation, commonly 4-bit, and kept frozen, while small LoRA adapters are trained. This significantly reduces the memory required for fine-tuning large models.

---

# 23. Interview Question: What Is the Difference Between LoRA and QLoRA?

```text
LoRA:
Base model → frozen
Adapter    → trainable

QLoRA:
Base model → quantized + frozen
Adapter    → trainable
```

The adaptation mechanism remains LoRA. QLoRA adds quantization to reduce memory usage.

---

# 24. Interview Question: Why Does LoRA Reduce Trainable Parameters?

Instead of learning:

```text
ΔW
```

directly, LoRA learns:

```text
ΔW ≈ BA
```

If:

```text
W = d × d
```

and:

```text
A = r × d
B = d × r
```

then:

```text
Full update:
d²

LoRA:
2dr
```

When:

```text
r << d
```

the reduction is enormous.

---

# 25. Interview Question: Does LoRA Modify the Original Model?

During LoRA training:

```text
Original weights → frozen
```

The learned adaptation is stored in the adapter.

You can:

```text
keep adapter separate
```

or:

```text
merge adapter into base weights
```

---

# 26. Interview Question: Can Multiple LoRAs Share One Base Model?

Yes.

```text
                 Base Model
                    |
       +------------+------------+
       |            |            |
    Finance       Legal       Support
      LoRA          LoRA         LoRA
```

This is useful when serving multiple specialized behaviors from one base model.

---

# 27. Interview Question: Why Is QLoRA Useful?

Large models may not fit comfortably in GPU memory.

QLoRA:

```text
Quantized base model
+
small LoRA adapters
```

reduces both:

- memory needed for the frozen base
- number of trainable parameters

That makes fine-tuning much more accessible.

---

# 28. Connection to Transformers

You are learning transformer internals, so connect LoRA to attention.

Attention contains projections:

```text
Q = XWq
K = XWk
V = XWv
```

LoRA can modify these:

```text
Q = X(Wq + BqAq)

K = X(Wk + BkAk)

V = X(Wv + BvAv)
```

The original weights remain frozen.

This is why understanding linear layers and attention makes LoRA much easier to understand.

---

# 29. Connection to Quantization and VRAM

You can connect your VRAM calculations:

```text
BF16 ≈ 2 bytes/parameter
INT8 ≈ 1 byte/parameter
4-bit ≈ 0.5 bytes/parameter
```

Therefore:

```text
Quantization
     ↓
Less base-model memory
```

while:

```text
LoRA
     ↓
Fewer trainable parameters
```

QLoRA combines both:

```text
                  QLoRA
                    |
          +---------+---------+
          |                   |
    Quantization            LoRA
          |                   |
 Reduce base-model      Reduce trainable
    memory               parameters
```

---

# 30. End-to-End QLoRA Workflow

```text
Pretrained LLM
      ↓
Quantize base model
      ↓
Freeze base model
      ↓
Attach LoRA adapters
      ↓
Training data
      ↓
Forward pass
      ↓
Loss
      ↓
Backpropagation
      ↓
Update LoRA parameters only
      ↓
Trained adapter
```

Result:

```text
Base Model + LoRA Adapter
```

---

# 31. Simple Mental Model

Think of a large pretrained model as a giant machine.

### Full fine-tuning

```text
Modify the entire machine.
```

### LoRA

```text
Keep the machine unchanged
+
attach a small control module.
```

### QLoRA

```text
Keep the machine unchanged
+
store the machine more compactly
+
attach the small control module.
```

---

# 32. Final Mental Map

```text
                    LLM ADAPTATION
                           |
             +-------------+-------------+
             |                           |
            RAG                     Fine-Tuning
             |                           |
      External knowledge                 |
             |                           |
        Vector DB                        |
                                         |
                                +--------+--------+
                                |                 |
                               Full              PEFT
                               FT                  |
                                                 |
                                      +----------+----------+
                                      |                     |
                                     LoRA                  QLoRA
                                      |                     |
                              Low-rank adapters      Quantized base
                              + frozen base          + LoRA
```

---

# 33. Three Definitions to Memorize

### PEFT

> A family of techniques that fine-tune a pretrained model by updating only a small subset of parameters.

### LoRA

> A PEFT technique that freezes the base model and learns a low-rank approximation of the weight update using small adapter matrices.

### QLoRA

> A technique that combines a quantized frozen base model with trainable LoRA adapters to make large-model fine-tuning substantially more memory-efficient.

---

# 34. One-Minute Interview Answer

> PEFT stands for Parameter-Efficient Fine-Tuning. Instead of updating all parameters of a pretrained LLM, PEFT methods update only a small number of parameters. LoRA is a popular PEFT technique where the base model is frozen and the weight update is represented using two small low-rank matrices, approximately `ΔW = BA`. QLoRA combines LoRA with low-bit quantization of the frozen base model, commonly 4-bit, to reduce memory requirements further. RAG is different: RAG supplies external knowledge at inference time, whereas LoRA/QLoRA adapt the model's behavior during training.
