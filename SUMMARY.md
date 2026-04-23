# Repository Summary: free-bootcamp-mlacademy

## What This Project Does

Bike rental demand forecasting using a **Kedro ML pipeline**. Trains a model on historical bike-sharing data, then runs rolling inference to predict next-hour bike counts. A Dash web UI visualizes predictions vs actuals in real time.

---

## Tech Stack

| Layer | Tool |
|-------|------|
| Pipeline orchestration | Kedro 1.1.1 |
| ML models | CatBoost (primary), Random Forest, Linear Regression |
| Data format | Parquet (via pandas) |
| UI | Dash + Plotly + Bootstrap |
| Config | YAML (`conf/base/`) |

---

## Project Structure

```
├── conf/base/
│   ├── catalog.yml         # Maps dataset names → file paths
│   └── parameters.yml      # All configurable values
├── data/
│   ├── 01_raw/             # Input parquet files
│   ├── 03_primary/         # Intermediate (inference batch)
│   ├── 06_models/          # Trained model artifacts
│   └── 07_model_output/    # Predictions parquet
├── entrypoints/
│   ├── training.py         # Run training pipeline once
│   ├── inference.py        # Simulated rolling inference loop
│   └── app_ui.py           # Start Dash dashboard server
└── src/
    ├── free_bootcamp_mlacademy/
    │   └── pipelines/
    │       ├── nodes.py        # ALL node logic lives here
    │       ├── feature_eng.py  # Feature engineering pipeline
    │       ├── training.py     # Training pipeline
    │       └── inference.py    # Inference pipeline
    └── app_ui/
        ├── app.py              # Dash app + callbacks
        └── utils.py            # Chart creation helpers
```

---

## Data Flow

### Training

```
bike_data_train.parquet
    → rename_columns()       # hr → hour, cnt → bike_count, etc.
    → get_features()         # Create lag features (lag 1, 2, 22, 23 hours)
    → make_target()          # Shift bike_count +1 hour to create target
    → split_data()           # 80% train / 20% test (temporal, not random)
    → train_model()          # Fit CatBoost/RF/LR
    → predict()              # Evaluate on test set
    → compute_metrics()      # MAE, RMSE, MAPE
    → save_model()           # data/06_models/forecast_model.cbm
```

### Inference (rolling loop)

```
bike_data_inference.parquet  (sliced to batch of 30 rows)
    → inference_batch.parquet  (written by entrypoint)
    → rename_columns()
    → get_features()           # Same lag features as training
    → load_model()             # Load from data/06_models/
    → predict()                # Next-hour prediction
    → join_timestamps()        # Attach datetime column
    → predictions.parquet      # Appended each step
```

---

## Pipelines

Defined in `pipeline_registry.py`. Two registered pipelines:

| Name | Composition |
|------|-------------|
| `training` | `feat_eng_training` + `create_training_pipeline` |
| `inference` | `feat_eng_inference` + `create_inference_pipeline` |

Run via: `kedro run --pipeline training`

---

## Key Node Functions (`pipelines/nodes.py`)

| Function | What it does |
|----------|--------------|
| `rename_columns(df, mapping)` | Renames raw column names to descriptive ones |
| `get_features(df, lag_params)` | Creates lag features by shifting columns N periods back |
| `make_target(df, target_params)` | Shifts bike_count forward 1 hour to create prediction target |
| `split_data(df, params)` | Temporal train/test split at `train_fraction` (0.8) |
| `train_model(x, y, params)` | Instantiates and fits chosen model type |
| `predict(model, x)` | Runs `model.predict()`, returns DataFrame |
| `compute_metrics(y_true, y_pred)` | Returns dict: MAE, RMSE, MAPE |
| `save_model(model, type, storage)` | CatBoost → `.cbm`, sklearn → `.pkl` via joblib |
| `load_model(type, storage)` | Reverse of save_model |
| `join_timestamps(preds, ts)` | Attaches datetime to predictions for visualization |

---

## Lag Features Explained

Lag features capture temporal patterns by looking backward in time:

```yaml
lag_params:
  bike_count: [1, 2, 22, 23]   # 1h, 2h ago + ~daily pattern (22-23h ago)
  hour:        [1, 2, 3]
  temperature: [1, 2, 3]
  humidity:    [1, 2, 3]
```

`bike_count_lag_22` = bike count 22 hours ago. Helps the model learn daily seasonality (rush hour yesterday predicts rush hour today).

---

## Model Configuration

Switch models by editing `conf/base/parameters.yml`:

```yaml
training:
  model_type: catboost   # Options: catboost, cb, random_forest, rf, linear_regression, linreg
```

| Model | Format saved | Key params |
|-------|-------------|------------|
| CatBoost | `.cbm` | learning_rate=0.2, depth=6, iterations=50 |
| Random Forest | `.pkl` | n_estimators=100, max_depth=6 |
| Linear Regression | `.pkl` | (none) |

---

## Entrypoints

### `entrypoints/training.py`
- Runs `training` Kedro pipeline once
- Output: trained model in `data/06_models/`
- Use when: first training, retraining, hyperparameter experiments

### `entrypoints/inference.py`
- Loops `num_steps_inference` times (default: 2000)
- Each step: slices a 30-row batch → writes to `inference_batch.parquet` → runs inference pipeline → sleeps 2 seconds
- Output: predictions appended to `data/07_model_output/predictions.parquet`
- Simulates streaming data arriving in real time

### `entrypoints/app_ui.py`
- Starts Dash server at `http://localhost:8050`
- Auto-refreshes chart every 2 seconds
- Shows actual vs predicted bike counts
- Lookback slider controls time window displayed

---

## Dashboard (Dash UI)

Two-panel layout:
- **Left sidebar**: lookback hours input + ML overview text
- **Right main**: interactive Plotly chart
  - Green line = predicted bike counts
  - Red X markers = actual bike counts
  - Gray dashed vertical line = boundary between historical and future

Data loaded from parquet files on every tick — no caching. Dash callback triggered by `dcc.Interval` every 2000ms.

---

## Kedro Concepts Used

| Concept | How used here |
|---------|---------------|
| **Node** | Python function + named inputs/outputs |
| **Pipeline** | Ordered collection of nodes (DAG) |
| **Data Catalog** | `catalog.yml` maps names to parquet files |
| **Parameters** | `parameters.yml` injected into nodes via `params:` prefix |
| **Session** | `KedroSession.create()` runs pipelines in entrypoints |
| **Pipeline composition** | `pipeline_a + pipeline_b` merges two pipelines |

---

## Running the Full System

```bash
# Step 1: Train model
python entrypoints/training.py

# Step 2: Start inference loop (Terminal 1)
python entrypoints/inference.py

# Step 3: Start dashboard (Terminal 2)
python entrypoints/app_ui.py
# Open: http://localhost:8050
```

---

## Questions & Answers

**Q1: What is Kedro and why is it used here?**
Kedro is a Python framework for building ML pipelines. It separates data loading (catalog.yml), configuration (parameters.yml), and logic (nodes.py). It handles wiring: if node A outputs `features` and node B takes `features` as input, Kedro connects them automatically. Used here to keep the ML code modular, testable, and reproducible.

**Q2: What are lag features and why do we need them?**
Lag features are past values of a column shifted N timesteps back. `bike_count_lag_1` = bike count 1 hour ago. They give the model access to recent history so it can learn temporal patterns — like rush hour spikes. Without lags, the model only sees current conditions (temperature, hour) and can't learn trends or momentum.

**Q3: Why is 22-hour lag included?**
Daily seasonality. If today is 8am, 22 hours ago was 10am yesterday. That's not a perfect 24-hour pattern, but combined with `lag_23` (9am yesterday) and `lag_24` would give daily context. The lags [22, 23] capture the approximate "same time yesterday" window to handle daily patterns.

**Q4: Why is the train/test split temporal, not random?**
Time-series data has temporal ordering — future values depend on past values. A random split would "leak" future information into training (the model would train on data from the future). A temporal split ensures the model is always evaluated on data it has never seen, simulating real deployment conditions.

**Q5: What does `make_target()` actually do?**
It shifts `bike_count` forward by 1 row (1 hour). So for each row, the target becomes the bike count of the **next** hour. This way the model learns to predict the future from current features. Without this shift, the model would just predict the current count (which it already knows).

**Q6: What is the difference between training and inference feature engineering?**
They use the same node functions (`rename_columns`, `get_features`) but load from different sources. Training loads from `bike_data_train.parquet`. Inference loads from `inference_batch.parquet` (a rolling 30-row window). The training pipeline also adds `make_target()` to create the label — inference does not need a target.

**Q7: How does the inference loop simulate real-time data?**
`entrypoints/inference.py` reads the full historical inference dataset, then steps through it one row at a time. Each step takes the last 30 rows as a "batch", saves it to `inference_batch.parquet`, runs the inference pipeline, then sleeps 2 seconds before the next step. This mimics a real system where new sensor data arrives every hour.

**Q8: How does the Dash UI update in real time?**
`dcc.Interval` fires a callback every 2000ms. The callback reads the latest `predictions.parquet` and `bike_data_inference.parquet` from disk, creates a Plotly figure, and returns it. Dash re-renders the chart. No websockets or streaming — just periodic polling of parquet files.

**Q9: What is the Data Catalog and why does it matter?**
`catalog.yml` maps logical names (like `train_data`, `predictions_with_timestamps`) to actual file paths and types. Nodes reference dataset names, not paths. If the file location changes, you update one line in `catalog.yml` — not every node that uses that data. It also handles type conversion automatically (e.g., reading parquet into a pandas DataFrame).

**Q10: How do you switch from CatBoost to Random Forest?**
In `conf/base/parameters.yml`, change `training.model_type` from `catboost` to `random_forest`. Then retrain: `python entrypoints/training.py`. The `train_model()` node reads the model type, instantiates the correct class, and saves in the appropriate format (`.pkl` for sklearn models).

**Q11: What happens if `predictions.parquet` doesn't exist when the UI starts?**
`utils.py` wraps the parquet load in a try/except. If the file doesn't exist, it returns an empty DataFrame. The chart renders with empty or partial data. Once inference runs and creates the file, the next interval tick picks it up automatically.

**Q12: What is the purpose of `join_timestamps()`?**
`get_features()` drops the `datetime` column (it's not a feature) but captures timestamps separately. After `predict()` generates predictions, `join_timestamps()` reattaches the timestamp to each prediction row. This makes the output parquet file plottable on a time axis — otherwise predictions are just an unlabeled sequence of numbers.

**Q13: Why does CatBoost save as `.cbm` instead of `.pkl`?**
CatBoost has its own native binary format (`.cbm`) which is more efficient and portable than pickle. It also supports loading without the original Python object (useful in production). Sklearn models (RF, LR) don't have a native format, so they use joblib's `.pkl` format.

**Q14: What does `compute_metrics()` return and what do the metrics mean?**
Returns a dict with:
- **MAE** (Mean Absolute Error): Average absolute difference between predicted and actual. Easy to interpret — same units as bike count.
- **RMSE** (Root Mean Squared Error): Like MAE but penalizes large errors more (squares them first). Sensitive to outliers.
- **MAPE** (Mean Absolute Percentage Error): Percentage error. Scale-independent. Uses `epsilon=1e-8` to avoid division by zero when actual count is 0.

**Q15: Can you run inference without first training?**
No. `load_model()` looks for `forecast_model.cbm` or `forecast_model.pkl` in `data/06_models/`. If neither exists, it raises a `ValueError`. Training must run at least once before inference is possible.
