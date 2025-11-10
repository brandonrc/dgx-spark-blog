---
title: "Down the Rabbit Hole: The MPI and Chroot Nightmare"
date: 2025-11-08T11:00:00-00:00
draft: false
tags: ["dgx-spark", "mpi", "chroot", "debugging"]
categories: ["Investigation"]
author: "Brandon Geraci"
showToc: true
TocOpen: false
description: "Sometimes the hardest part of debugging isn't finding the bug - it's just getting to a clean test environment. Here's how we spent hours fighting MPI libraries and chroot configurations before we could even start benchmarking."
---

## The Simple Plan (That Wasn't Simple)

After seeing the mysterious 26GB memory overhead in Docker, my plan was straightforward:

1. Extract the container's filesystem
2. Run the same Python scripts natively
3. Compare the results
4. Done!

Ha. Hahahaha. **No.**

## Attempt 1: Just Run It

My first thought: "Let's just run the Docker scripts on my system. How hard can it be?"

```bash
python /home/khan/benchmark-spark/benchmarks/trtllm_benchmark.py \
    --model deepseek-ai/DeepSeek-R1-Distill-Qwen-7B
```

What I got: **Runtime nightmare**.

```
ModuleNotFoundError: No module named 'tensorrt_llm'
ImportError: libmpi.so.40: cannot open shared object file
CUDA Error: no CUDA-capable device is detected
```

Of course. The container has TensorRT-LLM, specific CUDA versions, MPI libraries, and about a million other dependencies that my host system doesn't have (or has different versions of).

Okay, fine. Let's extract the container and use chroot.

## Extracting the Container

Docker containers are just fancy tarballs with layers. To extract:

```bash
# Export the container filesystem
docker create --name temp nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev
docker export temp > container.tar
docker rm temp

# Extract to a directory
mkdir -p /home/khan/container-rootfs
sudo tar -xf container.tar -C /home/khan/container-rootfs
```

Now I have the entire container filesystem at `/home/khan/container-rootfs/`. Cool!

## The MPI Library Hunt Begins

Time to run something:

```bash
sudo chroot /home/khan/container-rootfs python3 /workspace/benchmarks/trtllm_benchmark.py
```

New error:

```
[TRT-LLM] [I] MPI size: 1, rank: 0
MPI Error: undefined symbol: ucs_config_doc_nop
```

What? Let me check which MPI it's using:

```bash
which mpirun
# /usr/bin/mpirun  ← System MPI!
```

Ah. The chroot is finding the **system's MPI** (installed at `/usr/bin/mpirun`) instead of the **container's MPI** (at `/opt/hpcx/ompi/bin/mpirun` inside the rootfs).

## The PATH Dance

TensorRT-LLM uses HPC-X OpenMPI from NVIDIA. The container has it at `/opt/hpcx/ompi/`. But when I chroot, the PATH still points to system binaries first.

Solution: Explicitly set PATH to prioritize container binaries.

```bash
export PATH="/opt/hpcx/ompi/bin:$PATH"
```

Run again... new error:

```
mpirun: error while loading shared libraries: libucs.so.0: cannot open shared object file
```

The MPI binary is now correct, but it can't find its libraries!

## The LD_LIBRARY_PATH Saga

The container's MPI needs libraries from `/opt/hpcx/ompi/lib/`, but `LD_LIBRARY_PATH` doesn't include it.

```bash
export LD_LIBRARY_PATH="/opt/hpcx/ompi/lib:$LD_LIBRARY_PATH"
```

Run again... **DIFFERENT** error:

```
symbol lookup error: /opt/hpcx/ompi/lib/libucc.so.1: undefined symbol: ucp_worker_progress
```

Wait, what? Now it's finding the HPC-X libraries but they're conflicting with system libraries!

## The Real Problem

Here's what was happening:

1. The **system** has OpenMPI installed (`libmpi.so`)
2. The **container** has HPC-X OpenMPI (`/opt/hpcx/ompi/lib/libmpi.so`)
3. TensorRT-LLM needs the HPC-X version
4. But the dynamic linker was mixing system and container libraries

The fix: **Put container libraries at the FRONT** of `LD_LIBRARY_PATH`:

```bash
export LD_LIBRARY_PATH="/opt/hpcx/ompi/lib:/usr/local/lib/python3.12/dist-packages/tensorrt_libs:$LD_LIBRARY_PATH"
```

Finally, MPI works!

## The CUDA Version Surprise

Got MPI working. Started a benchmark. New error:

```
RuntimeError: Triton only supports CUDA 10.0 or higher, but got CUDA version: 13.0
```

Wait, CUDA 13.0? We're using CUDA 12.9!

Turns out: The system symlink `/usr/local/cuda` pointed to CUDA 13.0. The container needs CUDA 12.9.

Fix:

```bash
export CUDA_HOME="/usr/local/cuda-12.9"
export PATH="/usr/local/cuda-12.9/bin:$PATH"
```

## The Full Chroot Wrapper Script

After all this pain, I created a proper chroot wrapper that handles everything:

```bash
#!/bin/bash
SCRIPT_DIR="/home/khan/container-rootfs"

# Mount necessary filesystems
sudo mount -t proc /proc "${SCRIPT_DIR}/proc"
sudo mount --rbind /sys "${SCRIPT_DIR}/sys"
sudo mount --rbind /dev "${SCRIPT_DIR}/dev"
sudo mount --bind /home "${SCRIPT_DIR}/home"
sudo mount --bind /data "${SCRIPT_DIR}/data"

# DNS resolution
sudo cp /etc/resolv.conf "${SCRIPT_DIR}/etc/resolv.conf"

# Run in chroot with proper environment
sudo chroot "${SCRIPT_DIR}" /usr/bin/env -i \
    HOME=/root \
    PATH="/opt/hpcx/ompi/bin:/usr/local/cuda-12.9/bin:/usr/local/bin:/usr/bin:/bin" \
    LD_LIBRARY_PATH="/opt/hpcx/ompi/lib:/usr/local/lib/python3.12/dist-packages/tensorrt_libs:/usr/local/cuda-12.9/lib64" \
    CUDA_HOME="/usr/local/cuda-12.9" \
    PYTHONPATH="/usr/local/lib/python3.12/dist-packages" \
    "$@"

# Cleanup: unmount everything
sudo umount "${SCRIPT_DIR}/data"
sudo umount "${SCRIPT_DIR}/home"
sudo umount "${SCRIPT_DIR}/dev"
sudo umount "${SCRIPT_DIR}/sys"
sudo umount "${SCRIPT_DIR}/proc"
```

Now I can run:

```bash
./run_in_rootfs.sh python3 /home/khan/benchmark-spark/benchmarks/trtllm_benchmark.py
```

And it **actually works**.

## Lessons Learned

1. **Library paths matter**: System vs container libraries will bite you
2. **Environment is everything**: PATH, LD_LIBRARY_PATH, CUDA_HOME all critical
3. **MPI is picky**: HPC-X OpenMPI isn't interchangeable with system OpenMPI
4. **Filesystem mounts**: Need /proc, /sys, /dev, /home, and /data bind mounts
5. **DNS matters**: Even forgot `/etc/resolv.conf` initially!

## The Hard Part Is Over... Right?

With a working chroot environment, I could finally start benchmarking. But getting here took **hours** of debugging library paths and runtime errors.

Sometimes I wonder if this is why people just accept Docker's overhead - at least it works out of the box!

But now that we have both Docker and native environments working, we can actually compare them fairly. And that's where things get interesting...

In the next post: What Grace Hopper's unified memory architecture actually means, and why Docker's cgroups don't understand it.

---

**Previous**: [← Part 1: The Mystery](../01-the-mystery)  
**Next**: [Part 3: The Unified Memory Revelation →](../03-unified-memory-revelation)

**GitHub Repo**: [benchmark-spark](https://github.com/brandonrc/benchmark-spark)
