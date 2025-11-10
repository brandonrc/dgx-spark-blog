# DGX Spark Deep Dive Blog

A technical blog documenting the investigation into Docker container memory overhead on NVIDIA DGX Spark with Grace Hopper architecture.

## Live Site

**Blog**: https://brandonrc.github.io/dgx-spark-blog/

## About

This 5-part blog series tells the story of debugging a mysterious 20-30GB memory overhead when running LLM inference in Docker containers on Grace Hopper's unified memory architecture.

### Posts

1. **The Mystery** - YouTubers blamed NVIDIA hardware without technical analysis. We dug deeper.
2. **MPI and Chroot Nightmare** - The pain of setting up proper test environments with library path conflicts
3. **The Unified Memory Revelation** - Why Docker's cgroups double-count unified memory
4. **The Data: 60 Runs Don't Lie** - Comprehensive benchmark results across 3 models
5. **What We Learned (And What's Next)** - Conclusions, recommendations, and Phase 2 preview

## Key Findings

- **Docker containers use 20-30 GB more memory** than native execution on Grace Hopper
- **KV cache is reduced by 40-63%** in containers
- **Performance is identical** - same throughput, no speed penalty
- **Root cause**: Docker's cgroups double-count unified memory

## Related Repositories

- **Benchmark Code & Data**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)
- **Interactive Results**: [brandonrc.github.io/benchmark-spark](https://brandonrc.github.io/benchmark-spark/)

## Building Locally

```bash
# Install Hugo (requires 0.146.0+)
wget https://github.com/gohugoio/hugo/releases/download/v0.146.1/hugo_extended_0.146.1_linux-amd64.tar.gz
tar -xzf hugo_extended_0.146.1_linux-amd64.tar.gz
sudo mv hugo /usr/local/bin/

# Clone and build
git clone --recurse-submodules https://github.com/brandonrc/dgx-spark-blog.git
cd dgx-spark-blog
hugo server
```

Visit http://localhost:1313

## Tech Stack

- **Static Site Generator**: Hugo
- **Theme**: PaperMod
- **Deployment**: GitHub Pages (docs/ folder)

## License

Content: CC BY 4.0
Code: MIT

## Author

Brandon Geraci
GitHub: [@brandonrc](https://github.com/brandonrc)
