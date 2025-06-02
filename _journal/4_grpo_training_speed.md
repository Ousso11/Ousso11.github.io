---
title: "Speeding Up Distributed Training with vLLM, Flash Attention, and Checkpoint Resuming"
collection: journal
excerpt: "Improving distributed training speed using vLLM, Flash Attention, LoRA, gradient checkpointing, and stable checkpoint recovery across multi-node systems."
---

## Issue

While scaling up model fine-tuning with **distributed training**, we encountered several performance bottlenecks and reliability issues:

- **Slow inference and response latency** with traditional model serving
- **Memory overloads** with large models in multi-GPU settings
- **Unreliable checkpoint resumption** during multi-node training runs
- **Suboptimal memory usage and throughput** with standard attention implementations

## Solution

We applied the following stack of improvements to **significantly boost speed** and **reduce memory usage**, especially in distributed settings:

### 1. Use `vLLM` with Updated `torchrun` + `transformers` 0.16+

- Switched to **vLLM** for faster inference with paged attention.
- Ensured compatibility with **Transformers v0.16** and `torchrun` for clean distributed job launches.
- Set up **table-stable model loading** to support smooth rollout.

### 2. Enable Flash Attention (v2)

- Replaced standard attention with **Flash Attention v2**, yielding up to **3× speedup** and major **VRAM savings** during training.
- Confirmed compatibility with our model (no rotary embedding or special kernels blocking Flash).

### 3. Gradient Checkpointing

- Enabled **gradient checkpointing** to reduce memory usage at the cost of minimal compute overhead.
- Applied it selectively, avoiding attention blocks where Flash is already efficient.
- Greatly improved trainable model size and batch size scalability.

### 4. Use LoRA for Efficient Fine-Tuning

- Added **LoRA adapters** for parameter-efficient tuning of large language models.
- Enabled **LoRA weight merging** post-training.
- Reduced training time and memory usage while retaining model performance.

### 5. Fix Checkpoint Resume Bug

- Patched training logic to ensure **checkpoint recovery works** after mid-run failures in distributed settings (especially with `torchrun` + deepspeed).
- Ensured optimizer and scheduler states resume correctly to avoid performance drift.

## 🚀 Outcome

With this setup:

- Training was **significantly faster and more stable**
- We ran **7B+ parameter models** across multiple nodes **without hitting OOM**
- **Checkpoint resume was robust**, making experimentation safer and faster
- Overall system throughput increased by **2–4×** depending on hardware and model config

## 💡 Takeaway

Modern LLM fine-tuning pipelines benefit immensely from combining:

- **vLLM + Flash Attention** for memory-efficient speed
- **Gradient checkpointing + LoRA** for scalable adaptation
- **Robust checkpointing logic** to avoid reruns

This stack is essential for practical, scalable LLM training on clusters.
