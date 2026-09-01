---
title: "Open Source"
permalink: /open_source/
author_profile: true
---

Most of what I have built in the last year is public. This is the part worth looking at.

---

## 🚪 Context Gateway

**Go · Apache-2.0 · 637 ★ · [github.com/Compresr-ai/Context-Gateway](https://github.com/Compresr-ai/Context-Gateway)**

A proxy that sits between a coding agent and its LLM and keeps the context window small.

The interesting part is **timing**. Every agent eventually hits its context limit and compacts, and compaction is a blocking summarisation call — you sit there watching a spinner. Context Gateway summarises the conversation in a **background worker** and then intercepts the agent's own compaction request, so compacting returns something that already exists. The stall disappears.

The second idea is that **compression is reversible**. Tool outputs are swapped for references rather than deleted, and a phantom `expand_context` tool is slipped into the request on the way past — so the model can pull the original back when it turns out to need it, without the host agent ever knowing a tool was added.

- **8 LLM backends** behind one adapter interface, including re-signing SigV4 for Bedrock *after* the request body has been rewritten.
- **5 agent integrations**: Claude Code, Codex, Cursor, OpenCode, OpenClaw — I wrote the OpenClaw plugin.
- **40K+ lines of Go and 1K+ tests.** One cgo-free binary with the React dashboard embedded, released for 5 OS/arch targets over 12 versions.

Install and configuration are covered in the [repository README](https://github.com/Compresr-ai/Context-Gateway).

---

## 📦 Compresr SDKs

**Python · TypeScript · [`compresr` on PyPI](https://pypi.org/project/compresr/) · [`@compresr/sdk` on npm](https://www.npmjs.com/package/compresr)**

Matching clients for the compression API, with sync, async, batch and streaming calls.

Two constraints shaped them. **Dual ESM/CJS builds**, because half the JavaScript ecosystem is still on one and half on the other. And **every peer dependency optional**, so installing the SDK never drags a user's pinned `openai` or `anthropic` version along with it — the failure mode that makes people uninstall an SDK and never come back.

Five integrations — **LangChain, LangGraph, LlamaIndex, a LiteLLM guardrail and a Hermes plugin** (three of them also in TypeScript) — over a shared core. **1,000+ tests**, published from CI by tag rather than from a laptop.

---

## 🔧 Upstream Contributions

**[LiteLLM](https://github.com/BerriAI/litellm)** — security hardening of the Compresr guardrail over **12 commits and 3 merged PRs**: closed SSRF holes including DNS rebinding and IMDS access, isolated the recovery store so one tenant cannot read another's originals, added byte caps and LRU eviction, and took the test suite to 45+.

**[Hermes Agent](https://github.com/NousResearch)** (NousResearch) — co-authored two plugins covering context compaction and tool-output compression with content-addressed recovery.

---

## 📄 Research Code

**[GRAD](https://github.com/charafkamel/GRAD-demonstration-sampler)** — training code and models for [GRAD](/publications/GRAD/) (Findings of EMNLP 2025): an LLM trained with GRPO to generate few-shot demonstrations rather than retrieve them.

---

## 🧪 Benchmarks

**OpenClaw-Bench** — a **200-task** benchmark for assistant agents across productivity, research, writing and lifestyle work, verified with deterministic checks plus rubric-based LLM judging. Written as part of the [evaluation programme](/research_projects/context-compression-eval/) behind our compression claims.
