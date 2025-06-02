---
title: "Coin Counter: Deep Learning for Robust Multi-Class Coin Detection and Classification"
excerpt: "This project explores multiple deep learning models: CNNs, ResNet50, and Vision Transformers, for classifying coins in images. We use advanced segmentation techniques, custom over-segmentation filtering, and data augmentation to improve generalization. Developed for the EPFL IAPR course, this system demonstrates competitive accuracy and robustness for automated coin recognition tasks."
collection: course_projects
share: false
related: false
slidesurl: "../../files/Coin_counter.pptx"
---

## Problem Statement & Motivation

Accurately classifying coins in real-world images is a complex task due to variations in:
- Illumination and shadow,
- Image quality and scale,
- Occlusions and coin positioning.

This project addresses robust **coin classification** in a multi-class setting. It was developed as the final project for the EPFL course **Image Analysis and Pattern Recognition (IAPR)**. We aimed to evaluate and compare several modern computer vision approaches using both traditional CNNs,  ResNet50 and Vision Transformers (ViT).

## Method Overview

### 1. Image Segmentation

- A dedicated segmentation pipeline detects coin regions in input images.
- We added a **special "over-segmentation" class** to filter out noise or poorly segmented coins. If an image is predicted as such, it is excluded from the dataset.

### 2. Data Augmentation

To improve generalization and reduce overfitting, we applied:
- Random rotations,
- Brightness & contrast adjustments,
- Shifts in HSV color space,
- Gaussian noise.

These transformations were applied across all models.

## Models & Training

### Models
- CNN-Based Pipeline
- ResNet50
- Vision Transformer (ViT)

## Results

The models were evaluated both qualitatively and quantitatively:

- **Custom CNN** served as a lightweight and interpretable baseline, showing moderate results.
- **ViT**, implemented from scratch, struggled to generalize well due to limited optimization and training resources.
- **Fine-tuned ResNet50** delivered the best performance, benefiting from pretraining and stable skip connections.


## Future Work

- Improve segmentation accuracy to reduce false positives.
- Expand classes for different currencies or damaged coins.

