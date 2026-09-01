---
title: "The GPU That Said Yes and Ran on CPU"
collection: journal
permalink: /journal/onnx-silent-cpu-fallback/
excerpt: "onnxruntime reported CUDAExecutionProvider as available while every real session ran on CPU, because libcudnn.so.9 could not be loaded. Nothing raised. The chunker just got slow, and stayed slow in production for weeks."
---

## Issue

A production inference service was slow. Not broken — slow. The image had a GPU, `nvidia-smi` was happy, and `onnxruntime.get_available_providers()` listed `CUDAExecutionProvider`. Everything said the accelerator was in use.

Reproduced on a GPU host: **the session was running on CPU.** `libcudnn.so.9` could not be loaded, so the CUDA execution provider silently fell back to the CPU provider. The chunker on top of it does not raise when this happens. It just runs on CPU.

This is the worst class of bug. There is no error, no log line, no failed health check. There is only a number that is worse than it should be, and no reason to suspect the reason.

## Root Cause

Three separate things had to be true, and all three were.

**1. The loader path was incomplete.** The runtime `LD_LIBRARY_PATH` carried only `nvidia/cuda_runtime/lib`. cuDNN and cuBLAS live in *sibling* wheel directories. PyTorch gets away with this because it has RPATH entries pointing into them — onnxruntime does not. It resolves purely off the loader path, so its CUDA EP could never load, on any machine, ever.

**2. The build-time guard was asking the wrong question.** The check asserted on `get_available_providers()`. That function reports **what was compiled in**, not what can actually load. It passes cheerfully on a machine where the chunker demonstrably runs on CPU. The guard had been green for the entire life of the bug.

**3. The image-strip pass could delete the libraries and still report success.** Nothing verified that cuDNN and cuBLAS survived the layer that shrinks the image.

## Solution

**Glob every `nvidia/*/lib` directory onto the loader path at start.** Not an enumerated list — a glob, so a wheel-layout change in a future CUDA release cannot quietly break it again.

**Make the build check `dlopen` the CUDA EP itself**, which is the thing that actually pulls in cuDNN. No GPU is required for this, so it works inside a build.

That last part needed a second pass. `dlopen` can *never* succeed during a build: `libcuda.so.1` is the driver, and the container toolkit only injects it at run time. So the check was rewritten again to resolve the EP's dependencies with **`ldd`**, failing on anything unresolved that we actually ship — with the driver as the single permitted absence. It still catches the real bug (a missing cuDNN, cuBLAS or cuFFT) without demanding a GPU at build time.

**And the strip pass now fails outright** if cuDNN or cuBLAS are gone by the time it finishes, rather than producing an image that looks fine and isn't.

## 💡 Takeaway

- **"Available" and "loadable" are different questions.** Any capability check that reports compile-time configuration will pass on a broken machine. Test the thing you actually depend on.
- **Silent fallback to a slower path is a design smell.** A library that degrades from GPU to CPU without raising has converted a crash into a permanent, invisible performance tax.
- **When a fix must hold across future dependency versions, prefer a glob to a list.** The enumerated path was correct when it was written and wrong a release later.
