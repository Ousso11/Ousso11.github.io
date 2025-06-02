---
title: "Speeding Up Graph Similarity Matching with Efficient Tensor Ops"
collection: journal
share: false
related: false
excerpt: "The graph similarity algorithm for matching image-text graph pairs was too slow, particularly in the pairwise comparison step"

---

## Issue

The graph similarity algorithm for matching image-text graph pairs was **too slow**, particularly in the **pairwise comparison step**.

Each image-text pair requires computing a **similarity matrix** between the merged nodes of the image graph and the text graph. When extended to batch inference over many candidate graphs, the computational cost scales poorly: for every image graph, we must compute the similarity between **all node pairs** in the corresponding text graphs.

The initial implementation used nested for-loops or inefficient matrix operations, creating a bottleneck that made the system unusable for large-scale retrieval.

## Solution

Replaced the nested similarity computation with a **batched and vectorized tensor operation** using `torch.einsum`.

We represent node embeddings of image graphs as a 3D tensor `G_img` with shape `[B, N, D]` and text graph nodes as `G_txt` with shape `[B, M, D]`, where:

- `B` = batch size  
- `N`, `M` = number of nodes in image and text graphs  
- `D` = embedding dimension  

To compute pairwise similarities efficiently, we use:

```python
# Compute pairwise dot-product similarity between all nodes in image and text graphs
similarity_matrix = torch.einsum('bnd,bmd->bnm', G_img, G_txt)
```
