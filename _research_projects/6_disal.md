---
title: "Learning-Based Multi-Robot Lane Navigation: Scalable Trajectory Prediction using Neural Networks"
excerpt: "This project was conducted at [DISAL, EPFL](https://www.epfl.ch/labs/disal/). We explore trajectory generation for multi-robot navigation using neural networks. We propose a scalable alternative to Webots simulation by training models using reinforcement and imitation learning. The final approach produces accurate trajectories in a lane-based environment, balancing precision and efficiency in robotic control."
collection: research_projects
share: false
related: false
reporturl: "../../files/DISAL_final_report.pdf"

---

## Problem Statement & Motivation

Accurate trajectory prediction is crucial for safe and efficient robot navigation. While high-fidelity simulators like **Webots** offer reliable results, they lack scalability for real-time deployment or multi-agent scenarios. The goal of this project is to implement a **scalable neural network** model that can generate robot trajectories efficiently, with acceptable precision.

The task involves navigating a **cyclic path** with a variable number of lanes, introducing additional planning complexity.

## Our Method

We investigated a range of **machine learning techniques** to model robot motion:

1. **Environment Setup**:
   - The navigation task was simulated in Webots.
   - The environment includes a **cyclic lane path** with adjustable width and complexity (see Figure 1).

2. **Trajectory Prediction**:
   - Input: initial robot position.
   - Output: incremental prediction of next positions using the neural network.
   - The model is **autoregressive**: each predicted step becomes the input for the next.

3. **Learning Approaches**:
   - **MLP**, **RNN**, and **GNN**: struggled to maintain trajectory accuracy.
   - **Reinforcement Learning (PPO)**: yielded stable behaviors.
   - **Imitation Learning**: learned from expert trajectories.
   - **Combined RL + Imitation Learning**: achieved **best performance** in terms of accuracy and scalability.

4. **Simulation Loop**:
   - The predicted movement is integrated step-by-step (xₜ → xₜ₊₁).
   - The learned policy generalizes over multiple steps (see Figure 2).

## Evaluation & Results

- The **green trajectory**: ground-truth expert path from Webots.
- The **red trajectory**: neural network output.
- Figure 3 shows close alignment between the model prediction and expert simulation.
- The best-performing model used **imitation learning guided by reinforcement learning**, balancing precision with efficiency.

### Observations

- **Training time** is substantial and environment-specific.
- The learned controller is **scalable** and can potentially extend to **multi-agent settings** with further training.

