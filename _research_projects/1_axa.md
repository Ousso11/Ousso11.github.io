---
title: "Unified Graph-Based Matching: Text-Image Cross-Modal Retrieval using Scene Graph Alignment for Remote Sensing Applications"
excerpt: "We introduce a unified framework for cross-modal retrieval in remote sensing, aligning scene graphs from satellite imagery and text descriptions. Using the STAR dataset, our method encodes graph structures from both modalities and aligns them via contrastive learning. We evaluate multiple similarity strategies—node, edge, global, and hybrid—and propose a benchmark protocol for retrieval. This work opens up new directions for structured vision-language understanding in geospatial domains."
collection: research_projects
share: false
related: false
---

## Problem Statement & Motivation

Scene Graph Generation (SGG) has made significant strides in natural image domains, but its adaptation and performance in complex remote sensing imagery remain underexplored. The **STAR dataset**—the first large-scale benchmark for SGG in satellite imagery—opens new directions for structured understanding of overhead scenes. However, leveraging scene graphs for **image-text retrieval** presents unique challenges due to:

- **Compositional complexity**: Satellite imagery often contains numerous interrelated objects (e.g., roads, buildings, vehicles) whose spatial and functional relationships are essential for meaningful interpretation but difficult to encode into a single embedding vector.
- **Cross-modal alignment difficulties**: Accurately aligning graph-structured visual representations with free-form textual queries requires deep modeling of both relational semantics and contextual cues.
- **Scalability and generalization**: Scene graphs in remote sensing must scale to large, diverse geographic regions and generalize across vastly different environments and timeframes.
- **Ambiguity and sparsity in annotations**: Satellite images often lack dense, fine-grained annotations, making supervised learning of graph-based representations and their grounding in language more difficult.

## Our Method

To explore graph-based retrieval in the remote sensing context, we propose a **contrastive learning framework** that incorporates **scene graph embeddings** from both image and text modalities.

### 1. Scene Graph Construction

- From images: using existing annotations in the STAR dataset.
- From text: using GPT-4o-mini to extract structured scene graphs from natural language descriptions.

### 2. Graph Embedding and Similarity Retrieval

We explore several graph-based retrieval strategies:

- **Node similarity**: comparing object/entity types.
- **Edge similarity**: comparing relationships between objects.
- **Local matching**: combining node and edge features.
- **Global matching**: encoding full scene graphs with GNNs or graph transformers.
- **Hybrid retrieval**: fusing local and global similarity signals.

For each image, we extract a scene graph and encode it using a **graph transformer** with a designated **query vertex**. We apply the same process to text-generated graphs, ensuring structural consistency.

To improve alignment, we use **cross-attention mechanisms**, optionally applying the **Hungarian algorithm** for optimal node/edge matching. This supports multiple retrieval modes:

- **Object-centric**: retrieve based on key entities.
- **Relation-aware**: focus on spatial/semantic relationships.
- **Scene-type**: match overall graph structure (e.g., "urban area").
- **Graph-template**: retrieve scenes matching a user-defined graph.

---

### 3. Training

We train with a mix of:

- **Contrastive loss**: aligns matching image-text graph pairs.
- **Triplet loss**: enforces margin-based separation from incorrect matches.

This dual-loss setup improves both global alignment and fine-grained discrimination, supporting robust retrieval across diverse satellite scenes.

## Evaluation & Benchmarking

We design a **retrieval benchmark protocol** on the STAR dataset, featuring:

- Standard metrics: **Recall@K** and **Mean Average Precision (mAP)**.
- Structured evaluation across various scene types and relation complexities.

We compare our graph-based retrieval pipeline against a wide range of baselines from multimodal and remote sensing literature, including:

- **Vision-language models**: RemoteCLIP, GeoRSCLIP, RS-M-CLIP, SkyScript, AIR.
- **Prompt and image retrieval methods**: PIR-ITR, PIR-CLIP, SWPE.
- **Graph and attention-based approaches**: AMFMN, GaLR, SWAN, HVSA, DOVE, KAMCL, PE-RSITR, MSA.
- **Knowledge-augmented and embedding models**: EBAKER, iEBAKER-Split, LRSCLIP.

> *Note: The benchmarking pipeline is still under development, and integration of these baselines is in progress.*

This benchmark aims to standardize scene-graph-driven image-text retrieval evaluation in satellite imagery, encouraging reproducibility and cross-method comparison.
