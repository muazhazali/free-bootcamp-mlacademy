# FUTURE-PLAN.md — Energy Load Forecasting (New Repo)

## Goal

Predict next-hour electricity load (MW) for a European country.
Same Kedro architecture as bike repo. Replace bike data with energy data.

---

## Dataset Options (Ranked Best → Worst)

### Option 1 — Open Power System Data (OPSD) ✅ RECOMMENDED

**URL:** https://data.open-power-system-data.org/time_series/

**Why best:**
- No registration required
- Direct CSV download (no API key, no scraping)
- Pre-cleaned hourly load for 36 European countries (2006–2020)
- Already merged into one large CSV
- Used widely in ML research — well-documented

**What to download:**
- Go to https://data.open-power-system-data.org/time_series/
- Download `time_series_60min_singleindex.csv` (hourly, all countries)
- Filter to one country column e.g. `DE_load_actual_entsoe_transparency` (Germany)

**Columns you'll use:**

| Column | Meaning |
|--------|---------|
| `utc_timestamp` | Datetime (hourly) |
| `DE_load_actual_entsoe_transparency` | Germany actual load in MW |
| `DE_temperature` | (if available) temperature |

---

### Option 2 — EIA API (US Data)

**URL:** https://www.eia.gov/opendata/

**Why decent:**
- Free, hourly, covers all 64 US balancing authorities
- Up-to-the-hour data (truly real-time)
- Requires free API key (email registration)

**How to get key:** https://www.eia.gov/opendata/ → Register → get key by email

**Example API call:**
```
https://api.eia.gov/v2/electricity/rto/region-data/data/
  ?api_key=YOUR_KEY
  &frequency=hourly
  &data[0]=value
  &facets[type][]=D        (D = demand)
  &facets[respondent][]=MISO
  &start=2022-01-01T00
  &end=2024-01-01T00
  &sort[0][column]=period
  &sort[0][direction]=asc
  &length=5000
```

**Downside:** Needs API key, rate limits, US-only.

---

### Option 3 — ENTSO-E Transparency Platform (European)

**URL:** https://transparency.entsoe.eu/

**Why third:**
- Most authoritative European source
- Requires registration (free but extra step)
- Registration → CSV export or REST API via `entsoe-py` library

**Python library:** `pip install entsoe-py`
```python
from entsoe import EntsoePandasClient
client = EntsoePandasClient(api_key='YOUR_KEY')
ts = client.query_load('DE', start=pd.Timestamp('2022-01-01', tz='UTC'),
                                end=pd.Timestamp('2024-01-01', tz='UTC'))
```

---

## Recommended Dataset Decision

**Use OPSD** (Option 1). No key, direct download, clean CSV, works offline.
Country: **Germany (DE)** — largest European grid, clean data, good seasonality.

---

## Weather Features (Free, No Key)

Add temperature/wind/solar as features using Open-Meteo:

**URL:** https://open-meteo.com/en/docs/historical-weather-api

No API key required. Example Python fetch:
```python
import openmeteo_requests
import requests_cache
import pandas as pd
from retry_requests import retry

cache_session = requests_cache.CachedSession('.cache', expire_after=-1)
retry_session = retry(cache_session, retries=5, backoff_factor=0.2)
openmeteo = openmeteo_requests.Client(session=retry_session)

url = "https://archive-api.open-meteo.com/v1/archive"
params = {
    "latitude": 52.52,        # Berlin
    "longitude": 13.41,
    "start_date": "2015-01-01",
    "end_date": "2020-12-31",
    "hourly": ["temperature_2m", "wind_speed_10m", "shortwave_radiation"]
}
responses = openmeteo.weather_api(url, params=params)
```

Install: `pip install openmeteo-requests requests-cache retry-requests`

---

## Repo Structure (Mirror of Bike Repo)

```
energy-load-forecast/
├── conf/base/
│   ├── catalog.yml           # dataset paths
│   └── parameters.yml        # lag params, model config, UI config
├── data/
│   ├── 01_raw/
│   │   ├── load_train.parquet
│   │   └── load_inference.parquet
│   ├── 03_primary/
│   │   └── inference_batch.parquet
│   ├── 06_models/
│   │   └── forecast_model.cbm
│   └── 07_model_output/
│       └── predictions.parquet
├── entrypoints/
│   ├── training.py
│   ├── inference.py
│   └── app_ui.py
└── src/
    ├── energy_load_forecast/
    │   └── pipelines/
    │       ├── nodes.py
    │       ├── feature_eng.py
    │       ├── training.py
    │       └── inference.py
    └── app_ui/
        ├── app.py
        └── utils.py
```

---

## Feature Engineering Changes vs Bike Repo

Energy load has stronger seasonality than bike counts. Add:

| Feature | Why |
|---------|-----|
| `hour` | Daily peak/off-peak pattern |
| `day_of_week` | Weekday vs weekend load differs significantly |
| `month` | Seasonal heating/cooling demand |
| `is_holiday` | Holidays drop load significantly |
| `temperature` (from Open-Meteo) | Heating/cooling directly drives load |
| `load_lag_1` | 1 hour ago |
| `load_lag_2` | 2 hours ago |
| `load_lag_24` | Same hour yesterday |
| `load_lag_168` | Same hour last week |

**Key difference from bike repo:** Add `lag_168` (7 days × 24 hours = weekly pattern). Energy load has strong weekly seasonality (factories shut on weekends).

---

## Parameters (Suggested Starting Config)

```yaml
feature_engineering:
  lag_params:
    load_mw: [1, 2, 24, 168]
    temperature: [1, 2, 24]

training:
  target_params:
    shift_period: 1
    target_column: load_mw
    new_target_name: target
  train_fraction: 0.8
  model_type: catboost
  model_params:
    catboost:
      learning_rate: 0.05
      depth: 8
      iterations: 300
      loss_function: RMSE
      verbose: 50
      random_seed: 42

pipeline_runner:
  batch_size: 200          # Need 168+ rows for weekly lag
  num_steps_inference: 500
  inference_interval_seconds: 2
```

**Important:** `batch_size` must be > 168 (to compute `load_lag_168`). Use 200.

---

## Dashboard Changes

Same Dash layout. Change:
- Y-axis label: `Load (MW)` instead of `Bike Count`
- Title: `Real-time Energy Load Forecast`
- Actuals column: `load_mw` instead of `cnt`

---

## Steps to Build

1. Download OPSD CSV → `data/01_raw/`
2. Fetch Open-Meteo weather → merge on datetime → save as parquet
3. Split into train / inference parquet files
4. Copy bike repo structure, rename package
5. Update `nodes.py` — same functions, just add `day_of_week`, `month`, `is_holiday` columns in feature eng
6. Update `parameters.yml` — lag params, model params, paths
7. Update `catalog.yml` — new file paths
8. Update dashboard labels
9. Run training → check MAE in MW
10. Run inference + dashboard → watch live predictions

---

## Expected Model Performance

CatBoost on hourly German load with lag + weather features:
- **MAE: ~150–400 MW** (depends on season, dataset size)
- **MAPE: ~2–5%** (state-of-the-art for this type of model)
- German load ranges ~30,000–80,000 MW so 300 MW error ≈ <1%

---

## Sources

- [Open Power System Data — Time Series](https://data.open-power-system-data.org/time_series/)
- [EIA Open Data API](https://www.eia.gov/opendata/)
- [ENTSO-E Transparency Platform](https://transparency.entsoe.eu/)
- [Open-Meteo Historical Weather API](https://open-meteo.com/en/docs/historical-weather-api)
- [Open Power System Data — About](https://open-power-system-data.org/)
