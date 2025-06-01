---
title: "Coin Counter: Deep Learning for Robust Multi-Class Coin Detection and Classification"
excerpt: "This project explores multiple deep learning models—including CNNs, ResNet50, and Vision Transformers—for classifying coins in images. We use advanced segmentation techniques, custom over-segmentation filtering, and aggressive data augmentation to improve generalization. Developed for the EPFL IAPR course, this system demonstrates competitive accuracy and robustness for automated coin recognition tasks."
collection: course_projects
share: false
related: false
---

## Problem Statement & Motivation

Accurately classifying coins in real-world images is a complex task due to variations in:
- Illumination and shadow,
- Image quality and scale,
- Occlusions and coin positioning.

This project addresses robust **coin classification** in a multi-class setting. It was developed as the final project for the EPFL course **Image Analysis and Pattern Recognition (IAPR)**. We aimed to evaluate and compare several modern computer vision approaches using both traditional CNNs and state-of-the-art architectures like ResNet50 and Vision Transformers (ViT).

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

### CNN-Based Pipeline

- Input: Augmented images with balanced classes.
- Model: Standard convolutional neural network.
- Loss: Cross-entropy loss for classification.
- Optimizer: AdamW.

### ResNet50

- Pretrained on ImageNet-1K.
- Uses skip connections to prevent vanishing gradients.
- Ends with average pooling and a 3-class fully connected layer.
- Activation: ReLU layers after each convolution.
- Trained using AdamW optimizer.

> Feature Design (Per Residual Block):
> - **1st layer**: Dimensionality reduction (256 → 64 channels),
> - **2nd layer**: 3×3 convolution to extract spatial features,
> - **3rd layer**: Restore dimensionality (64 → 256) for skip connection.

### Vision Transformer (ViT)

- Uses attention mechanisms to learn spatial hierarchies.
- Accepts flattened image patches instead of convolutions.
- Optimized using AdamW with cross-entropy loss.

## Results

The models were evaluated qualitatively and quantitatively:

- **ViT** showed strong performance on generalization.
- **ResNet50** produced the most stable and accurate classifications, thanks to its pretraining and skip connections.
- **Custom CNN** provided a lightweight and interpretable baseline.

## Future Work

- Improve segmentation accuracy to reduce false positives.
- Integrate with real-time video input or mobile deployment.
- Expand classes for different currencies or damaged coins.


## Project Report

- **Authors**: Oussama Gabouj, Salim Boussofara, Yasmine Chaker
- **Course**: EE-451 - Image analysis and pattern recognition
- **Institution**: EPFL  
- **PDF Report**: [Download Full Report](https://Ousso11.github.io/files/Coin counter.pptx)