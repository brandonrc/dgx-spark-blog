---
title: "What We Learned (And What's Next)"
date: 2025-11-08T14:00:00-00:00
draft: false
tags: ["dgx-spark", "conclusions", "recommendations", "phase2"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "After 60 benchmarks and hours of debugging, here's what we learned about Grace Hopper, Docker, and the importance of understanding your full stack. Plus: what we're investigating next."
---

## The Journey So Far

Let's recap this wild ride:

1. **The Problem**: YouTube reviewers blamed NVIDIA hardware for being slow
2. **The Investigation**: We found 20-30GB memory overhead in Docker containers
3. **The Environment Setup**: MPI and chroot configuration nightmare
4. **The Revelation**: Docker's cgroups double-count unified memory
5. **The Data**: 60 runs confirmed the pattern consistently

Now, what do we actually **do** with this knowledge?

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

Based on our findings, here's what we recommend:

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

## The KV Cache Deep Dive (Phase 2)

Here's what really caught our attention: **The KV cache scaling**.

Look at these numbers again:

| Model | Size | Native KV Cache |
|-------|------|----------------|
| DeepSeek | 7B | 44.31 GiB |
| Qwen | 72B | **44.71 GiB** |
| GPT-OSS | 120B | 43.19 GiB |

Wait... the **72B model has MORE KV cache** than the 7B model? And almost as much as the 120B model?

This suggests:
- Qwen-72B has superior memory optimization
- Model architecture matters more than parameter count
- There's room to optimize other models similarly

**Phase 2 Goal**: Understand KV cache scaling and memory efficiency across different model architectures.

## Phase 2: The Factory Reset Experiment

We're planning a comprehensive Phase 2 investigation:

### The Plan

1. **Factory reset DGX Spark** - Start completely fresh
2. **Install bare metal software** - Match container versions exactly
3. **Create 1:1 configurations** - Container vs native, identical software
4. **Run extended benchmarks** - More models, longer contexts, varied batch sizes
5. **Investigate KV cache scaling** - Why does Qwen-72B use memory so efficiently?

### Goals

- Validate our findings with even cleaner test setup
- Eliminate any remaining configuration variables
- Deep dive into KV cache allocation strategies
- Understand memory efficiency across model families
- Test on other hardware (discrete GPU comparison)

### Open Questions

- Can we optimize Docker for unified memory? (cgroup v2, special configs)
- Do other unified memory systems (AMD MI300) show the same pattern?
- How does KV cache scale with context length (8k, 32k, 128k tokens)?
- What model architectures are most memory-efficient?

## Share Your Findings

If you're running Grace Hopper systems (or other unified memory architectures), we'd love to hear from you:

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

We did engineering. And we found the real answer.

---

## What's Your Experience?

Have you encountered similar issues? Different findings? Better solutions?

- Comment on [GitHub Discussions](https://github.com/brandonrc/benchmark-spark/discussions)
- Share your data
- Help us build Phase 2

Together, we can make GPU computing better for everyone - by actually understanding it instead of just pointing fingers.

---

**Previous**: [← Part 4: The Data - 60 Runs Don't Lie](../04-the-data)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)  
**Phase 2 Tracking**: [GitHub Issues](https://github.com/brandonrc/benchmark-spark/issues)

---

**Thanks for following along!** 🚀
