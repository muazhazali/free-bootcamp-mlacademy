# EXPLANATION.md — Detailed Architecture & Code Walkthrough

## What This System Does

Predicts how many bikes will be rented **next hour** based on historical patterns. It is a full ML system with three separate runtime components:

1. **Training** — builds and saves a model from historical data
2. **Inference** — runs the model on new data in a loop, saving predictions
3. **Dashboard** — reads saved predictions and shows a live chart

These three components are **decoupled** — they don't call each other directly. They communicate through **parquet files on disk**.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          DISK (shared state)                            │
│                                                                         │
│  data/01_raw/bike_data_train.parquet    ← training source               │
│  data/01_raw/bike_data_inference.parquet ← inference source             │
│  data/03_primary/inference_batch.parquet ← rolling batch (written each step) │
│  data/06_models/forecast_model.cbm      ← saved trained model          │
│  data/07_model_output/predictions.parquet ← output (appended each step) │
└───────────┬─────────────────────────┬───────────────────────┬──────────┘
            │                         │                       │
            ▼                         ▼                       ▼
   ┌─────────────────┐    ┌─────────────────────┐   ┌──────────────────┐
   │  entrypoints/   │    │  entrypoints/        │   │  entrypoints/    │
   │  training.py    │    │  inference.py        │   │  app_ui.py       │
   │                 │    │                      │   │                  │
   │  Runs ONCE      │    │  Loops 2000 times    │   │  Runs forever    │
   │  Kedro pipeline │    │  Kedro pipeline      │   │  Dash server     │
   │  → saves model  │    │  → appends preds     │   │  → reads preds   │
   └─────────────────┘    └─────────────────────┘   └──────────────────┘
```

The three entrypoints are **run in separate terminals concurrently**. Training must finish first (model must exist). Then inference + UI can run simultaneously.

---

## Kedro: The Pipeline Framework

Before going into the code, you need to understand Kedro's three concepts used here:

### 1. Node
A Python function wrapped with named inputs and outputs:
```python
node(
    func=rename_columns,
    inputs=["input_data", "params:feature_engineering.rename_columns"],
    outputs="renamed_data",
)
```
- `inputs` — dataset names from the catalog, or `params:` to inject from parameters.yml
- `outputs` — what name the return value gets (so the next node can reference it)
- Kedro wires nodes together by matching output names to input names

### 2. Pipeline
An ordered collection of nodes. Kedro resolves dependencies automatically — if node B needs `renamed_data` and node A outputs `renamed_data`, Kedro runs A before B.

```python
Pipeline([node_a, node_b, node_c])
```

Pipelines can be composed with `+`:
```python
training_full = feat_eng_pipeline + training_pipeline
```

### 3. Data Catalog (`catalog.yml`)
Maps logical names → physical file paths. Nodes never hardcode file paths — they reference catalog names.

```yaml
train_data:
  type: pandas.ParquetDataset
  filepath: data/01_raw/bike_data_train.parquet
```

When a node lists `"train_data"` as input, Kedro reads that parquet file and passes the DataFrame automatically.

---

## The Two Full Pipelines

Defined in `pipeline_registry.py`:

```python
"training"  → feat_eng_pipeline_training()  + create_training_pipeline()
"inference" → feat_eng_pipeline_inference() + create_inference_pipeline()
```

Both pipelines share the same feature engineering logic but load from different sources and end differently.

---

## Pipeline 1: Training

### Full Node Chain

```
train_data (parquet)
    │
    ▼
load_data(df)
    → returns: (input_data DataFrame, last_timestamp)
    │
    ▼
rename_columns(input_data, rename_dict)
    → returns: renamed_data DataFrame
    │
    ▼
get_features(renamed_data, lag_params)
    → returns: (features DataFrame, timestamps Series)
    │
    ▼
make_target(features, target_params)
    → returns: data_with_target DataFrame
    │
    ▼
split_data(data_with_target, params)
    → returns: x_train, x_test, y_train, y_test
    │
    ▼
train_model(x_train, y_train, params)
    → returns: trained_model
    │         │
    ▼         ▼
predict(trained_model, x_test)    save_model(trained_model, model_type, storage)
    → returns: predictions             → writes forecast_model.cbm to disk
    │
    ▼
compute_metrics(y_test, predictions)
    → returns: metrics dict (MAE, RMSE, MAPE)
```

### What each node does (with code)

---

#### `load_data(df)` — `nodes.py:223`

```python
def load_data(df: pd.DataFrame) -> Tuple[pd.DataFrame, pd.Timestamp]:
    last_timestamp = pd.to_datetime(df["datetime"]).iloc[-1]
    return df, last_timestamp
```

Kedro injects the parquet file as `df`. The node extracts the last timestamp (used later by `join_timestamps` in inference). Returns the same DataFrame unchanged plus the timestamp.

---

#### `rename_columns(df, renaming_dict)` — `nodes.py:11`

```python
def rename_columns(df, renaming_dict):
    return df.rename(columns=renaming_dict)
```

Raw parquet files use abbreviated column names from the original UCI dataset:

| Raw name | Renamed to |
|----------|-----------|
| `hr` | `hour` |
| `temp` | `temperature` |
| `hum` | `humidity` |
| `windspeed` | `wind_speed` |
| `cnt` | `bike_count` |
| `weathersit` | `weather` |
| `weekday` | `week_day` |

The mapping comes from `parameters.yml → feature_engineering.rename_columns`.

---

#### `get_features(df, lag_params)` — `nodes.py:16`

```python
def get_features(df, lag_params):
    for feature, lags in lag_params.items():
        for lag in lags:
            df[f"{feature}_lag_{lag}"] = df[feature].shift(lag).bfill()
    timestamps = pd.to_datetime(df['datetime'])
    df.drop(columns=['datetime'], inplace=True)
    return df, timestamps
```

This is the most important feature engineering step. It creates **lag features** — copies of columns shifted backward in time.

**Example with `bike_count` and lags `[1, 2, 22, 23]`:**

| Row | bike_count | bike_count_lag_1 | bike_count_lag_2 | bike_count_lag_22 | bike_count_lag_23 |
|-----|------------|-----------------|-----------------|------------------|------------------|
| 0   | 100        | 100 (backfill)  | 100 (backfill)  | 100 (backfill)   | 100 (backfill)   |
| 1   | 120        | 100             | 100 (backfill)  | 100 (backfill)   | 100 (backfill)   |
| 2   | 90         | 120             | 100             | 100 (backfill)   | 100 (backfill)   |
| 23  | 80         | (prev value)    | (prev value)    | 100              | 100 (backfill)   |

**Why lag 22 and 23?** To capture the approximate "same time yesterday" pattern. If today is 10am, 22 hours ago was 12am, and 23 hours ago was 11pm yesterday. Combined with `lag_1` and `lag_2`, the model sees short-term trend and daily seasonality.

The `datetime` column is dropped at the end — it's not a numeric feature. The timestamps are extracted separately and returned so they can be re-attached to predictions later.

Full lag parameters from `parameters.yml`:
```yaml
lag_params:
  bike_count: [1, 2, 22, 23]
  hour: [1, 2, 3]
  temperature: [1, 2, 3]
  humidity: [1, 2, 3]
```

---

#### `make_target(df, target_params)` — `nodes.py:40`

```python
def make_target(df, target_params):
    df[target_params["new_target_name"]] = (
        df[target_params["target_column"]].shift(-target_params["shift_period"]).ffill()
    )
    return df
```

Creates the prediction label. Shifts `bike_count` **forward** by 1 row (negative shift = look ahead).

**Why shift forward?** The model learns: "given features at time T, predict bike_count at time T+1". This is next-hour forecasting.

| Row (hour) | bike_count (actual) | target (what we want to predict) |
|------------|--------------------|---------------------------------|
| 8am        | 100                | 150 (what happens at 9am)       |
| 9am        | 150                | 200 (what happens at 10am)      |
| 10am       | 200                | 130 (what happens at 11am)      |

Parameters from `parameters.yml`:
```yaml
target_params:
  shift_period: 1          # How many steps ahead to predict
  target_column: bike_count
  new_target_name: target
```

---

#### `split_data(df, params)` — `nodes.py:48`

```python
def split_data(df, params):
    target_name = params["target_params"]["new_target_name"]
    features = [col for col in df.columns if col != target_name]
    x, y = df[features], df[target_name]
    train_size = int(params["train_fraction"] * len(df))
    x_train, x_test = x[:train_size], x[train_size:]
    y_train, y_test = y[:train_size], y[train_size:]
    return x_train, x_test, y_train, y_test
```

**Temporal split** — not random. With `train_fraction: 0.8`, first 80% of rows become training data, last 20% become test data.

**Why not random?** Time-series data has temporal order. Randomly shuffling would allow the model to "see the future" during training (e.g., training on rows from June while testing on rows from April). That's data leakage. The temporal split ensures the model is always evaluated on truly unseen future data.

```
[========== 80% train ==========][== 20% test ==]
(chronological order preserved)
```

---

#### `train_model(x_train, y_train, params)` — `nodes.py:65`

```python
def train_model(x_train, y_train, params):
    model_type = params["model_type"].lower().strip()
    model_params = params["model_params"][model_type]

    if model_type in ["catboost", "cb"]:
        model = CatBoostRegressor(**model_params)
    elif model_type in ["random_forest", "rf"]:
        model = RandomForestRegressor(**model_params)
    elif model_type in ["linear_regression", "linreg"]:
        model = LinearRegression(**model_params)
    else:
        raise ValueError(f"Unknown model type: {model_type}")

    model.fit(x_train, y_train)
    return model
```

Reads `model_type` from config, instantiates the right class, fits on training data.

**Model comparison:**

| Model | How it learns | Strengths | Weaknesses |
|-------|--------------|-----------|------------|
| CatBoost | Gradient boosting (builds many weak trees, each correcting the last) | High accuracy, handles missing values, fast | Black box, needs tuning |
| Random Forest | Builds many independent trees, averages results | Robust, less overfitting | Slower inference, less accurate |
| Linear Regression | Fits a weighted sum of features | Interpretable, very fast | Assumes linear relationships (often wrong) |

Active model set in `parameters.yml → training.model_type`. Default: `catboost`.

---

#### `predict(model, x)` — `nodes.py:108`

```python
def predict(model, x):
    y_pred = pd.DataFrame(model.predict(x), columns=["prediction"])
    print(f"Predictions {y_pred}")
    return y_pred
```

Calls `.predict()` on whatever model was passed. Wraps result in a DataFrame with column `"prediction"`. Used in both training (evaluation on test set) and inference (real predictions).

---

#### `compute_metrics(y_true, y_pred)` — `nodes.py:118`

```python
def compute_metrics(y_true, y_pred):
    y_true = np.array(y_true).ravel()
    y_pred = np.array(y_pred).ravel()

    mae  = float(mean_absolute_error(y_true, y_pred))
    rmse = np.sqrt(mean_squared_error(y_true, y_pred))
    mape = np.mean(np.abs((y_true - y_pred) / y_true + 1e-8)) * 100

    return {'MAE': round(mae,2), 'RMSE': round(rmse,2), 'MAPE': round(mape,2)}
```

Three metrics, each measuring error differently:

| Metric | Formula | What it tells you |
|--------|---------|------------------|
| MAE | mean(\|actual - predicted\|) | Average absolute error in bike units. Easy to interpret. |
| RMSE | sqrt(mean((actual - predicted)²)) | Like MAE but bigger errors are penalized more (squared). Sensitive to outliers. |
| MAPE | mean(\|actual - predicted\| / actual) × 100 | Percentage error. Scale-independent. `1e-8` prevents divide-by-zero when actual = 0. |

Only used in training (not inference) since inference has no known ground truth at prediction time.

---

#### `save_model(model, model_type, model_storage)` — `nodes.py:158`

```python
def save_model(model, model_type, model_storage):
    model_dir = Path(model_storage["path"])
    model_name = model_storage["name"]
    
    if model_type in ["catboost", "cb"]:
        model.save_model(str(model_dir / f"{model_name}.cbm"))
    else:
        joblib.dump(model, model_dir / f"{model_name}.pkl")
```

CatBoost has a native binary format (`.cbm`) — faster and more portable than pickle. sklearn models use joblib's `.pkl` format.

Result: `data/06_models/forecast_model.cbm` (or `.pkl`)

---

## Pipeline 2: Inference

### Full Node Chain

```
inference_batch.parquet  (30-row window, written by entrypoint)
    │
    ▼
load_data(df)
    → returns: (input_data DataFrame, last_timestamp)
    │
    ▼
rename_columns(input_data, rename_dict)
    → returns: renamed_data DataFrame
    │
    ▼
get_features(renamed_data, lag_params)
    → returns: (features DataFrame, timestamps Series)
    │
    ├──────────────────────────────────────┐
    ▼                                      │
load_model(model_type, model_storage)      │
    → returns: model                       │
    │                                      │
    ▼                                      │
predict(model, features)                   │
    → returns: predictions DataFrame       │
    │                                      │
    ▼                                      │
join_timestamps(predictions, timestamps) ←─┘
    → returns: predictions with datetime column
    │
    ▼
predictions_with_timestamps.parquet  (appended each step)
```

**Key difference from training:** No `make_target`, `split_data`, `compute_metrics`, or `save_model`. Inference just loads an existing model and generates predictions.

---

#### `load_model(model_type, model_storage)` — `nodes.py:189`

```python
def load_model(model_type, model_storage):
    model_dir = Path(model_storage["path"])
    model_name = model_storage["name"]
    
    if model_type in ["catboost", "cb"]:
        model = CatBoostRegressor()
        model.load_model(str(model_dir / f"{model_name}.cbm"))
    else:
        model = joblib.load(model_dir / f"{model_name}.pkl")
    return model
```

Reverse of `save_model`. Loads the artifact from disk. If the file doesn't exist (training was never run), this raises an error.

---

#### `join_timestamps(predictions, timestamps)` — `nodes.py:229`

```python
def join_timestamps(predictions, timestamps):
    predictions["datetime"] = timestamps
    return predictions
```

`get_features()` drops the `datetime` column because it's not a numeric feature. But the output predictions need a datetime column so they can be plotted on a time axis.

This node reattaches the timestamps (which were extracted and passed through the pipeline as a separate output) to the final predictions DataFrame.

---

## The Inference Entrypoint Loop

`entrypoints/inference.py` is the engine that drives the inference pipeline repeatedly. It does NOT just call `kedro run` once — it simulates streaming data arriving over time.

```python
for step in range(num_steps):                          # 2000 iterations
    current_idx = first_idx + step
    batch_start = max(0, current_idx - batch_size + 1) # rolling window start
    batch_end   = current_idx + 1

    batch = data.iloc[batch_start:batch_end].copy()    # last 30 rows

    batch.to_parquet(batch_path, index=False)          # write to disk

    with KedroSession.create(...) as session:
        session.run(pipeline_name="inference")         # run kedro pipeline

    time.sleep(interval_seconds)                       # wait 2 seconds
```

**Rolling window visualization:**

```
Step 1:  [rows 0..29]   → predict row 30's bike count
Step 2:  [rows 1..30]   → predict row 31's bike count
Step 3:  [rows 2..31]   → predict row 32's bike count
...
```

Each step:
1. Writes a new 30-row window to `inference_batch.parquet`
2. Runs the inference Kedro pipeline (which reads that file)
3. The pipeline appends one prediction row to `predictions.parquet`
4. Sleeps 2 seconds (simulating 1 hour of real time per step)

The predictions file grows by one row every 2 seconds. The dashboard reads this file on each interval tick.

---

## The Dashboard (Dash UI)

### Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│  dbc.Container (Bootstrap, gray background)                             │
│  ┌──────────────────────┐  ┌──────────────────────────────────────────┐ │
│  │  dbc.Col width=3     │  │  dbc.Col width=9                         │ │
│  │  (sidebar)           │  │  (main chart area)                       │ │
│  │                      │  │                                          │ │
│  │  "Control Panel"     │  │  "Real-time Bike Count Predictions"      │ │
│  │  ┌────────────────┐  │  │  ┌────────────────────────────────────┐  │ │
│  │  │ lookback-hours │  │  │  │  dcc.Graph id="graph"              │  │ │
│  │  │ number input   │  │  │  │                                    │  │ │
│  │  └────────────────┘  │  │  │  Green line = Predicted            │  │ │
│  │                      │  │  │  Red X = Actual                    │  │ │
│  │  ML Overview bullet  │  │  │  Gray dashed = time boundary       │  │ │
│  │  points              │  │  └────────────────────────────────────┘  │ │
│  └──────────────────────┘  └──────────────────────────────────────────┘ │
│                                                                         │
│  dcc.Interval id="interval" fires every 2000ms (hidden component)      │
└─────────────────────────────────────────────────────────────────────────┘
```

### How the Auto-Refresh Works

`dcc.Interval` is an invisible Dash component that fires an event every N milliseconds. The callback is wired to fire whenever either the interval ticks or the lookback hours input changes:

```python
@callback(
    Output("graph", "figure"),
    [Input("lookback-hours", "value"),
     Input("interval", "n_intervals")]
)
def update_graph(lookback_hours, _):
    df_actual = load_data(ACTUAL_DATA_PATH)
    df_pred   = load_data(PREDICTIONS_PATH)
    figure    = create_figure(df_actual, df_pred, lookback_hours)
    return figure
```

Every 2 seconds:
1. Read `bike_data_inference.parquet` (actuals) from disk
2. Read `predictions.parquet` (model output) from disk
3. Build Plotly figure
4. Return to Dash — browser re-renders chart

No WebSockets, no streaming. Pure polling of parquet files.

### How the Chart is Built (`utils.py:create_figure`)

```python
def create_figure(df_actual, df_pred, lookback_hours):
    max_time  = df_pred["datetime"].max()          # most recent prediction
    min_time  = max_time - Timedelta(hours=lookback_hours)  # window start
    current_time = max_time - Timedelta(hours=1)   # boundary between actual/predicted

    # Filter both datasets to the time window
    # Add green line for predictions
    fig.add_trace(go.Scattergl(x=..., y=df_pred["prediction"], name="Predicted", ...))
    # Add red X markers for actuals
    fig.add_trace(go.Scattergl(x=..., y=df_actual["cnt"], name="Actual", ...))
    # Add vertical gray dashed line at end of actual data
    fig.add_vline(x=last_actual_time, line_dash="dash", line_color="gray")
```

**Why `Scattergl` instead of `Scatter`?** `Scattergl` uses WebGL rendering — much faster for large datasets with thousands of points.

**The vertical line** marks where actual data ends and predictions begin. Left of line = known history. Right of line = model forecasts.

---

## Configuration Files

### `conf/base/parameters.yml` — The Control Panel

All tunable values live here. Change these without touching any Python code.

```yaml
feature_engineering:
  rename_columns:          # Column name mapping (raw → descriptive)
    hr: hour
    cnt: bike_count
    # ...
  lag_params:              # Which lags to create for which features
    bike_count: [1, 2, 22, 23]
    hour: [1, 2, 3]
    temperature: [1, 2, 3]
    humidity: [1, 2, 3]

training:
  target_params:
    shift_period: 1        # How many hours ahead to predict
    target_column: bike_count
    new_target_name: target
  train_fraction: 0.8      # 80% train, 20% test
  model_type: catboost     # ← Change this to switch models
  model_params:
    catboost:
      learning_rate: 0.2   # How fast the model learns (lower = more careful)
      depth: 6             # Max tree depth (higher = more complex patterns)
      iterations: 50       # Number of trees (higher = better but slower)
      loss_function: RMSE  # What error to minimize

model_storage:
  path: data/06_models
  name: forecast_model     # Saved as forecast_model.cbm or .pkl

ui:
  update_interval_ms: 2000 # Dashboard refresh rate
  default_lookback_hours: 24

pipeline_runner:
  batch_size: 30                         # Rows per inference window
  num_steps_inference: 2000              # How many steps to simulate
  inference_interval_seconds: 2          # Seconds between steps
  first_timestamp: '2012-06-01 00:00:00' # Where in the data to start
```

### `conf/base/catalog.yml` — File Path Registry

```yaml
train_data:                    # Used by: training pipeline
  type: pandas.ParquetDataset
  filepath: data/01_raw/bike_data_train.parquet

inference_data:                # Used by: inference entrypoint (reads full dataset)
  type: pandas.ParquetDataset
  filepath: data/01_raw/bike_data_inference.parquet

inference_batch:               # Used by: inference pipeline (reads 30-row window)
  type: pandas.ParquetDataset  # Written by: inference entrypoint each step
  filepath: data/03_primary/inference_batch.parquet

predictions_with_timestamps:   # Used by: dashboard (reads)
  type: pandas.ParquetDataset  # Written by: inference pipeline (appends)
  filepath: data/07_model_output/predictions.parquet
```

The catalog is the **contract** between components. If you change where a file lives, update catalog.yml once — all pipeline nodes and entrypoints that reference the logical name update automatically.

---

## How to Run the Full System

```bash
# Terminal 1: Train model (run once)
python entrypoints/training.py
# Output: data/06_models/forecast_model.cbm + metrics printed

# Terminal 2: Start inference loop
python entrypoints/inference.py
# Output: predictions.parquet grows by 1 row every 2 seconds

# Terminal 3: Start dashboard
python entrypoints/app_ui.py
# Open: http://localhost:8050
```

---

## Complete Data Flow Diagram

```
bike_data_train.parquet
        │
        ▼
   [Training Pipeline]
        │
   load_data()
        │
   rename_columns()
        │
   get_features()  ──── creates 13 lag columns
        │
   make_target()   ──── shifts bike_count +1 hour
        │
   split_data()    ──── 80/20 temporal split
        │
   train_model()   ──── fits CatBoost on training rows
        │         \
   predict()    save_model() ──────────────────────► forecast_model.cbm
        │
   compute_metrics() ──── prints MAE, RMSE, MAPE


bike_data_inference.parquet
        │
        ▼
[entrypoints/inference.py]  ←──── loops 2000 times
        │
   slice 30 rows
        │
   write inference_batch.parquet
        │
        ▼
   [Inference Pipeline]
        │
   load_data()
        │
   rename_columns()
        │
   get_features()  ──── same lag logic as training
        │
   load_model() ◄──────────────────────────── forecast_model.cbm
        │
   predict()
        │
   join_timestamps()
        │
   predictions.parquet  ◄──── appended each step


predictions.parquet ──────────────────────────────────────────┐
bike_data_inference.parquet ──────────────────────────────────┤
                                                              ▼
                                               [Dash Dashboard]
                                                 reads both every 2s
                                                 renders live chart
```

---

## Key Design Decisions

### Why Kedro?
Without Kedro, you'd write one big Python script with hardcoded file reads and writes. Kedro forces you to separate:
- **What** the data is (catalog.yml)
- **How** to transform it (nodes.py)
- **In what order** (pipeline definitions)
- **With what settings** (parameters.yml)

This makes each piece independently testable, swappable, and reproducible.

### Why Parquet?
- Columnar storage → fast for reading specific columns
- Compressed → smaller than CSV
- Preserves dtypes (datetime stays datetime, not string)
- Append-friendly for predictions

### Why decouple training / inference / UI?
Each component can be deployed separately (e.g., in Docker containers with shared volumes). The dashboard doesn't need to know about Kedro. The inference loop doesn't need to know about the UI. They only share files.

### Why rolling window for inference?
The model needs lag features — it must see the last 23 hours of bike counts to create `bike_count_lag_23`. A rolling batch of 30 rows gives enough history to compute all lags. The entrypoint slides this window forward one row at a time to simulate time passing.
