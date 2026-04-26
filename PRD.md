# Product Requirements Document (PRD)

## 1. Project Overview
This project provides an end-to-end Machine Learning pipeline framework for time-series forecasting. Originally built for bike rental demand forecasting, the architecture is designed to be highly modular and reproducible, enabling quick adaptation to new datasets and predictive goals. This PRD outlines the system architecture and serves as a blueprint for re-implementing the project with a new dataset.

## 2. Goals & Objectives
### Primary Goal
To maintain a scalable, easily deployable ML system capable of orchestrating model training and running real-time (or simulated rolling) inference with a live visualization dashboard.

### Secondary Goal
To adapt the existing Kedro-based architecture for a **new dataset** and a **new predictive target** while preserving the core orchestration, data flow, and visualization layers.

## 3. Architecture & Tech Stack
- **Pipeline Orchestration:** Kedro
- **Data Processing:** pandas
- **Machine Learning Models:** CatBoost (default/primary), Scikit-Learn (Random Forest, Linear Regression)
- **Storage Format:** Parquet (for efficient time-series storage and retrieval)
- **Web Interface:** Dash + Plotly + Bootstrap
- **Containerization:** Docker & Docker Compose (for training, inference, and UI services)
- **Package Management:** `uv`

## 4. Functional Requirements

### 4.1. Data Ingestion & Feature Engineering
- **Configurable Schema:** The system must handle time-series data containing a timestamp column and various numerical/categorical features.
- **Dynamic Feature Generation:** Ability to generate lag features (e.g., $t-1$, $t-2$, $t-N$) dynamically based on the configuration in `parameters.yml`.
- **Target Shifting:** The system must be able to shift the target variable forward by a specified horizon (e.g., predicting 1 step ahead) to create the training labels.

### 4.2. Model Training
- **Temporal Split:** Train/test split must respect the time dimension to avoid data leakage (no random shuffling).
- **Model Modularity:** System should allow switching between algorithms (CatBoost, RF, LR) seamlessly by updating `conf/base/parameters.yml`.
- **Artifact Management:** Trained models must be saved to `data/06_models/` in an appropriate format (`.cbm` for CatBoost, `.pkl` for Scikit-Learn).
- **Evaluation Metrics:** The training pipeline must compute and log standard regression metrics (MAE, RMSE, MAPE).

### 4.3. Inference
- **Rolling Inference Loop:** A background process (`entrypoints/inference.py`) must simulate real-time data arrival by processing small batches of historical/live data.
- **Statefulness:** The inference loop must continuously append new predictions to an output file (`data/07_model_output/predictions.parquet`) without overwriting historical predictions.
- **Configurable Pace:** The inference interval (sleep time) must be adjustable to simulate different data velocities.

### 4.4. Real-Time Dashboard (UI)
- **Live Updates:** The Dash UI must poll the predictions and actuals files at a configurable interval (e.g., every 1-2 seconds) without requiring a full page refresh.
- **Interactive Visualization:** The main chart must plot actuals vs. predictions, featuring a "future boundary" marker separating historical data from incoming predictions.
- **Lookback Control:** Users must be able to adjust the historical time window displayed on the chart.

## 5. Non-Functional Requirements
- **Reproducibility:** All parameters, data splits, and model configurations must be tracked via Kedro's parameter registry and Data Catalog.
- **Separation of Concerns:** Hardcoded file paths must be avoided. All data I/O must flow through Kedro's `conf/base/catalog.yml`.
- **Deployment:** The system must be launchable via a single `docker compose up --build` command, orchestrating `ml-train`, `ml-inference`, and `app-ui` containers in the correct sequence.

## 6. Migration Guide: Adapting to a New Dataset & Goal
To redo this project for a new dataset and goal, the following components must be modified:

### 6.1. Data Layer (`conf/base/catalog.yml`)
- Update file paths and dataset names for the new raw training and inference datasets in the `01_raw` namespace.
- Ensure the timestamp column format is consistent or update the parsing logic.

### 6.2. Configuration (`conf/base/parameters.yml`)
- **Target Variable:** Update the target column name to match the new objective.
- **Lag Features:** Redefine the `lag_params` dictionary based on the new data's seasonality (e.g., if the data is minute-by-minute, lag values will differ from hourly data).
- **Column Mapping:** Update the `rename_mapping` for standardizing raw columns.

### 6.3. Node Logic Updates (`src/.../pipelines/nodes.py`)
- The core nodes (`rename_columns`, `get_features`, `make_target`) are designed to be generic. However, they may need minor adjustments if the new dataset requires specific imputation strategies for missing values or handling of new categorical variables.

### 6.4. UI Updates (`src/app_ui/app.py`)
- Update dashboard titles, overview descriptions, and axis labels to reflect the new domain.
- Adjust the default `lookback_hours` (or adapt it to lookback *minutes/days*) based on the new time scale.

## 7. Deliverables for the New Project
1. **Cleaned & Integrated Data:** The new dataset properly formatted and registered in the Data Catalog.
2. **Updated Configuration:** Tuned `catalog.yml` and `parameters.yml` reflecting the new predictive goal.
3. **Retrained Models:** A new model artifact trained on the new dataset, achieving satisfactory evaluation metrics.
4. **Tailored Dashboard:** Dash application customized for the new use case.
