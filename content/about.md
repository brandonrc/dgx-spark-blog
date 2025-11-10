---
title: "About This Investigation"
date: 2025-11-08
draft: false
showToc: false
---

## Who & Why

I'm Brandon Geraci, and I was tired of seeing YouTube tech reviewers blame NVIDIA's DGX Spark hardware for being "slow" or "disappointing" without providing any technical analysis.

So I decided to actually investigate what was going on.

## What We Found

After 60 comprehensive benchmarks across 3 different LLM models, we discovered:

- **Docker containers use 20-30 GB more memory** than native execution on Grace Blackwell
- **KV cache is reduced by 40-63%** in containers
- **Performance is identical** - same throughput, no speed penalty
- **Root cause**: Docker's cgroups double-count unified memory

This isn't a hardware problem. It's a software stack mismatch.

## The Project

**GitHub Repository**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)

All code, data, and analysis are open source. We ran:
- 60 total benchmark runs
- 3 models: DeepSeek-7B, Qwen-72B, GPT-OSS-120B
- 2 environments: Native (chroot) vs Container (Docker)
- 10 iterations per configuration
- ~14 hours of comprehensive testing

## Phase 2: What's Next

We're planning a follow-up investigation:
- Factory reset the DGX Spark
- Create 1:1 bare metal vs container configs
- Deep dive into KV cache scaling
- Test on discrete GPU systems for comparison

## Get Involved

If you're running Grace Blackwell or other unified memory systems:
- Share your findings
- Open issues on GitHub
- Contribute data or analysis

Let's understand this together - with engineering, not speculation.

---

**Contact**: [GitHub](https://github.com/brandonrc)  
**Project**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)  
**Interactive Results**: [Results Dashboard](https://brandonrc.github.io/benchmark-spark/)
