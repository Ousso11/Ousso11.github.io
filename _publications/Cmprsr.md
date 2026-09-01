---
title: "Cmprsr: Abstractive Token-Level Question-Agnostic Prompt Compressor"
collection: publications
permalink: /publications/Cmprsr/
excerpt: "Co-author. A prompt compressor that does not need to know the question in advance, so one compressed prompt can serve whatever gets asked of it afterwards. Trained with supervised fine-tuning and preference optimisation, it beats LLMLingua-2 on maths reasoning, summarisation and multiple-choice QA at matched token budgets."
date: 2025-11-15
venue: 'arXiv preprint — under review at ACL 2026'
paperurl: 'https://arxiv.org/abs/2511.12281'
share: false
related: false
---

## Summary

Most prompt compressors are **query-specific**: they need the question up front in order to decide what to keep. That is fine for a single-turn QA pipeline and awkward everywhere else — caches, shared context, and any setting where the same context serves many different questions.

**Cmprsr** is **question-agnostic**. It compresses at the token level without seeing the query, so one compressed prompt can serve whatever gets asked of it afterwards. It is trained with supervised fine-tuning followed by preference optimisation.

## Results

At **matched token budgets**, Cmprsr beats **LLMLingua-2** on:

- mathematical reasoning
- summarisation
- multiple-choice question answering

## Why It Mattered Afterwards

This line of work started at **EPFL's dlab** as a benchmarking programme measuring LLMs as compressors, and became the research basis for a production compression API — the one now shipped by [Compresr](https://compresr.ai).

## Publication Details

- **Preprint**: [arXiv:2511.12281](https://arxiv.org/abs/2511.12281)
- **Status**: under review at **ACL 2026**
- **My role**: co-author
