---
title: "Generative Approaches to Kinetic Parameter Inference in Metabolic Networks via Latent Space Exploration"
collection: publications
permalink: /publications/LCSB/
redirect_from:
  - /publication/2025-04-05-latent-metabolic-generative-models
  - /publication/2025-04-05-latent-metabolic-generative-models/
excerpt: "Co-author, published in Nature Communications. We introduce a generative framework for constructing large-scale kinetic metabolic models through latent space exploration. By repurposing pretrained neural network generators across different physiological contexts, our method enables efficient and interpretable inference of kinetic parameters, facilitating targeted model design for diverse metabolic behaviors."
date: 2026-01-15
venue: 'Nature Communications'
paperurl: 'https://www.nature.com/articles/s41467-026-72184-3'
share: false
related: false
---

## Abstract
Generative machine learning methods that utilize neural networks to parameterize large-scale and near-genome-scale kinetic models have yielded significant efficiency gains in model construction, paving the way for high-throughput dynamic metabolism studies in biomedical and biotechnological applications. Nevertheless, challenges remain in interpreting the outputs of generative neural networks and developing strategies to quickly adapt these networks to different organisms and physiological contexts without having to restart the modeling process from scratch. Here, we present a systematic framework for repurposing generative neural networks trained on one physiological context to build large-scale kinetic models tailored for another context, thereby offering a new avenue for efficiently constructing models with targeted desired properties suitable for various physiological scenarios. We showcase the effectiveness of this method through three case studies: (i) adjusting the modeled response speed of cellular metabolism in aerobic E. coli cultures, (ii) improving interpretability by identifying key enzymatic steps that limit the dynamic response speed of the metabolic models, and (iii) adapting our neural network to capture the distinct dynamic behavior of anaerobic E. coli. Given the growing adoption of generative neural networks in biological systems modeling, our approach has the potential to advance personalized medicine and accelerate the high-throughput design of cell factories by streamlining model construction across diverse living organisms.

## Key Contributions

- **Latent Control**: Demonstrates how the generator’s latent space can be navigated to produce new kinetic models with desired properties.
- **Cross-Physiology Repurposing**: Allows transfer from one strain or condition (e.g., wild-type) to another (e.g., recombinant or fast-growing).
- **Scalable Parameter Inference**: Offers an interpretable and efficient alternative to Monte Carlo or ensemble methods.

## My Contribution

Generative-model work: using **GAN latent spaces** to infer kinetic parameters of metabolic networks, in place of expensive sampling-based estimation.

## Publication Details

- **Venue**: **Nature Communications**, 2026
- **Article**: [nature.com/articles/s41467-026-72184-3](https://www.nature.com/articles/s41467-026-72184-3)
- **Preprint**: [bioRxiv — DOI: 10.1101/2025.03.31.646317](https://www.biorxiv.org/content/10.1101/2025.03.31.646317v2)
- **Lab**: [LCSB, EPFL](https://www.epfl.ch/labs/lcsb/)

