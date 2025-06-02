---
title: "Kinetic Parameter Inference in Metabolic Networks via Latent Space Exploration"
excerpt: "We present a novel framework to interpret and control the latent spaces of generative neural network models for kinetic metabolic modeling. By perturbing structured latent spaces learned via REKINDLE or RENAISSANCE, our method generates new dynamic models with targeted properties such as specific response times, regulatory bottlenecks, or alternative physiologies, unlocking deeper insight and reusability across metabolic contexts."
collection: research_projects
date: 2025-04-05
paperurl: https://www.biorxiv.org/content/10.1101/2025.03.31.646317v1
share: false
related: false
---

## Problem Statement & Motivation

Modeling dynamic metabolic behavior is central to biomedical and biotechnological advances, ranging from drug design to microbial consortia engineering. However, kinetic models are limited by incomplete biochemical data and parameterization difficulties. Recently, **generative machine learning GAN** approaches like **REKINDLE** and **RENAISSANCE** have enabled rapid creation of dynamic models using trained neural network generators.

But one question remains: *how can we control and repurpose these trained models to explore entirely new physiological behaviors or guide biotechnological applications?*

## Our Method

We propose a framework that leverages the **latent space** of generative ML models to gain control over metabolic dynamics:

1. **Latent Space Mapping**: A generator is trained using REKINDLE/RENAISSANCE to map a simple Gaussian latent space to the kinetic parameter space of a target physiology.
2. **Latent Perturbation & Sampling**: We perturb and sample the latent space to generate **novel kinetic models** with specific dynamic properties.
3. **Property-Specific Exploration**: We use latent space axes to explore and control features like:
   - Dynamic response times.
   - Bottlenecks in enzymatic regulation.
   - Cross-physiology generalization (e.g., wild-type → recombinant strains).

The structured nature of the latent space allows nonlinear decoupling of features—making this an interpretable, scalable approach to dynamic metabolic modeling.

## Applications

We demonstrate the flexibility of this method in three metabolic modeling scenarios:

- **Tuning dynamic response times** in *E. coli* under aerobic growth.
- **Identifying bottleneck enzymes** through latent-based property navigation.
- **Repurposing generators** across physiologies (e.g., from slow- to fast-growing strains).

