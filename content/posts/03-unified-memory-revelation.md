---
title: "The Unified Memory Revelation: Why Docker Double-Counts"
date: 2025-11-08T12:00:00-00:00
draft: false
tags: ["dgx-spark", "grace-blackwell", "unified-memory", "cgroups", "docker"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "The 'aha!' moment: Docker's cgroups were designed for discrete GPUs, but Grace Blackwell has unified memory. The result? Docker counts GPU memory twice, creating 20-30GB of phantom overhead."
---

## The Question That Started It All

After getting both Docker and native environments working, I could finally run proper benchmarks. But I kept asking myself:

**"Where is the 26GB going?"**

It wasn't CPU overhead - containers don't add 26GB of process memory.  
It wasn't the Docker daemon - that's tiny.  
It wasn't duplicate libraries - bind mounts prevent that.

So... where?

## Traditional GPU Systems (The Old Way)

Let's start with how most GPU systems work. Take an NVIDIA H100 or A100:

```
┌────────────────┐      ┌────────────────┐
│   CPU (Host)   │      │   GPU (Device) │
│                │      │                │
│  DDR RAM       │◄────►│   HBM (VRAM)   │
│  64-512 GB     │ PCIe │   40-80 GB     │
└────────────────┘      └────────────────┘
```

**Key points:**
- CPU has its own RAM (DDR)
- GPU has its own VRAM (HBM)
- Data moves between them over PCIe bus
- They're **separate memory spaces**

When Docker runs on these systems:
- Docker's cgroups manage **CPU RAM only**
- GPU VRAM is outside Docker's control
- nvidia-docker just passes through GPU access
- **No double-counting** because they're separate

## Grace Blackwell: The Game Changer

Now look at Grace Blackwell (our DGX Spark):

```
┌─────────────────────────────────────┐
│      Grace Blackwell System            │
│                                     │
│  ┌────────────────────────────────┐ │
│  │   UNIFIED MEMORY (128 GB)      │ │
│  │   ┌─────────┐    ┌──────────┐  │ │
│  │   │ ARM CPU │◄──►│ GB10 GPU │  │ │
│  │   └─────────┘    └──────────┘  │ │
│  │   Coherent, Cache-Coherent     │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**Revolutionary differences:**
- **One memory pool** for both CPU and GPU
- No PCIe transfers - both access the same RAM
- Coherent at the hardware level
- The CPU "sees" GPU allocations and vice versa

This is elegant! No more copying between CPU and GPU. It's all one big shared memory space.

## The Docker cgroup Problem

Here's where things go sideways.

Docker uses **Linux cgroups** (control groups) to isolate and track container resources:

```bash
# Docker creates a memory cgroup for each container
/sys/fs/cgroup/docker/<container_id>/memory.max
/sys/fs/cgroup/docker/<container_id>/memory.current
```

On a traditional GPU system, cgroups see:
- CPU RAM: Managed by cgroup ✓
- GPU VRAM: Outside cgroup (invisible) ✓

On Grace Blackwell unified memory, cgroups see:
- The entire 128GB pool as "system RAM" ✗

## The Double-Counting

Here's what happens when you run a model in a Docker container on Grace Blackwell:

1. **Model loads** into unified memory (let's say 70GB)
2. **CUDA driver** records this as GPU allocation
3. **Docker's cgroup** sees 70GB of "container RAM" used
4. **TensorRT-LLM** tries to allocate KV cache
5. **Docker thinks**: "Container is using 70GB already, only X GB left"
6. **Reality**: That 70GB is THE SAME MEMORY, just counted twice!

Result: Docker reserves extra headroom because it thinks GPU memory is separate "container RAM", even though it's not.

## The Evidence

Let's look at what we saw:

### Native (Chroot)
```
Total unified memory: 119.64 GB
Model peak usage:     78.31 GB
Leaves:               41.33 GB
KV cache allocated:   37.26 GB (90% of available)
```

### Container (Docker)
```
Total unified memory: 119.64 GB
Model peak usage:     104.92 GB  ← What?!
Leaves:               14.72 GB
KV cache allocated:   13.31 GB (90% of available)
```

Docker's cgroup sees 104.92GB used, but the actual model only needs 78GB. The difference (26.6GB) is **phantom overhead** from Docker's memory accounting trying to "reserve" space that's already in use.

## Why Isn't This a Problem on Discrete GPUs?

On H100/A100 systems, cgroups **can't even see** GPU VRAM. It's on a separate PCIe device. So there's no double-counting:

```
Discrete GPU:
- cgroup tracks: CPU RAM only (e.g., 64GB)
- CUDA tracks: GPU VRAM separately (e.g., 80GB)
- Total: 64 + 80 = 144GB
- No overlap ✓

Unified Memory (Grace Blackwell):
- cgroup tracks: "System RAM" (128GB)
- CUDA tracks: Part of that same 128GB
- Docker's accounting: Confused!
- Creates overhead ✗
```

## The KV Cache Impact

This double-counting has a massive effect on KV cache:

```
Available for KV cache:
- Native:    37.26 GB
- Container: 13.31 GB
- Difference: 2.8x MORE in native mode
```

More KV cache means:
- Longer context windows
- Higher batch sizes
- More concurrent requests
- Better overall throughput scaling

That 26GB overhead isn't just wasted RAM - it's **stolen capacity** for serving workloads.

## Why Performance Is Still The Same

You might wonder: "If Docker has less KV cache, why is throughput identical?"

Good question! For our specific benchmark (50 requests, 128 output tokens):
- Even 13GB of KV cache was enough
- We weren't hitting the cache limit
- Throughput was compute-bound, not memory-bound

But in production with:
- Longer contexts (8k, 32k, 128k tokens)
- Higher batch sizes
- Many concurrent users

That reduced KV cache would **absolutely** become a bottleneck.

## Can Docker Be Fixed?

Maybe? Potential solutions:

1. **Update nvidia-container-toolkit** for unified memory awareness
2. **Use `--memory=unlimited`** to disable cgroup memory limits
3. **Special cgroup configuration** for Grace Blackwell
4. **Wait for Docker/kernel patches** that understand unified memory

But for now, the simplest solution: **Use native execution for large models on Grace Blackwell**.

## The Takeaway

This isn't Docker being "bad" or Grace Blackwell being "broken." It's a **mismatch between technology generations**:

- Docker's cgroups: Designed for discrete GPU era
- Grace Blackwell: Next-gen unified memory architecture
- Result: Software assumptions don't match hardware reality

And that's why you can't just blame the hardware. The entire stack matters.

In the next post, I'll show you the data: 60 comprehensive benchmark runs across 3 different models, proving this pattern holds consistently.

---

**Previous**: [← Part 2: MPI and Chroot Nightmare](../02-mpi-chroot-nightmare)  
**Next**: [Part 4: The Data - 60 Runs Don't Lie →](../04-the-data)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)
