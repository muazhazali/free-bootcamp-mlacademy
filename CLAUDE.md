# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies (uses uv lockfile)
pip install -e ".[dev]"

# Run full training pipeline (feature eng + training)
kedro run

# Run specific pipeline
kedro run --pipeline training
kedro run --pipeline inference

# Run training entrypoint directly
python entrypoints/training.py

# Run inference loop (simulates real-time, 2s between steps)
python entrypoints/inference.py

# Run Dash UI (port 8050)
python entrypoints/app_ui.py

# Lint
ruff check src/
ruff format src/

# Tests
pytest

# Single test
pytest tests/test_run.py::test_name

# Kedro viz (pipeline visualization)
kedro viz
```

## Architecture

This is a **Kedro** ML project for bike count forecasting using time-series data.

### Data flow

```
data/01_raw/              → raw parquet files (train + inference)
data/03_primary/          → inference_batch.parquet (rolling window)
data/06_models/           → serialized models (.cbm for CatBoost, .pkl for sklearn)
data/07_model_output/     → predictions.parquet (appended each inference step)
```

### Pipelines (`src/free_bootcamp_mlacademy/pipelines/`)

- **`feature_eng.py`** — two variants: `feat_eng_pipeline_training` and `feat_eng_pipeline_inference`. Calls `rename_columns` then `get_features` (lag creation).
- **`training.py`** — calls `make_target` → `split_data` → `train_model` → `save_model`.
- **`inference.py`** — calls `load_model` → `get_features` → `predict` → `join_timestamps` → saves predictions.

All node logic lives in **`pipelines/nodes.py`**. Pipelines wire nodes to catalog datasets.

### Entrypoints (`entrypoints/`)

These bypass `kedro run` CLI and drive pipelines programmatically:
- **`training.py`** — one-shot training run.
- **`inference.py`** — rolling inference loop: slices a batch window from inference data, writes it to `data/03_primary/inference_batch.parquet`, then runs the inference pipeline. Sleeps `inference_interval_seconds` between steps.
- **`app_ui.py`** — starts the Dash server. The UI polls predictions every 2s and renders actual vs forecast chart.

### Config (`conf/base/`)

- **`catalog.yml`** — dataset paths and types (all `pandas.ParquetDataset`).
- **`parameters.yml`** — feature engineering lag params, model hyperparams, model storage path, UI config, and inference loop config. Change `model_type` here to switch between `catboost`, `random_forest`, or `linear_regression`.

### Model support

`nodes.py:train_model` supports: `catboost`/`cb`, `random_forest`/`rf`, `linear_regression`/`linreg`. CatBoost saves as `.cbm`; sklearn models save as `.pkl` via joblib.

### UI (`src/app_ui/`)

Dash app at port 8050. Reads actual data and predictions parquet files on each interval tick. Configured via `parameters.yml → ui` section.
