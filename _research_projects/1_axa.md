---
title: "Unified Graph-Based Matching: Text-Image Cross-Modal Retrieval using Scene Graph Alignment for Remote Sensing Applications"
excerpt: "We introduce a unified framework for cross-modal retrieval in remote sensing, aligning scene graphs from satellite imagery and text descriptions. Using the STAR dataset, our method encodes graph structures from both modalities and aligns them via contrastive learning. We evaluate multiple similarity strategies—node, edge, global, and hybrid—and propose a benchmark protocol for retrieval. This work opens up new directions for structured vision-language understanding in geospatial domains."
collection: research_projects
share: false
related: false
---

## Problem Statement & Motivation

Scene Graph Generation (SGG) has made significant strides in natural image domains, but its adaptation and performance in complex remote sensing imagery remain underexplored. The **STAR dataset**—the first large-scale benchmark for SGG in satellite imagery—opens new directions for structured understanding of overhead scenes. However, leveraging scene graphs for **image-text retrieval** presents unique challenges due to:

- The **abstract, high-variance nature** of satellite scenes.
- The **domain gap** between visual content and textual descriptions.
- The **lack of standard benchmarks** for retrieval grounded in graph semantics.

## Our Method

To explore graph-based retrieval in the remote sensing context, we propose a **contrastive learning framework** that incorporates **scene graph embeddings** from both image and text modalities.

### 1. Scene Graph Construction

- From images: using existing annotations in the STAR dataset.
- From text: using GPT-4o-mini to extract structured scene graphs from natural language descriptions.

### 2. Graph Embedding and Similarity Retrieval

We explore multiple graph-based retrieval strategies:

- **Node similarity**: comparing object/entity types.
- **Edge similarity**: comparing relationships between objects.
- **Local matching**: combining node and edge features.
- **Global matching**: encoding full scene graphs using GNNs or graph encoders.
- **Hybrid retrieval**: fusing both local and global similarity signals.

### 3. Training

- We use a **contrastive loss** to align image and text scene graph embeddings.

## Evaluation & Benchmarking

We design a **retrieval benchmark protocol** for the STAR dataset, including:

- Standard retrieval metrics: **Recall@K**, **Mean Average Precision (mAP)**.
- Comparison against baselines such as:
  - **CLIP** (vision-language model).
  - **Graph-RISE** (graph-based image-text retrieval).
  - Standard SGG-based matching models.

## Results

Our experiments show that **graph-aware retrieval models** outperform naive baselines in capturing fine-grained scene structure. Hybrid models combining local and global matching perform best under high variance satellite scenes.

## What’s Missing / Future Work

- A **standard retrieval benchmark** for graph-based remote sensing tasks.
- **Ablation studies**: evaluating node-only, edge-only, and full-graph matching.
- Further exploration of **multilingual or multimodal grounding** in complex geospatial environments.
