---
title: "Evaluating LLM Context Compression: Benchmarks, Agents, and Noise"
collection: research_projects
permalink: /research_projects/context-compression-eval/
excerpt: "How do you tell whether compressing an LLM's context actually cost you anything? At Compresr I built the evaluation programme that answers that: 12 long-context suites, a sandboxed harness for running coding agents under compression, a 200-task assistant-agent benchmark, and — the part that changed how we read all of it — a measurement of our own judging noise."
date: 2026-01-10
share: false
related: false
---

## Problem Statement & Motivation

Compression research has an uncomfortable property: **the compressed prompt always looks fine.** Fluent, on-topic, shorter. Whether it dropped the one number the question depended on is invisible until you run the downstream task — and by then you are measuring a chain of a compressor, a target model, and a judge, each with its own variance.

The result is a field where it is very easy to publish a win that is an artefact. I spent much of my time at [Compresr](/work/compresr/) building the machinery to avoid doing that.

## The Benchmark Harness

I created the company's benchmark repository and remain its largest contributor. It covers **12 long-context suites** — FinanceBench, QMSum, LongBench-v2, BigLaw Bench, CUAD, MultiHiertt, BrazLaw among them.

Two design decisions matter more than the suite list:

- **Every question is written out atomically.** A full sweep runs six hours; machines die inside six hours. Atomic writes mean a crashed run resumes instead of restarting.
- **Judging runs from stored model outputs, not live ones.** Rubrics change more often than models do. Decoupling the two means a rubric revision costs a judging pass, not a re-compression of everything.

## Agents Under Compression

Static QA is the easy case. The hard case is an **agent**, where compression changes what the model sees at step 3 and you only find out at step 40.

I built the harness for this: it cross-compiles the [Context Gateway](https://github.com/Compresr-ai/Context-Gateway) proxy for whichever architecture the sandbox turns out to be, caches the binary behind a file lock, drops it in front of the agent, and then reconciles the agent's full trajectory against cost and token telemetry — so a "cheaper" run that quietly took twice as many steps shows up as what it is.

On top of that I wrote **OpenClaw-Bench**, a **200-task** benchmark for assistant agents spanning productivity, research, writing and lifestyle work, verified with a mix of deterministic checks and rubric-based LLM judging.

## Measuring Our Own Noise

The most useful number I produced was not a score. It was the **noise floor of our own evaluation**:

| Setting | Rubric macro variability |
|---|---|
| Per-sample, temperature 0 | **±0.07** |
| Averaged over 10 samples | **±0.003** |

That figure is why we throw out differences that look like wins but sit inside their own error bars. It is also how we found something we would otherwise have shipped a wrong claim about: **compression works well as post-retrieval refinement, but it cannot replace retrieval.**

## Serving

The evaluation work sits directly on top of the inference work — our compression models served under **vLLM on EKS**, autoscaled by a custom CloudWatch metric that beat the AWS built-ins on GPU instance-hours. Cheap inference is what makes running 12 suites repeatedly affordable enough to be honest.

## Related

- [Cmprsr: Abstractive Token-Level Question-Agnostic Prompt Compressor](/publications/Cmprsr/) — the research line this grew out of, started at EPFL dlab.
- [Co-founder & CTO, Compresr](/work/compresr/) — the rest of the engineering.
