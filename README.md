# Satellite-derived crop-stress detection and early warning, SW Nigeria

Can satellite observations and climate data identify emerging crop stress early enough to help prioritise agricultural monitoring? This project builds and tests a workflow for Oyo, Osun and northern Ogun states (Nigeria), using Google Earth Engine and Python in JupyterLab.

**There are no ground labels or yield records.** Stress is a proxy (an unusually low MODIS NDVI for the pixel and time of year), so results are *proxy-validated*, not validated yield forecasts. Predictors come from sources other than the optical NDVI used for the label.

**[Live interactive map](https://abdulquawiyy-owolabi/sw-nigeria-crop-stress/hotspot.html)**: predicted risk vs. the observed stress proxy for every 16-day window, 2016 to 2025. Click any cell for its time series and local skill. The page is `index.html` in this repository.

Full write-up: see the research report in `docs/`.

## Headline result

- Persistence of the vegetation state is the dominant predictor.
- Climate (CHIRPS, ERA5-Land, MODIS land surface temperature) and Sentinel-1 radar improve **same-window detection (0 days)**: the best model beats persistence by +0.24 PR-AUC lift (95% CI +0.14 to +0.37), and by +0.42 for emerging stress, in 9 of 10 test years.
- No gain is demonstrated at 16 days or beyond. This is a detection system, not a long-lead forecast.

## Repository layout

The analysis lives in Jupyter notebooks, one per step. The number is the run order.

| Notebook | Purpose |
|---|---|
| `aoi_mask_label.ipynb` | Study area, cropland mask, stratified sample points, MODIS NDVI export |
| `build_baseline.ipynb` | Per-pixel seasonal baseline, z-scores, stress flags (leave-one-year-out) |
| `extract_climate.ipynb` | CHIRPS, ERA5-Land and MODIS land surface temperature per 16-day window |
| `extract_s1.ipynb` | Sentinel-1 backscatter per window (`--diagnose` shows what the area holds) |
| `features_and_lags.ipynb` | Climate anomalies, SPI, lag analysis |
| `s1_features.ipynb` | Track-consistent Sentinel-1 features (dB, cross ratio, RVI, anomalies) |
| `models.ipynb` | Baselines and XGBoost / Random Forest with year-by-space validation |
| `analysis_maps.ipynb` | Bootstrap by year, breakdowns, skill maps, interactive map, SHAP |
| `data_provenance.csv` | Every dataset: Earth Engine ID, version, variables, processing, script |

Outputs from the final steps (figures, result tables, `index.html`) are in `docs/`. Large intermediate data (CSVs, parquet) is not committed.

## Requirements

- JupyterLab with Python 3.10+.
- A Google Earth Engine account and a Cloud project registered for Earth Engine. The free noncommercial tiers have a monthly compute quota; the extraction steps are resumable.
- Packages: `earthengine-api`, `pandas`, `numpy`, `scipy`, `scikit-learn`, `xgboost`, `matplotlib`, `pyarrow` (optional, falls back to CSV), `shap` (optional, for SHAP).

## How to run

Each notebook has a cell that defines the code, then a cell that calls `main([...])` with the options for that step. For every notebook:

1. Run the code-definition cell (it only defines functions).
2. Run the `main([...])` cell below.
3. **Restart the kernel before opening the next notebook.** The notebooks reuse function names (`main`, `run`, `plot`), so running two in one session makes the second overwrite the first.

Edit `POINTS_ASSET` in `00`, `02`, and `02b` to your own Earth Engine asset path first. Paths below are placeholders.

```python
# 00: sample points, then MODIS NDVI (one CSV per year)
main(["points", "--project", "GEE_PROJECT_NAME"])
main(["ndvi", "--project", "GEE_PROJECT_NAME", "--local-dir", "data/raw"])

# 01: label (baseline, z-scores, stress flags)
main(["--raw-dir", "data/raw", "--out-dir", "data/label"])

# 02 and 03: climate predictors
main(["--project", "GEE_PROJECT_NAME", "--local-dir", "data/climate"])
main(["--clim-dir", "data/climate", "--label-dir", "data/label", "--out-dir", "data/features"])

# 02b and 03b: Sentinel-1 (check coverage first; the ascending pass covers this area)
main(["--project", "GEE_PROJECT_NAME", "--diagnose", "--years", "2020"])
main(["--project", "GEE_PROJECT_NAME", "--local-dir", "data/s1", "--pid-file", "data/keep_pids.csv"])
main(["--s1-dir", "data/s1", "--feat-dir", "data/features", "--out-dir", "data/features",
      "--label-dir", "data/label"])

# 04: models (train on all years, test on 2016-2025)
main(["--feat-dir", "data/features", "--feat-file", "features_long_s1", "--label-dir", "data/label",
      "--out-dir", "data/models", "--sets", "ndvi", "climate+ndvi", "ndvi+s1", "climate+ndvi+s1",
      "--models", "xgb", "--eval-min-year", "2016", "--year-groups", "5"])

# 05: analysis, maps, SHAP (point --models-script at the 04 notebook; .ipynb files are supported)
main(["--feat-dir", "data/features", "--feat-file", "features_long_s1", "--label-dir", "data/label",
      "--model-dir", "data/models", "--out-dir", "data/analysis", "--model", "xgb",
      "--eval-min-year", "2016", "--map-horizon", "0",
      "--steps", "boot", "breakdown", "maps", "shap",
      "--boot-refs", "pers", "ndvi|xgb", "climate+ndvi|xgb",
      "--models-script", "04_models.ipynb"])
```

`keep_pids.csv` lists the points kept after the data-quality filter (those with at least 70% valid windows; see `point_valid_fraction.csv` from step 01).

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

## Notes for committing notebooks

Clear outputs before you commit (Kernel, Restart Kernel and Clear Outputs of All Cells), so the repository stays small, and no local paths or project names leak into the saved output. GitHub renders `.ipynb` files directly.

## Licence and citation

MIT License
