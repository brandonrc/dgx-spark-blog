---
title: "KV Cache Deep Dive: The 2x Reduction Mystery"
date: 2025-11-08T15:00:00-00:00
draft: false
tags: ["dgx-spark", "kv-cache", "memory", "tensorrt-llm", "deep-dive"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "Why do Docker containers allocate roughly half the KV cache of native execution? And why do all models converge to ~44 GiB in native mode? Let's dig deep into the memory allocation mechanics."
---

## The Pattern That Demands Explanation

In [Part 4](../04-the-data), I showed you 60 benchmark runs that revealed a consistent pattern. But one table in particular kept me up at night:

| Model | Native KV | Container KV | Reduction Factor |
|-------|-----------|--------------|------------------|
| DeepSeek-7B | 44.31 GiB | 16.57 GiB | **2.7x less** |
| GPT-OSS-120B | 43.19 GiB | 23.65 GiB | **1.8x less** |
| Qwen2.5-72B | 44.71 GiB | 26.72 GiB | **1.7x less** |

Two questions immediately jump out:

1. **Why do containers consistently allocate ~2x less KV cache?** (The main mystery)
2. **Why do all native runs converge to ~44 GiB?** (The secondary puzzle)

Let's answer both.

## Question 1: Why ~2x Less KV Cache in Containers?

### The High-Level Answer

From [Part 3](../03-unified-memory-revelation), you know Docker's cgroups double-count unified memory. But **how does that translate to exactly 40-60% less KV cache?**

The answer lies in how TensorRT-LLM allocates KV cache memory.

### TensorRT-LLM's Memory Budget Calculation

When TensorRT-LLM starts up, it performs a memory budget calculation:

```
1. Query total available memory
2. Load model weights into unified memory
3. Allocate space for activations and temporary buffers
4. Calculate remaining "free" memory
5. Allocate KV cache = f(remaining memory)
```

Step 1 is where Docker breaks everything.

### Native Execution: Clean Memory View

In native/chroot execution, TensorRT-LLM queries memory directly via CUDA:

```
┌─────────────────────────────────────────────┐
│  Total Grace Hopper Unified Memory: 128 GB  │
│                                             │
│  ┌─────────────┐  ┌──────────┐  ┌────────┐ │
│  │   Model     │  │  System  │  │  FREE  │ │
│  │   Weights   │  │  Reserve │  │        │ │
│  │   ~15 GB    │  │   ~8 GB  │  │ ~105GB │ │
│  └─────────────┘  └──────────┘  └────────┘ │
│                                             │
│  TensorRT-LLM sees: 105 GB available        │
└─────────────────────────────────────────────┘
```

**Native allocation logic:**
- Total available: ~105 GB
- Model + activations: ~60-65 GB
- Remaining: ~40-45 GB
- **KV cache allocated: ~44 GB** ✓

Clean, simple, accurate.

### Container Execution: Docker's Interference

In Docker containers, TensorRT-LLM still uses CUDA APIs, but Docker's cgroup accounting pollutes the results:

```
┌─────────────────────────────────────────────┐
│  Total Unified Memory: 128 GB               │
│                                             │
│  ┌─────────────┐  ┌──────────┐  ┌────────┐ │
│  │   Model     │  │  System  │  │  FREE  │ │
│  │   Weights   │  │  Reserve │  │        │ │
│  │   ~15 GB    │  │   ~8 GB  │  │ ~105GB │ │
│  └─────────────┘  └──────────┘  └────────┘ │
│       ↓                                     │
│  Docker cgroup sees this as                 │
│  "container RAM usage"                      │
│       ↓                                     │
│  ┌──────────────────────────────┐          │
│  │  Docker's view:              │          │
│  │  "Container using 70GB RAM"  │          │
│  │  "Only 58GB left for alloc"  │          │
│  └──────────────────────────────┘          │
│                                             │
│  TensorRT-LLM sees: ~60 GB available        │
│  (Reduced by Docker's accounting!)          │
└─────────────────────────────────────────────┘
```

**Container allocation logic:**
- Docker reports: ~60 GB available (wrong!)
- Model + activations: ~60-65 GB
- Remaining: ~0-5 GB (uh oh...)
- **KV cache allocated: ~16-26 GB** (conservative!)

TensorRT-LLM plays it safe and allocates less KV cache to avoid OOM errors.

### The Math: Where Does the Overhead Come From?

Look at this correlation from [Part 5](../05-what-we-learned):

| Model | Container Overhead | KV Cache Reduction |
|-------|-------------------|-------------------|
| DeepSeek-7B | **+30.83 GiB** | **-27.74 GiB** |
| Qwen-72B | **+19.99 GiB** | **-17.99 GiB** |
| GPT-OSS-120B | **+21.71 GiB** | **-19.54 GiB** |

Notice: **Container overhead ≈ KV cache reduction** (within 1-3 GB)

This isn't a coincidence. Here's what's happening:

**Native memory allocation:**
```
Total:     128 GB
System:     ~8 GB (OS, drivers, etc.)
Model:     ~15 GB (weights)
Workspace: ~45 GB (activations, temp buffers)
KV Cache:  ~44 GB
Available: ~16 GB (safety margin)
────────────────
Total:     128 GB ✓
```

**Container memory allocation:**
```
Total:     128 GB
System:     ~8 GB (OS, drivers, etc.)
Model:     ~15 GB (weights)
Docker:    +20-30 GB (phantom overhead from double-counting!)
Workspace: ~45 GB (activations, temp buffers)
KV Cache:  ~16-26 GB (reduced!)
Available: ~14-24 GB (same safety margin)
────────────────
Total:     128 GB ✓
```

Docker's double-counting creates **phantom overhead** that steals memory from the KV cache budget.

### Why Not 50% Exactly?

You might expect exactly 50% reduction if Docker was reporting half the memory. But the reduction varies (1.7x - 2.7x) because:

1. **Base model size differs** - Smaller models have proportionally more KV cache
2. **Docker overhead scales** - Larger allocations = more double-counting
3. **TensorRT-LLM is conservative** - It leaves safety margins

The pattern: **Smaller models suffer worse** because the fixed Docker overhead consumes a larger percentage of their memory budget.

## Question 2: Why Do All Natives Converge to ~44 GiB?

This is the secondary mystery that caught my attention:

| Model Size | Parameters | Native KV Cache |
|-----------|-----------|----------------|
| DeepSeek | 7B | 44.31 GiB |
| Qwen | 72B | 44.71 GiB |
| GPT-OSS | 120B | 43.19 GiB |

Wait... the 7B model uses essentially the **same KV cache** as the 120B model?

### Why This Seems Wrong

Intuitively, you'd expect:
- Larger models → More parameters → More layers → More KV cache needed

But that's not what the data shows!

### The Explanation: Available Memory Ceiling

TensorRT-LLM doesn't allocate KV cache based on **model size**. It allocates based on **available memory after loading the model**.

Let's look at the native memory breakdown:

**DeepSeek-7B (Small model):**
```
Total memory:       128 GB
Model weights:      ~7 GB (small!)
Activations:        ~10 GB
System overhead:    ~8 GB
Available:          ~103 GB
────────────────────────────
KV cache allocated: 44.31 GB (43% of available)
Safety margin:      ~58 GB
```

**Qwen2.5-72B (Large model):**
```
Total memory:       128 GB
Model weights:      ~45 GB (large!)
Activations:        ~35 GB
System overhead:    ~8 GB
Available:          ~40 GB
────────────────────────────
KV cache allocated: 44.71 GB (112% of calculated available!)
Safety margin:      Minimal
```

Wait, that math doesn't work...

### The Real Answer: TensorRT-LLM's Allocation Strategy

After analyzing the pattern, I believe TensorRT-LLM uses a **ceiling-based allocation strategy**:

```python
# Pseudocode for TensorRT-LLM KV cache allocation
def allocate_kv_cache():
    total_memory = query_total_unified_memory()  # ~128 GB

    # Load model first
    model_memory = load_model_weights()

    # Calculate available
    available = total_memory - model_memory - system_reserve

    # Allocate KV cache with ceiling
    max_kv_cache = 44 GB  # Hardcoded or configured ceiling?
    kv_cache = min(available * 0.6, max_kv_cache)

    return kv_cache
```

This would explain why:
- Small models: Have tons of available memory, but hit the 44 GB ceiling
- Large models: Have less available memory, but still allocate close to 44 GB

### Why a Ceiling Exists

There are good engineering reasons for a KV cache ceiling:

1. **Prevents memory thrashing** - Too large a cache can hurt performance
2. **Reserves memory for batching** - Need room for concurrent requests
3. **Conservative defaults** - Better to be safe than OOM
4. **Hardware limitations** - Memory bandwidth or cache line considerations

### The 72B Anomaly

Notice Qwen-72B has **the highest native KV cache** (44.71 GB). This might be because:
- Its model architecture is more memory-efficient
- Quantization or sparsity reduces weight memory
- TensorRT-LLM optimizations for this model family

But it's still **very close** to the ~44 GB ceiling.

## What This Means for Production

Understanding these two patterns has huge implications:

### 1. Container KV Cache Reduction Matters More at Scale

For small workloads (like my benchmark: 50 requests, 128 tokens), even 16 GB of KV cache is enough.

But in production:
- **8k context window**: You'll hit the limit fast
- **32k context window**: Containers will OOM or refuse requests
- **128k context window**: Forget it - you need native execution

**The 2x KV cache reduction becomes a 2x throughput bottleneck at scale.**

### 2. Model Size Doesn't Predict KV Cache Availability

The ~44 GB convergence means:
- Small models (7B) don't get "extra" KV cache just because they're small
- Large models (72B+) aren't starved of KV cache just because they're large

**Recommendation**: Choose models based on:
- Accuracy requirements (obviously)
- Weight memory vs KV cache tradeoff
- Don't assume bigger = less memory for serving

### 3. Docker is OK for Specific Use Cases

If you're running:
- Short contexts (≤2k tokens)
- Low concurrency (1-4 simultaneous requests)
- Small batches

Then Docker's reduced KV cache might be acceptable. But know you're leaving **40-60% of serving capacity on the table**.

### 4. Native Execution for Maximum Throughput

For production serving with:
- Long contexts (8k+)
- High concurrency (10+ simultaneous users)
- Large batch sizes

**Use native/chroot execution**. The 2x KV cache advantage translates directly to 2x serving capacity.

## Visual Summary

Let me show you the full picture:

```
┌────────────────────────────────────────────────────────┐
│  NATIVE EXECUTION (Optimal)                           │
│  ┌──────────┬─────────────┬──────────────────────┐   │
│  │ Model    │ Activations │  KV Cache (~44 GB)   │   │
│  │ 7-45 GB  │  10-35 GB   │  ████████████████    │   │
│  └──────────┴─────────────┴──────────────────────┘   │
│  Clean memory view = Maximum KV cache                │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│  CONTAINER EXECUTION (Reduced)                        │
│  ┌──────────┬──────────┬─────────────┬──────────┐    │
│  │ Model    │ Docker   │ Activations │ KV Cache │    │
│  │ 7-45 GB  │ Phantom  │  10-35 GB   │ (~20 GB) │    │
│  │          │ +20-30GB │             │  ████     │    │
│  └──────────┴──────────┴─────────────┴──────────┘    │
│  Docker overhead = Stolen KV cache budget             │
└────────────────────────────────────────────────────────┘
```

## Key Takeaways

**On the ~2x container reduction:**
1. Docker's cgroup double-counts unified memory
2. TensorRT-LLM sees "less available" memory
3. Conservatively allocates smaller KV cache
4. Container overhead ≈ KV cache reduction (almost 1:1)
5. Smaller models suffer worse reduction ratios

**On the ~44 GiB native convergence:**
1. TensorRT-LLM likely has a KV cache ceiling (~44 GB)
2. Small models hit the ceiling despite having more free memory
3. Large models allocate near the ceiling despite tight budgets
4. This is good engineering: prevents pathological cases
5. Model size doesn't predict KV cache availability

## What's Next?

This deep dive answered the "what" and "why" of the KV cache patterns. But there are still open questions:

- Can we configure TensorRT-LLM's KV cache ceiling?
- Can Docker be patched to handle unified memory correctly?
- Do other unified memory systems (AMD MI300X) show the same pattern?
- What's the optimal KV cache size for different workloads?

These are questions for **Phase 2** of the investigation. For now, the message is clear:

**If you're running large language models on Grace Hopper in production, understand your KV cache allocation. It's the difference between maximal throughput and leaving half your serving capacity on the table.**

---

**Previous**: [← Part 5: What I Learned (And What's Next)](../05-what-we-learned)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)
**Interactive Charts**: [Results Dashboard](https://brandonrc.github.io/benchmark-spark/)

---

**Got questions or observations?** Open an issue or discussion on the [GitHub repo](https://github.com/brandonrc/benchmark-spark/discussions)!
