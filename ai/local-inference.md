# Local LLM Inference: Ollama, llama.cpp, GPUs, VRAM, RAM, and Shared Memory

> A practical mental model for understanding what actually happens when you run an LLM on your own computer.

Running an AI model locally can feel surprisingly magical.

You type a command such as:

```bash
ollama run <model>
```

and suddenly a large language model is generating text on your own machine.

But what is actually happening?

Where does the model live? Why does GPU memory matter so much? What happens when the model is bigger than your VRAM? What is shared memory? And where do **Ollama** and **llama.cpp** fit into all of this?

The easiest way to understand local inference is to start with one mental model:

> **A local LLM is essentially a large collection of numbers stored in memory, while your CPU and/or GPU repeatedly perform mathematical operations on those numbers to generate one token at a time.**

Once that clicks, Ollama, llama.cpp, quantization, VRAM, RAM, GPU offloading, and unified memory become much easier to understand.

---

## 1. What does “local inference” mean?

When you use a cloud AI service, the rough architecture looks like this:

```text
Your computer
     │
     ▼
  Internet
     │
     ▼
Cloud server
     │
     ▼
Cloud GPU
     │
     ▼
   Model
     │
     ▼
   Answer
```

With **local inference**, the model and inference workload live on your own machine:

```text
Your computer
     │
     ▼
Local model
     │
     ▼
CPU / GPU
     │
     ▼
   Answer
```

That is the fundamental idea behind local inference.

It can provide useful advantages:

- the model can run without an internet connection;
- prompts and generated text can remain on your machine;
- you control the hardware and runtime;
- you can experiment with models without depending on a hosted API.

Of course, “local” does not automatically mean “nothing ever communicates over the network.” Applications can have their own update, download, telemetry, or integration behavior. The important distinction is that **the inference itself can happen on your hardware**.

---

## 2. What is an LLM, physically?

An LLM is not primarily a giant database of sentences.

At its core, it contains a huge number of learned numerical values called **parameters** or **weights**.

Conceptually:

```text
Model
├── Transformer layer 1
│   ├── weights
│   └── other parameters
├── Transformer layer 2
│   ├── weights
│   └── other parameters
├── ...
└── Transformer layer N
```

When people say that a model has **7 billion parameters**, they mean it contains roughly seven billion learned numerical values.

That number matters because those values have to be stored somewhere while the model is running.

For a simplified example, suppose every parameter were stored as a 16-bit value:

```text
7,000,000,000 parameters × 2 bytes
≈ 14 GB
```

That is already larger than many consumer GPUs.

This is one of the reasons **quantization** is so important for local inference.

---

## 3. Quantization: making models smaller

A model does not necessarily need to store every weight using 16 or 32 bits.

Quantization reduces the numerical precision used to represent weights.

Very roughly:

| Representation | Bits per parameter | Approx. 7B weight storage |
|---|---:|---:|
| FP32 | 32 | ~28 GB |
| FP16 | 16 | ~14 GB |
| 8-bit | 8 | ~7 GB |
| 4-bit | 4 | ~3.5 GB |

These are simplified estimates. Real model files and runtime memory requirements include additional metadata and working memory.

You will encounter names such as:

```text
Q4_K_M
Q5_K_M
Q8_0
FP16
```

The exact schemes are more complicated than the labels suggest, but the first intuition is simple:

> **Quantization trades some numerical precision for much lower memory requirements.**

This is one of the major reasons modern open-weight models can run on consumer hardware. llama.cpp supports multiple quantization levels, including 1.5-, 2-, 3-, 4-, 5-, 6-, and 8-bit integer quantization. citeturn0search3

---

## 4. So what is llama.cpp?

**llama.cpp is an inference engine/runtime for running LLMs locally.**

It is written in C/C++ and is designed to run models efficiently across a wide range of hardware. Its official project describes goals including minimal setup, broad hardware support, quantization, and CPU/GPU acceleration. citeturn0search3

A simplified picture is:

```text
                 Your application
                       │
                       ▼
                    llama.cpp
                       │
              ┌────────┴────────┐
              ▼                 ▼
             CPU               GPU
```

It can make use of CPU instruction sets, Apple Silicon/Metal, NVIDIA CUDA, AMD HIP, Vulkan, SYCL, and other backends. citeturn0search3

This is why llama.cpp is such an important project in local AI: it sits close to the actual inference machinery.

Think of it as the **engine**.

---

## 5. Then what is Ollama?

This is the easiest distinction:

> **llama.cpp is an inference engine; Ollama is a much more user-friendly way to manage and run local models.**

Ollama provides a simple workflow for downloading, running, and integrating models, including a local API. Its documentation describes it as a way to get up and running with models such as Gemma, DeepSeek-R1, Qwen, and others. citeturn0search0

The conceptual stack is:

```text
                 Your application
                       │
                       ▼
                    Ollama
                       │
             ┌─────────┴─────────┐
             │                   │
       Model management       Local API
             │                   │
             └─────────┬─────────┘
                       ▼
                Inference runtime
                       │
                  CPU / GPU
```

For a beginner, the important difference is:

### Ollama

Think:

> “I want to run a model locally without worrying about all the machinery.”

### llama.cpp

Think:

> “I want to understand and control more of the machinery that actually executes the model.”

One correction worth making to the simplified story often told online: **don't treat Ollama and llama.cpp as competing concepts.** They operate at different layers, and the relationship between specific Ollama releases and inference backends can evolve. Ollama's current documentation is the best source for its supported runtime behavior. citeturn0search0

---

# 6. The most important hardware concept: VRAM

Suppose you have a discrete GPU:

```text
GPU
├── GPU compute
└── VRAM
```

**VRAM** is the GPU's dedicated high-speed memory.

For example:

```text
GPU
│
├── GPU compute
│
└── 12 GB VRAM
```

When an LLM runs on the GPU, its weights and other runtime data ideally live in VRAM.

Imagine:

```text
Model weights       5 GB
KV cache             2 GB
Runtime overhead     1 GB
-------------------------
Total                8 GB
```

An 8 GB GPU is now right at the edge.

A 12 GB GPU has some breathing room.

The exact requirements depend heavily on the model, quantization, context length, batch size, runtime, and hardware backend.

---

# 7. RAM is not the same as VRAM

A typical desktop might look like this:

```text
CPU
 │
 └── 32 GB system RAM

GPU
 │
 └── 12 GB VRAM
```

These are separate memory pools on a conventional discrete-GPU system.

So:

```text
32 GB RAM + 12 GB VRAM
```

does **not** mean that the GPU simply has 44 GB of equally fast memory.

This distinction is fundamental.

The GPU's dedicated VRAM is normally much better suited to keeping the model close to the GPU's compute units.

---

# 8. What happens when the model is too big for VRAM?

This is where local inference becomes especially interesting.

Suppose:

```text
Model              20 GB
GPU VRAM            12 GB
System RAM          32 GB
```

The entire model cannot fit in the GPU's dedicated memory.

A runtime can sometimes split the workload:

```text
                 20 GB model
                      │
             ┌────────┴────────┐
             ▼                 ▼
          GPU VRAM          System RAM
            12 GB              8 GB
```

This is commonly described as **CPU/GPU offloading**, **GPU offloading**, or hybrid inference, depending on the runtime and configuration.

llama.cpp explicitly supports CPU+GPU hybrid inference, allowing models larger than available VRAM to be partially accelerated by the GPU. citeturn0search3

But there is a catch:

> **System RAM is not a drop-in replacement for dedicated VRAM.**

Moving data across the CPU/GPU memory boundary can become a bottleneck.

So, as a rough intuition:

```text
Entire working set in VRAM
        ↓
      FAST

Some data in VRAM
Some data in RAM
        ↓
      SLOWER
```

“SLOWER” does not mean “useless.” Hybrid inference can be an excellent way to run models that otherwise would not fit.

---

# 9. Apple Silicon makes this more interesting

Apple Silicon systems use a **unified memory architecture**.

The simplified picture is:

```text
              Unified Memory
             ┌───────────────┐
             │               │
        CPU  │               │  GPU
          ───┤    Memory     ├───
             │               │
             └───────────────┘
```

Instead of a conventional desktop with:

```text
CPU → system RAM
GPU → dedicated VRAM
```

CPU and GPU can work from a shared physical memory pool.

This is one reason Macs with large unified-memory configurations can be surprisingly capable local-inference machines.

But there is an important caveat:

> **Unified memory capacity is not identical to dedicated GPU VRAM performance.**

Capacity and bandwidth are separate considerations.

A machine with 64 GB of unified memory is not automatically equivalent to a machine with 64 GB of high-end discrete GPU VRAM.

---

# 10. What does “shared memory” mean?

This term causes a lot of confusion because it can describe several related but different concepts.

### Integrated GPU + system RAM

A computer with an integrated GPU may use system RAM for graphics:

```text
CPU
 │
 ├── system RAM
 │
 └── integrated GPU
         │
         └── same physical memory
```

The amount available to the GPU may be managed dynamically by the operating system and platform.

### Apple unified memory

This is a more tightly integrated architecture where CPU and GPU share a physical memory pool.

### “Shared GPU memory” in operating-system tools

Some operating systems report both:

```text
Dedicated GPU memory
Shared GPU memory
```

The shared portion generally refers to system memory that the GPU can access.

It should **not** be interpreted as equivalent to having that amount of additional dedicated VRAM.

---

# 11. The four hardware numbers to care about

When evaluating a machine for local LLM inference, four concepts are especially useful.

## 1. VRAM / usable accelerator memory

This is often the first hard constraint for a discrete GPU.

Very roughly:

```text
8 GB    → small/local models
12 GB   → comfortable small/medium models
16 GB   → substantially more flexibility
24 GB   → serious local inference territory
48 GB+  → large-model territory
```

These are broad categories, not hard limits.

## 2. System RAM

RAM becomes especially important when the model doesn't fit entirely in accelerator memory.

A rough intuition:

```text
16 GB   → limiting
32 GB   → good starting point
64 GB   → excellent for many local workloads
128 GB+  → useful for large models
```

Again, the model and context determine what is actually possible.

## 3. Memory bandwidth

This one is often overlooked.

LLM inference repeatedly reads enormous amounts of model data.

So two systems with the same memory capacity can behave very differently if their memory bandwidth differs significantly.

Think:

```text
Memory capacity
      +
Memory bandwidth
      +
Compute
```

rather than simply:

```text
“How many GB does it have?”
```

## 4. Compute

The GPU also has to perform the mathematical operations.

Modern GPUs contain large amounts of parallel compute hardware, which is well suited to the matrix operations used by transformer models.

But raw compute is not the only factor. For many inference workloads, memory capacity and bandwidth can be just as important.

---

# 12. Why GPUs are so good at LLM inference

A CPU might have:

```text
8–32 powerful CPU cores
```

while a GPU can have a very large number of parallel execution units.

LLMs contain huge amounts of parallel mathematical work:

```text
CPU
████████
A smaller number of powerful workers

GPU
████████████████████████████
A much larger number of parallel workers
```

This makes GPUs extremely effective for many inference workloads.

CPUs still matter, though.

They can run models by themselves, and they often participate in hybrid CPU/GPU inference.

---

# 13. CPU inference is not useless

Suppose you have:

```text
64 GB RAM
```

and a quantized model that fits in that memory.

You can potentially run it using CPU inference.

The main problem is usually **speed**, not whether it is possible at all.

You might experience something like:

```text
CPU → a few tokens/sec
GPU → tens of tokens/sec or more
```

The actual numbers can vary enormously with the CPU, GPU, model, quantization, context, and runtime.

The important idea is:

> **CPU inference expands what you can run; GPU inference often determines how quickly you can run it.**

---

# 14. Tokens per second

Local LLM discussions frequently use **tokens/sec** as a performance metric.

If a model generates:

```text
30 tokens/sec
```

it is generating roughly thirty output tokens every second.

A useful intuition:

```text
5 tok/s    → slow
10 tok/s   → usable
20 tok/s   → comfortable
40+ tok/s  → very responsive
```

These aren't universal thresholds. People have different expectations, and tokens/sec isn't the entire performance story.

---

# 15. Prompt processing vs. generation

There is another important distinction.

Imagine you paste a huge document into a model.

The system first processes the input.

This is often called **prefill** or **prompt processing**.

Then the model starts generating new tokens.

That's **decode**.

Conceptually:

```text
Large prompt
     │
     ▼
  PREFILL
     │
     ▼
First generated token
     │
     ▼
   DECODE
     │
     ├── token
     ├── token
     ├── token
     └── ...
```

A system can therefore have:

```text
very fast prompt processing
+
moderate generation speed
```

or another combination.

This matters particularly for long documents and long-context applications.

---

# 16. The other big memory consumer: the KV cache

This is one of the most important ideas after model weights.

You might look at a quantized model and think:

> “The model file is only 5 GB, so my 6 GB GPU should be fine.”

Not necessarily.

During inference, the transformer maintains information about previous tokens called the **KV cache**.

A simplified memory picture is:

```text
VRAM
│
├── Model weights
├── KV cache
├── Temporary tensors
└── Runtime overhead
```

And the KV cache generally grows as the active context grows.

For example:

```text
4K context
   ↓
smaller KV cache

32K context
   ↓
larger KV cache

128K context
   ↓
potentially much larger KV cache
```

So:

> **Model file size is not the same thing as total inference memory requirement.**

A useful simplified formula is:

```text
Total memory ≈
    model weights
  + KV cache
  + runtime / temporary memory
```

That one line explains a huge amount of local-inference behavior.

---

# 17. Model size and context length are different

These two ideas are easy to mix up.

### Model size

Mostly relates to:

```text
number of parameters
×
storage precision
```

### Context length

Describes how much token history the model can work with at a given time.

For example:

```text
7B model
4K context
```

and:

```text
7B model
128K context
```

may have radically different memory behavior during inference.

Long context can therefore become a hardware problem even when the model itself fits comfortably in memory.

---

# 18. What happens when you run Ollama?

When you execute something like:

```bash
ollama run <model>
```

the conceptual sequence is:

```text
                 Ollama
                   │
                   ▼
             Find the model
                   │
                   ▼
              Load model
                   │
                   ▼
       Detect/use available hardware
                   │
             ┌─────┴─────┐
             ▼           ▼
            CPU         GPU
             │           │
             └─────┬─────┘
                   ▼
             Run inference
                   │
                   ▼
             Generate tokens
```

The important point is that Ollama hides much of the low-level machinery.

That is its strength.

---

# 19. What llama.cpp lets you understand

When you move closer to llama.cpp, concepts such as these become important:

```text
model
context
batch size
GPU layers
CPU threads
quantization
KV cache
offloading
CUDA
Metal
Vulkan
```

You can think of a simplified hybrid setup like:

```text
Model
│
├── Layers 0–15
│      ↓
│    GPU
│
└── Layers 16–19
       ↓
      CPU
```

This is one of the reasons llama.cpp is such a useful learning tool: it exposes many of the knobs and architectural decisions hidden by higher-level applications.

---

# 20. Why quantization changes everything

Imagine a hypothetical 70B-parameter model.

At FP16:

```text
70 billion × 2 bytes
≈ 140 GB
```

That is enormous.

At roughly 4 bits per parameter:

```text
70 billion × 0.5 bytes
≈ 35 GB
```

The real number will be somewhat larger because of metadata and other requirements, but the difference is enormous.

Suddenly a large model becomes conceivable on a machine with:

```text
64 GB RAM
```

or across multiple GPUs.

This is why quantization is one of the foundational technologies behind practical local LLM inference.

---

# 21. The local-inference stack

At this point, we can put everything together.

```text
                       LOCAL AI
                          │
          ┌───────────────┴───────────────┐
          │                               │
        MODEL                           RUNTIME
          │                               │
   ┌──────┴──────┐               ┌────────┴────────┐
   │             │               │                 │
parameters   quantization      Ollama          llama.cpp
                                      │
                                      │
                              inference execution
                                      │
                                  HARDWARE
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                   CPU               GPU              Memory
                                      │                 │
                                  compute        ┌───────┴──────┐
                                                  │              │
                                                 VRAM           RAM
```

This is the mental model I would keep.

---

# 22. Don't confuse the model, the runtime, and the interface

These are three different things.

### The model

The neural network itself.

Examples include:

```text
Llama
Qwen
Gemma
Mistral
DeepSeek
```

### The inference engine

Software that actually executes the model.

One important example is:

```text
llama.cpp
```

### The user-facing runtime/application

Software that makes managing and using models easier.

One important example is:

```text
Ollama
```

So the conceptual relationship is:

```text
Model
  ↓
Inference engine
  ↓
Hardware
```

with a higher-level application sitting around that stack to make it easier to use.

---

# 23. Training vs. inference

Everything discussed here is primarily about **inference**.

Training is a completely different scale of problem.

### Inference

```text
Already-trained model
       ↓
Load weights
       ↓
Generate output
```

### Training

```text
Huge dataset
     ↓
Forward pass
     ↓
Calculate loss
     ↓
Backpropagation
     ↓
Update billions of weights
     ↓
Repeat
```

You can therefore run surprisingly capable models locally without owning anything close to the hardware required to train them from scratch.

---

# 24. A practical learning path

If you're learning local AI from scratch, don't start by trying to understand every CUDA flag.

Use this progression.

## Level 1 — Ollama

Start with a model and learn:

```text
model size
quantization
context
RAM usage
VRAM usage
tokens/sec
```

The official Ollama documentation is a good starting point for installation, model management, APIs, and supported workflows. citeturn0search0

## Level 2 — Hardware

Learn:

```text
VRAM
RAM
memory bandwidth
GPU compute
CPU threads
PCIe
unified memory
```

At this stage, you should be able to look at a computer's specifications and make a rough prediction about what kinds of models it can run.

## Level 3 — llama.cpp

Then explore:

```text
GPU offloading
layers
batch size
KV cache
quantization
CPU/GPU split
```

The llama.cpp project is particularly useful here because its documentation and command-line tooling expose many of these concepts directly. citeturn0search3

## Level 4 — Transformer internals

Finally learn:

```text
tokens
embeddings
attention
Q / K / V
transformer layers
MLP
logits
sampling
KV cache
```

At that point, local inference stops feeling like magic.

You start seeing it as a large, understandable computational pipeline.

---

# 25. The five ideas to remember

If you remember only five things, remember these:

### 1. Model weights need memory

```text
model → RAM / VRAM
```

### 2. VRAM is the GPU's fast dedicated memory

More VRAM generally gives you more room to keep the model and inference state close to the GPU.

### 3. A model can sometimes be split between GPU and RAM

That makes larger models runnable, but crossing the CPU/GPU memory boundary can reduce performance.

### 4. Quantization makes models dramatically smaller

Fewer bits per parameter means less storage and memory pressure, usually with some trade-off in numerical precision and potentially model quality.

### 5. Ollama and llama.cpp operate at different layers

Think:

```text
Ollama     → convenience
llama.cpp  → inference machinery
model      → neural network
hardware   → where the computation happens
```

And keep this formula in your head:

```text
Total inference memory ≈
    model weights
  + KV cache
  + runtime / temporary memory
```

That is the foundation for understanding why one model flies on one computer, barely runs on another, and doesn't fit at all on a third.

---

## Where to go next

The natural next step is to take **one real computer** and work through the numbers.

For example:

```text
CPU:       ?
RAM:       ?
GPU:       ?
VRAM:      ?
Memory bandwidth: ?
```

Then take a real model:

```text
7B / 8B
14B
32B
70B
```

and calculate:

```text
Does it fit?
How much VRAM does it need?
How much RAM?
What does Q4 vs Q5 vs Q8 change?
What happens to the KV cache?
How many layers can fit on the GPU?
What kind of tokens/sec should we expect?
```

That exercise turns the abstract concepts into something you can actually use when choosing hardware or models.

---

## Sources and further reading

- [Ollama documentation](https://docs.ollama.com/) — official documentation and getting-started material. citeturn0search0
- [llama.cpp repository](https://github.com/ggml-org/llama.cpp) — official source, documentation, hardware backends, quantization, and examples. citeturn0search3

---

*This article was generated with assistance from **ChatGPT (GPT-5.6 Luna)** on **September 2, 2026**.*
