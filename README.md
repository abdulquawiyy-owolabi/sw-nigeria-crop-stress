# Satellite-derived crop-stress detection and early warning, SW Nigeria

Can satellite observations and climate data identify emerging crop stress early enough to help prioritise agricultural monitoring? This project builds and tests a workflow for Oyo, Osun and northern Ogun states (Nigeria), using Google Earth Engine and Python.

**There are no ground labels or yield records.** Stress is a proxy (an unusually low MODIS NDVI for the pixel and time of year), so results are *proxy-validated*, not validated yield forecasts. Predictors come from sources other than the optical NDVI used for the label.

## Headline result

- Persistence of the vegetation state is the dominant predictor.
- Climate (CHIRPS, ERA5-Land, MODIS land surface temperature) and Sentinel-1 radar improve **same-window detection (0 days)**: the best model beats persistence by +0.24 PR-AUC lift (95% CI +0.14 to +0.37), and by +0.42 for emerging stress, in 9 of 10 test years.
- No gain is demonstrated at 16 days or beyond. This is a detection system, not a long-lead forecast.

**[Live interactive map](https://abdulquawiyy-owolabi.github.io/sw-nigeria-crop-stress/)**: predicted risk vs. the observed stress proxy for every 16-day window, 2016 to 2025. Click any cell for its time series and local skill. The page is `index.html` in this repository.

Full write-up: see the project report in `docs/`.

## Repository layout

| Script | Purpose |
|---|---|
| `00_aoi_mask_label.py` | Study area, cropland mask, stratified sample points, MODIS NDVI export |
| `01_build_baseline.py` | Per-pixel seasonal baseline, z-scores, stress flags (leave-one-year-out) |
| `02_extract_climate.py` | CHIRPS, ERA5-Land and MODIS land surface temperature per 16-day window |
| `02b_extract_s1.py` | Sentinel-1 backscatter per window (`--diagnose` shows what the area holds) |
| `03_features_and_lags.py` | Climate anomalies, SPI, lag analysis |
| `03b_s1_features.py` | Track-consistent Sentinel-1 features (dB, cross ratio, RVI, anomalies) |
| `04_models.py` | Baselines and XGBoost / Random Forest with year-by-space validation |
| `05_analysis_maps.py` | Bootstrap by year, breakdowns, skill maps, interactive map, SHAP |
| `data_provenance.csv` | Every dataset: Earth Engine ID, version, variables, processing, script |

## Requirements

- A Google Earth Engine account and a Cloud project registered for Earth Engine. The free noncommercial tiers have a monthly compute quota; extraction scripts are resumable.
- Python 3.10+ with: `earthengine-api`, `pandas`, `numpy`, `scipy`, `scikit-learn`, `xgboost`, `matplotlib`, `pyarrow` (optional, falls back to CSV), `shap` (optional, for SHAP).

## How to run

Each script has a `main([...])` function, designed to be called from Jupyter. Because the scripts share function names (`main`, `run`, `plot`), load each from its own file so they do not overwrite each other:

```python
import importlib.util
def load(path, name):
    spec = importlib.util.spec_from_file_location(name, path)
    mod = importlib.util.module_from_spec(spec); spec.loader.exec_module(mod)
    return mod
m00 = load("00_aoi_mask_label.py", "m00")   # and so on for the others
```

Edit `POINTS_ASSET` in `00`, `02`, `02b` to your own Earth Engine asset path first. Paths below are placeholders.

```python
# 1. Sample points, then MODIS NDVI (one CSV per year)
m00.main(["points", "--project", "YOUR_PROJECT"])
m00.main(["ndvi", "--project", "YOUR_PROJECT", "--local-dir", "data/raw"])

# 2. Label: baseline, z-scores, stress flags
m01.main(["--raw-dir", "data/raw", "--out-dir", "data/label"])

# 3. Climate predictors
m02.main(["--project", "YOUR_PROJECT", "--local-dir", "data/climate"])
m03.main(["--clim-dir", "data/climate", "--label-dir", "data/label", "--out-dir", "data/features"])

# 4. Sentinel-1 (check coverage first; the ascending pass is the one that covers this area)
m02b.main(["--project", "YOUR_PROJECT", "--diagnose", "--years", "2020"])
m02b.main(["--project", "YOUR_PROJECT", "--local-dir", "data/s1", "--pid-file", "data/keep_pids.csv"])
m03b.main(["--s1-dir", "data/s1", "--feat-dir", "data/features", "--out-dir", "data/features",
           "--label-dir", "data/label"])

# 5. Models: train on all years, test on 2016-2025
m04.main(["--feat-dir", "data/features", "--feat-file", "features_long_s1", "--label-dir", "data/label",
          "--out-dir", "data/models", "--sets", "ndvi", "climate+ndvi", "ndvi+s1", "climate+ndvi+s1",
          "--models", "xgb", "--eval-min-year", "2016", "--year-groups", "5"])

# 6. Analysis, maps, SHAP
m05.main(["--feat-dir", "data/features", "--feat-file", "features_long_s1", "--label-dir", "data/label",
          "--model-dir", "data/models", "--out-dir", "data/analysis", "--model", "xgb",
          "--eval-min-year", "2016", "--map-horizon", "0",
          "--steps", "boot", "breakdown", "maps", "shap", "--boot-refs", "pers", "ndvi|xgb", "climate+ndvi|xgb",
          "--models-script", "04_models.py"])
```

`keep_pids.csv` lists the points kept after the data-quality filter (those with at least 70% valid windows; see `point_valid_fraction.csv` from step 2).

## Methods in brief

- **Label:** NDVI z-score against the same 16-day window in 2001 to 2020, leave-one-year-out; stressed if z < -1.
- **Features:** anomalies of CHIRPS rainfall and SPI, ERA5-Land soil moisture, temperature and vapour pressure deficit, land surface temperature, and single-track Sentinel-1 VV, VH, cross ratio and RVI, at the issue time and the two windows before it.
- **Validation:** five year groups crossed with five spatial blocks and a 0.15 degree buffer; metrics per fold; paired bootstrap with years as the independent unit.
- **Baselines:** climatology, an SPI rule, persistence.

## Known limitations

- The label is a vegetation anomaly, not crop damage.
- Many points share one climate pixel, so the effective sample is closer to the number of cells than the number of points.
- About a fifth of points fall in the overlap of two Sentinel-1 tracks and have no radar data.
- Radar comparisons rest on 10 test years and about 70 comparisons; only the 0-day results are safe to lean on.

## Licence and citation

MIT
