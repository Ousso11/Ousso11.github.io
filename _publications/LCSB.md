---
title: "Generative Approaches to Kinetic Parameter Inference in Metabolic Networks via Latent Space Exploration"
collection: publications
# category: manuscripts # conferences
permalink: /publication/2025-04-05-latent-metabolic-generative-models
excerpt: "We present a novel generative framework that leverages latent space exploration to generate dynamic metabolic models with targeted properties. This work introduces a new approach to controllably infer kinetic parameters in large-scale biological systems using pretrained neural network generators such as REKINDLE and RENAISSANCE."
date: 2025-04-05
venue: 'bioRxiv'
paperurl: 'https://www.biorxiv.org/content/10.1101/2025.03.31.646317v1'
# slidesurl: ''
# bibtexurl: 'https://www.biorxiv.org/content/10.1101/2025.03.31.646317v1.article-info'
share: false
related: false
---

## Abstract

Dynamic kinetic models of metabolism are essential for understanding how cells adapt to internal and external changes. However, generating such models remains limited by scarce experimental data and computational challenges in parameter estimation. This paper introduces a novel framework that leverages **generative machine learning models** and their **latent space** representations to explore and control dynamic behaviors in metabolic systems.

We build on existing approaches like **REKINDLE** and **RENAISSANCE**, training neural network generators to learn the mapping from a Gaussian latent space to valid kinetic parameter sets. By exploring the structure of this latent space, we can fine-tune specific model properties such as dynamic response times, regulatory bottlenecks, or even transfer learned behaviors across different physiological conditions.

## Key Contributions

- **Latent Control**: Demonstrates how the generator’s latent space can be navigated to produce new kinetic models with desired properties.
- **Cross-Physiology Repurposing**: Allows transfer from one strain or condition (e.g., wild-type) to another (e.g., recombinant or fast-growing).
- **Scalable Parameter Inference**: Offers an interpretable and efficient alternative to Monte Carlo or ensemble methods.

## Publication Details

- **Date**: April 5, 2025  
- **Preprint**: [bioRxiv - DOI: 10.1101/2025.03.31.646317](https://www.biorxiv.org/content/10.1101/2025.03.31.646317v1)  
- **License**: CC-BY-NC-ND 4.0  
- **PDF**: [Download Full Paper](https://www.biorxiv.org/content/10.1101/2025.03.31.646317v1.full.pdf)

