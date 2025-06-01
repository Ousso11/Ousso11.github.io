---
title: "GemmaEdu: Enhancing Scientific Learning via Fine-Tuned Language Models and RAG"
excerpt: "We developed an educational chatbot built on the quantized Gemma 7B model, optimized with Direct Preference Optimization (DPO) and enhanced with Retrieval-Augmented Generation (RAG). By leveraging fine-tuning on student-generated preference data and incorporating relevant external documents, our system significantly improves accuracy in answering STEM multiple-choice questions, outperforming baseline models like Mistral and Llama2."
collection: research_projects
share: false
related: false
reporturl: "../../files/GemmaEdu.pdf"
---

## Problem Statement & Motivation

Large language models are increasingly central to student learning, but they often fail to answer **scientific multiple-choice questions (MCQs)** accurately, especially in STEM subjects. Existing models lack proper alignment with educational contexts and struggle with returning concise, grounded responses.

To address this, **GemmaEdu** aims to create a **fine-tuned, preference-aligned educational assistant** that integrates **retrieval mechanisms** to enhance factual precision and context relevance.

## Our Method

We built an advanced educational chatbot based on the **quantized Gemma 7B** model and improved it through:

### 1. Fine-Tuning with Direct Preference Optimization (DPO)

- **Preference data** was generated from 100+ MCQs by EPFL Master students across CS, physics, EE, and AI topics.
- We fine-tuned the model with **DPO-only** and **SFT + DPO** approaches.
- The best-performing model used only DPO on a merged dataset (M1 + Argilla), improving both **accuracy** and **margin** scores.

### 2. Retrieval-Augmented Generation (RAG)

- We built a **RAG pipeline** using LlamaIndex + ChromaDB and the BAAI/bge-small-en-v1.5 embedding model.
- Documents included:
  - Open-source textbooks across relevant subjects.
  - Wikipedia pages extracted via GPT keyword generation.
  - RAKE keyword summaries from student questions.

### 3. Prompt Engineering

- We experimented with **Chain-of-Thought (CoT)** prompts, few-shot examples, and **verbalizers** to extract single-letter MCQ answers.
- The final pipeline included generating hints first, then prompting for a concise choice (e.g., “A”, “B”).

## Evaluation & Benchmarks

- **Dataset**: Subset of **MMLU** (focusing on 22 STEM subjects).
- **Metrics**: Accuracy and margin on MCQ responses.
- **Comparisons**: Baselines include:
  - `unsloth/gemma-7b-bnb-4bit`
  - Llama2-7B
  - Mistral-7B

## Results

- **Accuracy** improved across all categories, notably in physics and EE.
- Our model **outperforms all baselines** in most subjects and in overall MMLU STEM performance.
- RAG integration showed **selective improvements**, especially outside the computer science domain.

| Subject       | Edu (RAG) | Gemma | Mistral | Llama2 |
|---------------|-----------|-------|---------|--------|
| Physics       | **41.2**  | 34.3  | 24.5    | 21.5   |
| EE            | **51.7**  | 46.2  | 39.3    | 24.0   |
| CS            | 42.0      | 42.0  | 32.0    | 23.0   |
| Overall STEM  | **44.4**  | 36.4  | 36.0    | 35.2   |

## Challenges & Future Work

- **Noise in student-generated data** limited DPO training efficiency.
- **RAG chunk tuning** and **prompt wording** were crucial for effectiveness.
- Future improvements include:
  - Larger multilingual RAG databases.
  - Better prompt patterns for concise answer generation.
  - Real-time feedback and adaptability features.


## Project Report

- **Authors**: Oussama Gabouj, Salim Boussofara, Yasmine Chaker
- **Course**: CS-552 - Modern Natural Language Processing
- **Institution**: EPFL  
