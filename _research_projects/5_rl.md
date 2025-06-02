---
title: "From Novice to Expert: Dimensionality Reduction and Policy Distillation in Reinforcement Learning for Motor Control"
excerpt: "This project investigates how to accelerate motor skill acquisition in reinforcement learning using curriculum-based learning, dimensionality reduction, and policy distillation. Using the Myosuite Baoding balls task, we explore how expert policies can be transferred to novice agents via PCA-reduced feature and action spaces, offering an efficient alternative to prolonged training times."
collection: research_projects
share: false
related: false
reporturl: "../../files/Reinforcement_Learning_report.pdf"

---

## Problem Statement & Motivation

Understanding biological motor control, like manipulating objects using muscle coordination, is a long-standing challenge in neuroscience and robotics. The control complexity increases significantly due to the high degrees of freedom and nonlinear interactions in the musculoskeletal system.

This project tackles the **transition from novice to expert** policies in reinforcement learning (RL) through efficient strategies, focusing on the *Baoding balls* task from the NeurIPS **MyoChallenge**. Inspired by Chiappa et al.’s **Static-to-Dynamic Stabilization (SDS)** curriculum, we explore whether it's possible to transfer **motor synergies** learned by expert agents to novices using dimensionality reduction.

## Our Method

We build upon recurrent PPO (Proximal Policy Optimization with LSTM) for policy learning and investigate two key **policy distillation** strategies:

### 1. Train Novice Agent Using Phase II Architecture
- Agent is first trained on Phase I (counter-clockwise rotations) using architecture built for more complex Phase II (clockwise, hold, friction variations).
- This serves as our baseline expert model.

### 2. Dimensionality Reduction for Policy Transfer

We analyze and compress both **feature and action spaces** using Principal Component Analysis (PCA):

- **Feature space reduction**: From the hidden layer of expert’s policy network.
- **Action space reduction**: From muscle activation outputs of expert policy.
  
Two architectures were used:
- **Static PCA layer**: Uses frozen expert PCs.
- **Dynamic bottleneck layer**: Learns compression end-to-end.

We also test projection/back-projection methods and latent space exploration for constrained agent behavior.

## Results & Analysis

| Experiment | Method                              | Mean Reward | Episode Length |
|------------|-------------------------------------|-------------|----------------|
| 1          | Static PCA on feature space         | 485         | 121.4          |
| 2          | Dynamic bottleneck on features      | 321         | 111.3          |
| 3          | PCA on action space (16 PCs)        | 41          | 22.5           |
| 4          | Latent action exploration           | 39          | 21.5           |
| Expert     | Full expert policy                  | 990.4       | 200            |
| Baseline   | No guidance                         | –           | –              |

- **Best results** came from **Experiment 1**, showing PCA can reduce training complexity without major performance loss.
- **Experiments 3/4** highlight that **over-reduction** in action space leads to failure in complex tasks (e.g. rotation).
- Findings show a tradeoff: **more compression = faster learning**, but with risk of losing critical motor signals.

## Key Takeaways

- A sweet spot exists in feature reduction: ~40 PCs for features, ~16 for action retained ~90% variance.
- Policy distillation via **static PCA** is effective and can cut training time from hundreds to under 24 hours.
- PCA might not fully capture task-specific nuances—methods like **Independent Component Analysis (ICA)** could be more suitable.
- Placement of dimensionality reduction layers in the network matters; future work should explore this further.

**PDF Poster**: [Download Full Report](../../files/Reinforcement_Learning_Poster.pdf)

