---
title: "AI Research Intern — AXA Group Operations"
collection: work_experience
excerpt: "I led applied research and prototyping efforts in multimodal AI, focusing on cross-modal representation learning, graph-based embeddings, and neural search systems. I developed scalable pipelines to generate scene graphs from satellite imagery and knowledge graphs from textual data, and to align their graph embeddings in a shared representation space."
location: Lausanne, Switzerland
share: false
related: false
---

## 🧠 Problem Statement & Motivation

Insurance companies require better tools to analyze satellite imagery and textual reports for risk profiling, disaster assessment, and infrastructure mapping. Traditional models fall short in capturing relationships across data modalities.

The goal was to create a **graph-based retrieval system** aligning **scene graphs** from satellite imagery and **knowledge graphs** from text in a shared latent space — enabling semantic search across modalities.

## 🔧 My Contributions

- Built pipelines to extract **scene graphs from satellite images** using MMDetection and OpenMMLab.
- Generated **knowledge graphs** from textual descriptions via LangChain and graph parsing techniques.
- Trained **graph transformer encoders** to align multimodal graphs using **contrastive learning**.
- Indexed graph embeddings using **Faiss** for scalable neural retrieval.
- Delivered **interactive demos** for internal use cases.

## 🧪 Evaluation & Ablation

- Assessed the system’s ability to generalize in both **closed-set** and **open-vocabulary** settings, reflecting real-world insurance use cases.
- Benchmarked against standard retrieval baselines (CLIP, ViLT, BLIP) using metrics such as **Recall@K** and **mean Average Precision (mAP)**.
- Performed detailed **ablation studies** to analyze the impact of:
  - Graph structure type: **scene graphs vs. knowledge graphs**
  - **Retrieval modes**: node-only, edge-only, and hybrid configurations
  - **Contrastive loss variants** and alignment strategies
- Results showed superior robustness and generalization, especially in **open-set scenarios** involving unseen objects / relations.


## 🚀 Technology Stack

### 🧠 ML/DS Tools
- Graph learning: PyTorch Geometric, Graph Transformers, NetworkX  
- Language & vision models: HuggingFace (BLIP, LLaVA, Qwen), CLIP, Bert, RoBerta 
- Knowledge pipelines: LangChain, LiteLLM, Open AI 
- Retrieval: Faiss, scikit-learn, NumPy, Pandas  

### 🖥️ DevOps & Prototyping
- GitHub CI/CD for continuous integration  
- Docker + Conda for environment setup  
- Weights & Biases for model tracking and experiments