---
title: "AI Research Engineer — AXA Group Operations"
collection: work_experience
excerpt: "I led applied research and prototyping efforts in multimodal AI, focusing on cross-modal representation learning, graph-based embeddings, and neural search systems. I developed scalable pipelines to generate scene graphs from satellite imagery and knowledge graphs from textual data, and to align their graph embeddings in a shared representation space."
location: Lausanne, Switzerland
share: false
related: false
---

## Summary


My main project was building a **cross-modal retrieval system** that uses **scene graph generation for satellite imagery** and **knowledge graph construction from natural language**. The resulting graphs were embedded and aligned in a shared latent space using **contrastive learning** for efficient retrieval.


## Core Project

**Cross-Modal Graph-Based Retrieval System**  
- Generated scene graphs from satellite imagery using **MMDetection** and **OpenMMLab tools**.  
- Parsed textual descriptions into structured knowledge graphs using **LangChain** pipelines.  
- Aligned visual and textual graphs via **graph transformer-based encoders** trained with **contrastive loss**.  
- Indexed scene-text embeddings with **Faiss** for scalable retrieval in insurance-related geospatial queries.  
- Delivered internal demos for geospatial AI use cases (e.g., risk assessment, disaster zone matching).

## Technologies & Skills

### 🧠 Libraries, Frameworks & Tools

- **Core AI/ML Frameworks**:  
  PyTorch, PyTorch Lightning, HuggingFace Transformers (BERT, ViLT, BLIP, LLaVA, T5, CLIP), 
  PyTorch Geometric, OpenMMLab (MMDetection, MMCV)

- **Graph & NLP Tooling**:  
  Graph Transformers, NetworkX, LiteLLM, LangChain 

- **Retrieval & Embedding**:  
  Faiss, scikit-learn, NumPy, Pandas

- **MLOps & Prototyping**:  
  Docker, Conda, GitHub CI/CD, Weights & Biases (tracking)
