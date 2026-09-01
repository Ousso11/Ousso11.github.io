---
permalink: /
title: "👋 About Me"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

I'm **Oussama Gabouj**, co-founder and CTO of **[Compresr](https://compresr.ai)** (**Y Combinator W26**), where we build context-compression infrastructure for LLMs. I hold an **MSc in Data Science** from **EPFL** with a **minor in Cyber Security**.

My work is about making language models behave under tight context budgets — first as a researcher at **[EPFL's dlab](https://dlab.epfl.ch)**, where I was first author at **EMNLP 2025 Findings**, and now as the person who builds and runs the systems that put that research into production: the API and billing platform, an open-source Go proxy, the GPU serving layer, and the benchmark harness we use to decide whether a change is real.

I also keep an **[Engineering & Research Journal](/journal/)** — write-ups of bugs and measurement failures worth remembering: a GPU that silently ran on CPU, an autoscaling study that crowned a policy which never scaled, a benchmark headline that didn't survive correct cost accounting.

## 🚀 Currently

**Co-founder & CTO, [Compresr Inc.](https://compresr.ai)** — San Francisco / Lausanne | **Sep 2025 – Present**

We spun out of EPFL's dlab, were taken into **Y Combinator W26**, and raised **$1.2M+**. I'm the infrastructure engineer on a four-person team, which in practice means:

- **[Context Gateway](https://github.com/Compresr-ai/Context-Gateway)** — an open-source Go proxy (**637 ★**, Apache-2.0) that sits between a coding agent and its LLM and keeps the context window small. It summarises in a background worker and intercepts the agent's own compaction request, so compacting returns something that already exists instead of stalling the session. Works with Claude Code, Codex, Cursor, OpenCode and OpenClaw.
- **The cloud platform** — a multi-tenant API with Postgres row-level security, Redis buffering and Stripe billing, plus an on-premise product for customers who can't send data out.
- **GPU serving** — our compression models served under vLLM on EKS, autoscaled by a custom CloudWatch metric.
- **The benchmark harness** — 12 long-context suites, plus a sandboxed setup for running coding agents under compression.

One production integration cut a customer's inference cost **55% ($33K/yr)** and their latency **3×**, with answer quality slightly *up* afterwards.

## 📄 Selected Publications

- **[GRAD: Generative Retrieval-Aligned Demonstration Sampler for Efficient Few-Shot Reasoning](/publications/GRAD/)** — *first author (equal contribution)*, **EMNLP 2025 Findings**. Instead of retrieving few-shot examples from a database, we train an LLM with GRPO to write them per question.
- **[Cmprsr: Abstractive Token-Level Question-Agnostic Prompt Compressor](/publications/Cmprsr/)** — *co-author*, arXiv:2511.12281, under review at ACL 2026.
- **[Generative approaches to kinetic parameter inference in metabolic networks](/publications/LCSB/)** — *co-author*, **Nature Communications**, 2026.

## 🎓 Education

**MSc in Data Science**, Minor in Cyber Security — GPA 5.41/6  
*Master Thesis:* Structured Representations for Fine-Grained Text-to-Image Retrieval in Remote Sensing (Prof. Devis Tuia, EPFL ENAC — hosted by AXA Group Operations)  
**EPFL, Switzerland** | **2023 – 2025**

**BSc in Microengineering** — GPA 5.33/6  
**EPFL, Switzerland** | **2020 – 2023**

## 🏢 Labs & Industry Experience

### Research Labs @ EPFL
- [**DLab**](https://dlab.epfl.ch): The Data Science Lab focuses on transforming large-scale data into meaningful insights by developing algorithms in natural language processing, machine learning, and computational social science. I spent a year there as a research assistant with Prof. Robert West.
- [**LCSB**](https://www.epfl.ch/labs/lcsb/): The Laboratory of Computational Systems Biotechnology specializes in reconstructing and analyzing biological networks to understand cellular processes through computational models.
- [**DISAL**](https://www.epfl.ch/labs/disal/): The Distributed Intelligent Systems and Algorithms Laboratory develops methodologies for distributed, intelligent systems, emphasizing cyber-physical systems like multi-robot systems and sensor networks. I joined on a competitive *Summer in the Lab* fellowship.

### Industry
- [**Compresr Inc.**](https://compresr.ai): LLM context-compression infrastructure. Y Combinator W26.
- [**AXA Group Operations (Switzerland)**](https://www.axa.ch/en/private-customers.html): The IT services division of AXA, focusing on creating innovative technology and data solutions to support AXA's ambition of being a customer-focused, tech-led company.
- [**Pixalione (Paris)**](https://www.pixalione.co.uk/): A digital marketing agency specializing in SEO, Paid Media, and Data Analytics, combining human expertise with proprietary algorithmic tools to optimize web presence.

## 🏆 Honours & Awards

- **Y Combinator W26** — selected for the Winter 2026 batch.
- **$1.2M+ pre-seed** raised for Compresr as technical co-founder.
- **Summer in the Lab Scholarship**, DISAL, EPFL — a competitive fellowship funding a full-time summer of research (2023).

## 🛠️ Technical Skills

- **Languages**: Python, Go, TypeScript, SQL, C/C++, Bash
- **ML & LLMs**: PyTorch, HuggingFace Transformers, TRL, PEFT/LoRA, vLLM, reinforcement learning (GRPO, PPO, reward design), DPO, quantisation
- **LLM systems**: LangChain, LangGraph, LlamaIndex, LiteLLM, MCP, agent tooling, RAG and long-context evaluation
- **Backend**: FastAPI, PostgreSQL (row-level security), Redis, Stripe, REST/SSE streaming APIs, Next.js, React
- **Cloud & infra**: AWS (EKS, ECS Fargate, SageMaker, ElastiCache, ECR, ALB, CloudWatch), Terraform, Docker, Kubernetes, Karpenter, GitHub Actions
- **Practices**: pytest / Vitest / Playwright, CI/CD, observability (Sentry, Prometheus), container security & SBOM, SOC 2
