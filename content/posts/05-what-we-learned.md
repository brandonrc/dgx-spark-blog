---
title: "What I Learned (And What's Next)"
date: 2025-11-08T14:00:00-00:00
draft: false
tags: ["dgx-spark", "conclusions", "recommendations", "phase2"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "After 60 benchmarks and hours of debugging, here's what I learned about Grace Hopper, Docker, and the importance of understanding your full stack. Plus: what I'm investigating next."
---

## The Journey So Far

Let's recap this wild ride:

1. **The Problem**: YouTube reviewers blamed NVIDIA hardware for being slow
2. **The Investigation**: I found 20-30GB memory overhead in Docker containers
3. **The Environment Setup**: MPI and chroot configuration nightmare
4. **The Revelation**: Docker's cgroups double-count unified memory
5. **The Data**: 60 runs confirmed the pattern consistently

Now, what do I actually **do** with this knowledge?

## Key Finding: Don't Blame the Hardware

The most important lesson from this entire investigation:

**Hardware isn't the problem when you haven't understood the software stack.**

Those YouTube reviews that said "DGX Spark is slow" or "Grace Hopper is disappointing"? They were wrong. Not because the numbers were wrong, but because they **stopped at the numbers**.

The hardware is fine. The software assumptions are outdated.

Docker's cgroups were designed in an era of discrete GPUs where:
- CPU RAM and GPU VRAM are separate
- Memory spaces don't overlap
- No double-counting is possible

Grace Hopper introduced unified memory:
- One coherent memory pool
- Both processors access the same RAM
- Elegant... but Docker doesn't understand it yet

**The lesson**: Dig deeper. Understand the full stack. Don't just blame the hardware.

## Practical Recommendations

Based on my findings, here's what I recommend:

### For Large Models on Grace Hopper (> 10B params):

**✅ Use Native/Chroot Execution**

Why:
- 20-30 GB memory savings
- 1.7-2.7x more KV cache
- No performance penalty
- Better resource utilization

Trade-off: Less isolation, more setup complexity

### For Small Models (< 10B params):

**🤔 Docker is Acceptable**

If 30GB overhead is acceptable for your use case:
- Easier deployment and management
- Better isolation for multi-tenancy
- Standard container tooling
- Simpler CI/CD integration

### For Discrete GPU Systems (H100, A100):

**⚠️ This Finding is Grace Hopper Specific**

Traditional discrete GPU systems should NOT exhibit this pattern because:
- GPU VRAM is outside Docker's cgroups
- No double-counting possible
- Standard container best practices apply

## The KV Cache Mystery (Phase 2)

Here's what really caught my attention: **The relationship between container overhead and KV cache**.

Look at the pattern across all models:

| Model | Native Total | Container Total | Overhead | Native KV | Container KV | KV Reduction |
|-------|--------------|-----------------|----------|-----------|--------------|--------------|
| DeepSeek-7B | 70.47 GiB | 101.30 GiB | **+30.83 GiB** | 44.31 GiB | 16.57 GiB | **-27.74 GiB** |
| Qwen-72B | 70.03 GiB | 90.02 GiB | **+19.99 GiB** | 44.71 GiB | 26.72 GiB | **-17.99 GiB** |
| GPT-OSS-120B | 71.72 GiB | 93.43 GiB | **+21.71 GiB** | 43.19 GiB | 23.65 GiB | **-19.54 GiB** |

**The Real Question**: Where is that 20-30 GB container overhead going? And why does it result in lower KV cache allocation?

**Hypothesis**: Docker's cgroups are double-counting unified memory, making TensorRT-LLM think it has less available memory. The framework then conservatively allocates less KV cache to avoid OOM errors.

Notice:
- All three models use **~44 GiB KV cache in native mode** (very similar!)
- Container overhead directly correlates with KV cache reduction
- The overhead isn't going to computation - it's just... disappearing

**Phase 2 Goal**: Figure out exactly where the container overhead is going and why it prevents proper KV cache allocation.

## Phase 2: Deep Dive into Container Memory Accounting

I'm planning a comprehensive Phase 2 investigation to understand exactly where that overhead is going:

### The Plan

1. **Profile memory allocation in real-time**
   - Use `nvidia-smi dmon` during container vs native runs
   - Track CUDA memory allocation patterns
   - Monitor cgroup memory accounting vs actual GPU usage

2. **Test Docker memory configurations**
   - Different cgroup versions (v1 vs v2)
   - Various `--gpus` configurations
   - Test with `--privileged` mode
   - Try `--ipc=host` and other isolation tweaks

3. **Instrument TensorRT-LLM**
   - Add logging to see how much memory it thinks is available
   - Track KV cache allocation decisions
   - Compare memory queries between environments

4. **Compare with discrete GPUs**
   - Run same tests on H100/A100 system
   - Confirm this is Grace Hopper unified memory specific
   - Establish baseline for normal Docker behavior

### Key Questions to Answer

- **Where is the 20-30 GB going?** Is it actually allocated, or just counted differently?
- **Why does TensorRT-LLM allocate less KV cache?** What signal is it reading?
- **Can Docker be configured to handle unified memory?** Are there flags/configs we're missing?
- **Is this NVIDIA Container Toolkit specific?** Would native containerd or podman behave differently?

### Expected Outcomes

- Pinpoint the exact mechanism causing double-counting
- Determine if there's a Docker configuration fix
- Document whether this affects other unified memory systems (AMD MI300X, future Intel solutions)
- Provide concrete recommendations for Grace Hopper containerization

## Share Your Findings

If you're running Grace Hopper systems (or other unified memory architectures), I'd love to hear from you:

- Are you seeing similar patterns?
- Have you found workarounds?
- Do you have additional data to share?

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)  
**Open an issue** or **submit a PR** with your findings!

## Resources

All the code, data, and analysis are open source:

- **📊 Interactive Results**: [brandonrc.github.io/benchmark-spark](https://brandonrc.github.io/benchmark-spark/)
- **📄 Full Analysis**: [ANALYSIS.md](https://github.com/brandonrc/benchmark-spark/blob/main/ANALYSIS.md)
- **🔧 Benchmark Scripts**: [scripts/](https://github.com/brandonrc/benchmark-spark/tree/main/scripts)
- **📦 Raw Data**: [results/comprehensive/](https://github.com/brandonrc/benchmark-spark/tree/main/results/comprehensive)

## Final Thoughts

This investigation reinforced something fundamental:

**Modern AI infrastructure is a stack**:
- Hardware (Grace Hopper)
- Kernel (Linux cgroups)
- Drivers (NVIDIA, CUDA)
- Runtime (Docker, containerd)
- Software (TensorRT-LLM, PyTorch)
- Applications (Your LLM workload)

A problem at any layer can look like a problem at any other layer. 

When something seems slow or inefficient, resist the urge to blame the most visible component (usually the hardware or the framework). Instead:

1. **Measure everything** - Get real data
2. **Isolate variables** - Test different configurations
3. **Understand the stack** - Know what each layer does
4. **Share findings** - Help the community

The YouTubers who blamed NVIDIA weren't doing engineering. They were doing performance theater.

I did engineering. And I found the real answer.

---

## What's Your Experience?

Have you encountered similar issues? Different findings? Better solutions?

- Comment on [GitHub Discussions](https://github.com/brandonrc/benchmark-spark/discussions)
- Share your data
- Help me build Phase 2

Together, you and I can make GPU computing better for everyone - by actually understanding it instead of just pointing fingers.

---

**Previous**: [← Part 4: The Data - 60 Runs Don't Lie](../04-the-data)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)  
**Phase 2 Tracking**: [GitHub Issues](https://github.com/brandonrc/benchmark-spark/issues)

---

**Thanks for following along!** 🚀
