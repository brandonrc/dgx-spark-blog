---
title: "The Data: 60 Runs Don't Lie"
date: 2025-11-08T13:00:00-00:00
draft: false
tags: ["dgx-spark", "benchmarking", "data", "results"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "I ran 60 comprehensive benchmarks across three different model sizes. The results are consistent, surprising, and prove that unified memory + Docker cgroups = significant overhead."
---

## The Comprehensive Test Plan

Anecdotes are interesting. Single data points are suggestive. But **60 benchmark runs** across multiple models? That's science.

Here's what I did:

### Test Matrix
- **3 Models**:
  - DeepSeek-R1-Distill-Qwen-7B (7 billion parameters)
  - Qwen2.5-72B-Instruct (72 billion parameters)
  - GPT-OSS-120B (120 billion parameters, MXFP4 quantized)
- **2 Environments**: Native (chroot) vs Container (Docker)
- **10 Iterations** per model per environment
- **Total**: 60 benchmark runs

### Methodology
- **Framework**: TensorRT-LLM (`trtllm-bench` CLI)
- **Workload**: 50 requests, 128 output tokens per request
- **Cooldown**: 5 minutes + GPU temp check (< 45°C) between runs
- **Duration**: ~14 hours total
- **Metrics**: Peak memory, KV cache, throughput, latency, temperature

## DeepSeek-7B Results

My smallest model, but the most shocking results:

| Metric | Native | Container | Difference |
|--------|--------|-----------|------------|
| **Peak Memory** | 70.47 GiB | 101.30 GiB | **+30.83 GiB (44% overhead!)** |
| **KV Cache** | 44.31 GiB | 16.57 GiB | **-27.74 GiB (63% less!)** |
| **Throughput** | 119.79 tok/s | 119.40 tok/s | 0.3% difference |
| **Std Dev (σ)** | 0.55 | 0.16 | Very stable |

**Analysis**: The 7B model shows the **largest overhead** at 30.8GB. Container KV cache is reduced to only **37% of native**. That's massive.

## GPT-OSS-120B Results

The largest model (120B params, though MoE means 5.1B active):

| Metric | Native | Container | Difference |
|--------|--------|-----------|------------|
| **Peak Memory** | 71.72 GiB | 93.43 GiB | **+21.71 GiB (30% overhead)** |
| **KV Cache** | 43.19 GiB | 23.65 GiB | **-19.54 GiB (45% less)** |
| **Throughput** | 120.26 tok/s | 120.41 tok/s | -0.1% difference |
| **Std Dev (σ)** | 0.32 | 0.54 | Very stable |

**Analysis**: Despite being the largest model, GPT-OSS shows moderate overhead due to MXFP4 quantization. Native still has **1.8x more KV cache**.

## Qwen2.5-72B Results

The 72B parameter model - the sweet spot:

| Metric | Native | Container | Difference |
|--------|--------|-----------|------------|
| **Peak Memory** | 70.03 GiB | 90.02 GiB | **+19.99 GiB (29% overhead)** |
| **KV Cache** | 44.71 GiB | 26.72 GiB | **-17.99 GiB (40% less)** |
| **Throughput** | 119.33 tok/s | 119.51 tok/s | -0.2% difference |
| **Std Dev (σ)** | 0.28 | 0.20 | Very stable |

**Analysis**: Qwen shows **the most efficient memory usage** of all models. Native mode has the **highest KV cache** at 44.71 GiB - even more than the 7B model!

## The Pattern: Container Overhead Scales

Look at the overhead across models:

| Model | Size | Overhead | Pattern |
|-------|------|----------|---------|
| DeepSeek | 7B | **+30.83 GiB (44%)** | Highest % |
| Qwen | 72B | **+19.99 GiB (29%)** | Middle |
| GPT-OSS | 120B | **+21.71 GiB (30%)** | Middle |

**Insight**: Overhead appears proportional to base memory usage, not model size. This suggests Docker's cgroup accounting scales with allocation size, confirming our unified memory double-counting theory.

## Performance Parity: The Good News

Across all 60 runs, performance was virtually identical:

```
Throughput Range:
- Native:    119.33 - 120.26 tokens/sec
- Container: 119.40 - 120.51 tokens/sec
- Difference: < 0.3% (within margin of error)
```

**Standard deviations** were incredibly low (σ < 0.6 tokens/sec), showing:
- Excellent thermal management
- Consistent GPU performance
- No thermal throttling

## The KV Cache Revelation

This is the real story. Across all models:

| Model | Native KV | Container KV | Ratio |
|-------|-----------|--------------|-------|
| DeepSeek | 44.31 GiB | 16.57 GiB | **2.7x more** |
| GPT-OSS | 43.19 GiB | 23.65 GiB | **1.8x more** |
| Qwen | 44.71 GiB | 26.72 GiB | **1.7x more** |

Native mode provides **1.7-2.7x more KV cache** across the board. This is huge for:
- Longer context windows
- Higher batch sizes
- Better concurrent request handling
- Overall throughput scaling

## Temperature Analysis

Interestingly, containers ran slightly cooler:

| Environment | Avg End Temp | Range |
|-------------|-------------|-------|
| Native | 60.6°C | 59-61°C |
| Container | 58.6°C | 57-59°C |

**Why?** Container overhead means more time in cooldown between runs. Actual compute time is the same, but total wall time is longer due to additional memory management.

## Data Quality and Reproducibility

All 60 runs completed successfully with:
- ✅ **0 failures** - Every benchmark completed
- ✅ **Consistent results** - Very low standard deviations
- ✅ **Proper thermal management** - No throttling
- ✅ **Comprehensive logging** - Full metadata saved

Raw data available: [GitHub - benchmark-spark/results](https://github.com/brandonrc/benchmark-spark/tree/main/results/comprehensive)

## Interactive Charts

Want to explore the data visually? Check out our [interactive results page](https://brandonrc.github.io/benchmark-spark/) with:
- Peak memory comparisons
- KV cache allocation charts
- Throughput performance graphs

## What This Proves

After 60 runs across 3 models, the findings are clear:

1. **Container overhead is real**: 20-30 GB consistently
2. **KV cache reduction is significant**: 1.7-2.7x less in containers
3. **Performance is identical**: No speed penalty for native execution
4. **Pattern is consistent**: Happens across all model sizes
5. **Root cause confirmed**: Unified memory + cgroup double-counting

This isn't a fluke. This isn't a configuration error. This is a fundamental architectural mismatch.

Next up: What we learned, practical recommendations, and what we're investigating next (spoiler: that KV cache scaling is fascinating).

---

**Previous**: [← Part 3: The Unified Memory Revelation](../03-unified-memory-revelation)  
**Next**: [Part 5: What We Learned (And What's Next) →](../05-what-we-learned)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)  
**Interactive Charts**: [Results Dashboard](https://brandonrc.github.io/benchmark-spark/)
