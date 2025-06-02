---
title: "Distributed Movie Recommendation Pipelines with Apache Spark"
excerpt: "This project builds a full-scale movie recommendation system using Apache Spark, incorporating data analytics, keyword-based filtering. Implemented on the MovieLens dataset, the system supports efficient data preprocessing, incremental rating updates, and personalized movie recommendations through LSH and predictive models."
collection: course_projects
share: false
related: false
---

## Problem Statement & Motivation

Modern data-driven applications in video streaming rely on real-time analytics and personalization to provide relevant content. This project aims to explore large-scale, distributed data processing for movie recommendation tasks using **Apache Spark**. By implementing Spark-based pipelines, we address:

- The need for scalable movie rating aggregation and analytics.
- The demand for efficient **personalized recommendation systems** using distributed collaborative filtering.
- The use of **locality-sensitive hashing (LSH)** for scalable similarity search in keyword-based movie descriptions.

## Our Method

We implemented a **multi-phase pipeline** based on the MovieLens dataset, covering:

### 1. Data Preprocessing & Analytics

- Loaded movie metadata and rating logs from pipe-separated files.
- Implemented Spark RDD transformations for grouping, filtering, and rating statistics (e.g., most-rated movies by year or genre).

### 2. Rating Aggregation & Incremental Updates

- Built an **aggregator** for computing average ratings per movie.
- Enabled **incremental update logic** to efficiently maintain aggregates after new ratings.

### 3. Keyword-based Filtering

- Supported genre/keyword filtering using both:
  - Standard filtering
  - **Broadcast variables** for optimization

### 4. Prediction Models & Recommenders

- Implemented a **baseline predictor** using user rating normalization.
- Applied **collaborative filtering via ALS** using Spark MLlib.
- Developed **LSH-based indexing** for near-neighbor search on movie genres.
- Combined LSH + prediction for **personalized movie recommendations**.

## Evaluation

The system supports both local development and distributed Spark execution on a cluster (via EPFL IC infrastructure). It was validated through:

- **Automated unit tests** provided by CS460 instructors.
- Performance benchmarking on datasets of varying sizes (small to large).
- Accurate top-N recommendations for users based on previous ratings.

