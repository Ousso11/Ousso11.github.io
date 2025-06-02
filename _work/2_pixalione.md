---
title: "Predictive Budget Forecasting for Google Ads Campaigns - Pixalione"
excerpt: "Developed a machine learning pipeline to forecast daily ad spend on Google Ads based on client-specific campaign data. Deployed a web backend for dynamic budget strategy adjustment, automated alerts, and integration with Azure Cloud infrastructure."
collection: work_experience
share: false
related: false
---

## 🧠 Problem Statement & Motivation

Each client manages one or more Google Ads campaigns with a fixed daily budget. Spending occurs per keyword match and is driven by live auction dynamics, click rates, and impressions. However, most clients lack tools to **anticipate daily trends**, often resulting in inefficient budget use.

Our goal was to create a predictive system that helps clients **understand and control their spending behavior**, improving their ROI by proactively adjusting campaign strategies.

## 🔧 My Contributions

- Built a robust **machine learning pipeline** to predict daily ad spend using past trends in **click-through rates, impressions, and user profiles**.
- Enabled **per-client budget optimization**, integrating live forecasts into business logic.
- Deployed a **Flask-based web API** to serve predictions and trigger **real-time notifications**.
- **Daily Forecast Email**: Clients receive a daily email with the forecast of today’s spending, helping them stay informed and plan accordingly.

## 🚀 Technology Stack

### 🧠 ML/DS Tools
- scikit-learn, XGBoost, Facebook Prophet, LightGBM,
- Pandas, NumPy for feature engineering and preprocessing  
- Time-series modeling: seasonal decomposition, exponential smoothing (ETS), ARIMA, Prophet  
- Deep learning: RNN, LSTM for sequential trend prediction  
- Tree-based models: Decision Trees, Random Forest, Gradient Boosted Trees  
- Model evaluation: cross-validation

### 🖥️ DevOps & Backend
- **GitLab CI/CD** for automation and testing  
- **Docker** for containerized deployment  
- **Azure Cloud** for hosting and data storage  
- **Flask + REST API** for real-time interaction  

## Outcomes

- Established **Forecast Accuracy Baseline**: Created the initial baseline for ad spend forecast accuracy, enabling future measurement and improvement efforts.

- Enabled **hands-free campaign monitoring** for clients with low technical overhead.
