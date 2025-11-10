---
title: "The Mystery: Don't Just Blame the Hardware"
date: 2025-11-08T10:00:00-00:00
draft: false
tags: ["dgx-spark", "performance", "debugging", "docker", "grace-blackwell"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "YouTube tech reviewers were quick to blame NVIDIA's new DGX Spark for being slow. But was it really the hardware's fault, or was I missing something deeper in the software stack?"
---

## The YouTube Problem

If you search for "DGX Spark performance" on YouTube, you'll find plenty of videos with clickbait titles like "NVIDIA's $X Machine is a DISAPPOINTMENT" or "Grace Blackwell: Overhyped and Underdelivering."

And that really bothers me.

Not because I'm an NVIDIA fanboy (I'm not), but because **none of these reviewers provided a technical explanation** of why performance wasn't meeting expectations. They just pointed at benchmark numbers, said "slow," and moved on. No investigation into kernel settings, driver versions, container configurations, or software stack optimization. Just... blame the hardware.

That's not how engineering works.

## My Setup: The New Kid on the Block

I got my hands on an NVIDIA DGX Spark - a genuinely interesting piece of hardware:

- **CPU**: ARM Cortex (X925 + A725), 20 cores total
- **GPU**: NVIDIA GB10 (Blackwell architecture, Grace Blackwell design)
- **Memory**: 128GB **unified** (CPU and GPU share the same RAM pool)
- **Architecture**: aarch64 (ARM64)

The unified memory part is key. Unlike traditional systems where you have separate CPU RAM and GPU VRAM that copy data back and forth over PCIe, Grace Blackwell has one coherent memory space. Both processors can access the same 128GB directly.

Pretty elegant, right?

## Prior Art: Docker GPU Passthrough Overhead

Before diving into our own investigation, I found this paper: [Benchmarking GPU Passthrough Performance on Docker for AI Cloud System](https://www.researchgate.net/publication/396583672_Benchmarking_GPU_Passthrough_Performance_on_Docker_for_AI_Cloud_System).

The authors tested Docker GPU passthrough on consumer hardware (RTX 3060) and found:
- Native execution: Faster (1.52s avg)
- Docker containers: Slower (2.55s avg), but higher GPU utilization

**This validated that Docker overhead exists.** But that study used consumer GPUs with simple matrix multiplication. I wanted to understand:

1. Does this apply to enterprise DGX hardware?
2. What about production LLM workloads, not just matmul?
3. **WHY** does this overhead exist in my specific case?

## The Test

I ran TensorRT-LLM inference benchmarks in two environments:

**Docker Container** (NVIDIA's official image):
```bash
docker run --rm --gpus all --ipc=host --shm-size=60g \
    nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev \
    python benchmarks/trtllm_benchmark.py
```

**Native (Chroot)**: Same code, same libraries, but running directly on the host using chroot to access the container's filesystem.

## The Shocking Result

| Environment | Peak Memory | KV Cache Available |
|-------------|-------------|-------------------|
| Docker Container | 104.92 GiB | 13.31 GiB |
| Native (Chroot) | 78.31 GiB | 37.26 GiB |

Wait... what?

The container was using **26.6 GB MORE memory** and had **less than half the KV cache** available for the model!

## But... Performance Was Identical

Here's what made this even more confusing:

```
Native:    120.44 tokens/sec
Container: 121.60 tokens/sec
```

Same throughput. Same latency. The container wasn't *slower*, it was just using way more memory for no apparent reason.

This wasn't a "slow hardware" problem. **This was something deeper.**

## Why This Matters

You might think "who cares about 26GB if performance is the same?" But:

1. **KV Cache**: That 26GB could enable longer contexts or higher batch sizes
2. **Multi-tenancy**: Less memory per container = fewer workloads per box
3. **Cost**: In cloud deployments, you pay for memory
4. **Principle**: When hardware isn't the problem, blaming hardware is lazy

## The Investigation Begins

Rather than accept this as "Docker is bloated" or "Grace Blackwell is broken," I decided to dig in:

1. Run comprehensive benchmarks (60 runs across 3 different model sizes)
2. Test multiple LLMs: 7B, 72B, and 120B parameter models
3. Understand what Grace Blackwell's unified memory architecture really means
4. Figure out exactly **where** that 26GB is going

The journey involved:
- MPI library path hell
- Chroot filesystem nightmares
- A revelation about Linux cgroups and unified memory
- Some genuinely surprising discoveries about KV cache scaling

**Spoiler**: I found the root cause. And it's not what you think.

In the next post, I'll walk through the pain of setting up a proper test environment. Because sometimes the hardest part of debugging isn't finding the bug - it's just getting to a clean test in the first place.

---

**Next**: [Part 2: Down the Rabbit Hole - MPI and Chroot →](../02-mpi-chroot-nightmare)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)

**References**:
- [Benchmarking GPU Passthrough Performance on Docker for AI Cloud System](https://www.researchgate.net/publication/396583672_Benchmarking_GPU_Passthrough_Performance_on_Docker_for_AI_Cloud_System)
