---
title: "Research Assistant — Data Science Lab (dlab), EPFL"
collection: work
permalink: /work/dlab/
excerpt: "A year of NLP research with Prof. Robert West on making language models behave under tight context budgets. First-authored GRAD (EMNLP 2025 Findings), where an LLM is trained with GRPO to generate few-shot demonstrations instead of retrieving them, and designed the prompt-compression benchmark programme that became the basis for an ACL submission and, later, a product."
location: Lausanne, Switzerland
share: false
related: false
---

**Sep 2024 – Sep 2025** · [dlab.epfl.ch](https://dlab.epfl.ch) · Prof. Robert West

## 🧠 The Question

Both projects below come from the same place: **language models are given far more context than they can use well, and context is the thing you pay for.** One project attacks that from the demonstration side, the other from the compression side.

## 📄 GRAD — generating demonstrations instead of retrieving them

*First author (equal contribution) — [Findings of EMNLP 2025](/publications/GRAD/), presented in Suzhou.*

Few-shot prompting usually means retrieving examples from a database you have to build, index and maintain. In **GRAD** we trained an LLM with **GRPO** to *write* an example for each specific question instead, which removes the database entirely.

**What I ran:**

- **Reward design** — a multi-objective, truncation-aware reward combining answer log-probability, accuracy and a demonstration count, with an explicit cap so the model cannot reward-hack by padding.
- **PPO vs GRPO** — compared both; kept GRPO for training stability.
- **LoRA** to fit training inside the memory available.
- **GRADi**, an SFT warm-started variant of the same recipe.

**Result:** beat RAG baselines on **GSM8K, MATH and ARC** across Qwen2.5-3B/7B/14B and Llama-3.2-3B — with *shorter* demonstrations and *shorter* answers. Trained only on maths, it generalised out of distribution to physics, chemistry and computer science. Code and models are public.

## 🗜️ LLMs as compressors — the benchmark programme

I designed the benchmark programme measuring LLMs as prompt compressors against **LLMLingua-2**, across mathematical reasoning, summarisation and multiple-choice QA at matched token budgets.

That programme became the evaluation backbone of **[Cmprsr](/publications/Cmprsr/)** (arXiv:2511.12281, under review at ACL 2026) — a **question-agnostic** token-level compressor, meaning one compressed prompt can serve whatever gets asked of it afterwards — and, a year later, the research basis for a production compression API at [Compresr](/work/compresr/).

## 🛠️ Stack

PyTorch • HuggingFace Transformers • TRL • PEFT / LoRA • vLLM • Weights & Biases • SLURM / Run:ai
