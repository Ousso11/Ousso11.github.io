---
title: "Co-founder & CTO — Compresr Inc. (YC W26)"
collection: work
permalink: /work/compresr/
excerpt: "I co-founded Compresr out of EPFL's dlab to build context-compression infrastructure for LLMs. We were taken into Y Combinator W26. I am the infrastructure engineer on a four-person team: the multi-tenant cloud platform, the on-premise product, an open-source Go proxy, the GPU serving layer, and the benchmark harness we make decisions with."
location: San Francisco, USA / Lausanne, Switzerland
share: false
related: false
---

**Sep 2025 – Present** · [compresr.ai](https://compresr.ai) · Y Combinator W26

## 🧠 What the Company Does

Long context is expensive and, past a point, actively harmful — models degrade, agents stall on compaction, and bills scale with tokens nobody reads. **Compresr** builds compression infrastructure that sits between an application and its LLM: it decides what to keep, keeps it, and lets the rest be recovered on demand.

The company spun out of **EPFL's [dlab](https://dlab.epfl.ch)** and was selected for **Y Combinator W26**. I am the only infrastructure engineer on a four-person team.

## 🔧 What I Build

### Context Gateway — open-source Go proxy
[github.com/Compresr-ai/Context-Gateway](https://github.com/Compresr-ai/Context-Gateway) · **637 ★** · Apache-2.0

A proxy that sits between a coding agent and its LLM and keeps the context window small. The interesting part is timing: it summarises the conversation in a **background worker** and intercepts the agent's own compaction request, so compacting returns something that already exists instead of stalling the session.

- **Compression is reversible.** Tool outputs are swapped for references, and a phantom `expand_context` tool is slipped into the request, so the model can pull the original back when it needs it — without the host agent ever knowing the tool was added.
- **Eight LLM backends** behind one adapter interface, including re-signing SigV4 for Bedrock after the request body has been rewritten.
- Works with **Claude Code, Codex, Cursor, OpenCode and OpenClaw**; I wrote the OpenClaw plugin.
- **40K+ lines of Go with 1K+ tests**. Ships as a single cgo-free binary with the React dashboard embedded, released for five platforms across 12 versions.

### Cloud platform
Multi-tenant API and billing: **135 HTTP endpoints across 24 routers**, **Postgres row-level security (74 policies)** for tenant isolation, Redis write-behind buffering, Stripe billing, SSE streaming. **2,411 tests.**

### On-premise / BYOC product
For customers who cannot send data out: three deployment channels from one image, Ed25519-JWT auth, HMAC-verified telemetry, a six-layer security gate, and cosign + SBOM + OpenVEX supply-chain attestation.

### GPU serving & infrastructure
Our compression models served under **vLLM on EKS**, autoscaled by a **custom CloudWatch EMF metric** that meaningfully outperformed the AWS built-ins on GPU instance-hours. **14 reusable Terraform modules** and 7 root stacks underneath.

### Client SDKs
Matching **Python (`compresr` on PyPI)** and **TypeScript (`@compresr/sdk` on npm)** clients with sync, async, batch and streaming calls. Dual ESM/CJS builds and every peer dependency optional, so installing the SDK never drags a user's pinned `openai` or `anthropic` version along with it. Five integrations — LangChain, LangGraph, LlamaIndex, a LiteLLM guardrail, a Hermes plugin — over a shared core, with **1,000+ tests**, published from CI by tag rather than from a laptop.

## 📊 Evaluation

I created the company's benchmark repository and remain its largest contributor. It covers **12 long-context suites** — FinanceBench, QMSum, LongBench-v2, BigLaw Bench, CUAD, MultiHiertt, BrazLaw — and is built to survive reality: every question is written out atomically so a six-hour sweep can resume after a crash, and judging runs from stored outputs so a rubric change does not mean re-compressing everything.

I also wrote the harness for testing **coding agents under compression**: it cross-compiles the Go proxy for whichever architecture the sandbox turns out to be, caches the binary behind a file lock, drops it in front of the agent, then reconciles the agent's trajectory against cost and token telemetry. On top of that sits **OpenClaw-Bench**, a **200-task** benchmark for assistant agents across productivity, research, writing and lifestyle work, verified with a mix of deterministic checks and rubric-based LLM judging.

We **measured our evaluation noise** rather than assuming it: **±0.07 per-sample at temperature 0**, and **±0.003 at ten samples**. That number is why we throw out differences that look like wins but are not — and it is also how we found that compression works as post-retrieval refinement but cannot replace retrieval.

## 🚀 Production Impact

Production integrations have delivered substantial inference-cost and latency reductions without a quality regression — in at least one case the LLM-judge score came out marginally *higher* after compression than before. Every rollout is staged behind parity gates, so a quality drop blocks the change rather than being discovered later.

## 🔐 Beyond Code

- **Primary security owner** for our **SOC 2 Type I**.
- Author of the engineering interview process we hire with.
- Upstream: hardened the Compresr guardrail in **LiteLLM** over 12 commits and three merged PRs — closed SSRF holes including DNS rebinding and IMDS access, isolated the recovery store so one tenant cannot read another's originals, added byte caps and LRU eviction, and took the suite to 45+ tests.

## 🛠️ Stack

**Languages:** Go • Python • TypeScript • SQL  
**Serving:** vLLM • SageMaker • GPU autoscaling • Kubernetes  
**Backend:** FastAPI • PostgreSQL (RLS) • Redis • Stripe • SSE  
**Cloud:** AWS (EKS, ECS Fargate, SageMaker, ElastiCache, ECR, ALB, CloudWatch) • Terraform • Kubernetes • Karpenter  
**Practices:** pytest / Vitest / Playwright • CI/CD • Sentry • Prometheus • SBOM & container security • SOC 2
