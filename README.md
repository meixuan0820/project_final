# Detecting Satellite Manoeuvres from TLE via Forecasting Residuals

One-liner: **Detect satellite manoeuvres using time-series forecasting and residual-based threshold search on TLE orbital data.**

---

## Repository structure

```
.
├─ satellit_data/                          # Raw inputs (TLEs, manoeuvre logs)  ← raw data lives here
├─ EDA.ipynb                               # Clean raw data, build GT timestamps (median), plots
├─ pipeline/
│  ├─ project_a.R/                         # ARIMA workflow (R) + residual extraction helpers
│  ├─ baseline.Rmd/                        # ARIMA workflow (R) + residual extraction for ARIMA(1,1,1)
│  ├─ SARAL/                               # Per-satellite notebooks (variants, tuning, residuals, saves)
│  ├─ S3A/                                 #    "
│  ├─ FY2D/                                #    "
│  └─ End to end/                          # Extended LSTM "E2E" study (train/val/test, hyperparam search)
├─ evaluation/                             # Precision–Recall comparisons for the four experiments
│  ├─ xgb diff vs no diff.ipynb/           # XGB experiments set PR curves, elbow points
│  ├─ lstm vs autoencoder lstm.ipynb/      # LSTM experiments PR curves, elbow points
│  ├─ 4 sats comparison.ipynb/             # baseline, XGB diff, LSTM and tuned ARIMA comparsion - PR curves, elbow points
│  ├─ window size.ipynb/                   # different window size PR curves, elbow points
│  └─ quantative calculation.ipynb/        # Summary tables (best F1/F2 per model/window)
├─ result/
│  ├─ pr_model.csv/                        # Saved PR detail CSVs per satellite/model/window/threshold
│  ├─ sat_model_residuals.csv/             # extracted residuals by model-sat combination
│  └─ metrics_model.csv/                   # evaluation results
└─ README.md
```

> Notes  
> • EDA saves two pickles used everywhere else: **`project_a/orbitals.pkl`** and **`project_a/maneuvers.pkl`**.

---

## Pipeline at a glance

1) **Raw → Clean (EDA.ipynb)**  
   - Load TLE + manoeuvre logs from `satellit_data/`.  
   - Standardise timestamps; generate **median manoeuvre time** per event for alignment/plots.  
   - Produce exploratory plots of orbital elements aligned to ground-truth manoeuvres.  
   - Save cleaned artifacts: `.../orbitals.pkl`, `.../maneuvers.pkl`.

2) **Modeling & Residual Extraction (pipeline/)**  
   - Per-satellite notebooks under `pipeline/SARAL`, `pipeline/S3A`, `pipeline/FY2D`.  
   - Implement ARIMA (R via `project_a.R`), XGBoost variants, and LSTM forecaster.  
   - Tune (where applicable), predict next step, **compute residuals**, and persist outputs.

3) **End-to-End LSTM study (end_to_end/)**  
   - Full train/val/test split with hyperparameter search.  
   - Targets **highest F1 under a 3-day max-matching window**; exports best config & residuals.

4) **Evaluation & Threshold Search (evaluation/, quantitative_calculation.ipynb)**  
   - Generate **Precision–Recall curves** across **1-, 3-, 5-day** matching windows.  
   - Compute **best F1** and **best F2** thresholds per model×satellite×window.  
   - Save PR detail CSVs, summary tables, and figures in `evaluation/` and `result/`.

---

## Quickstart

### 1) Environment

**Python (3.9–3.11):**
```bash
# from repo root
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt   # if missing, install: numpy, pandas, scipy, matplotlib, scikit-learn, xgboost, statsmodels, jupyter
```

**R (≥4.2) for ARIMA workflow in `project_a.R`:**
```r
install.packages(c("forecast","tseries","ggplot2","readr","dplyr"))
```

### 2) Run the pipeline

**A. EDA & cleaning**
- Open **`EDA.ipynb`** and run all cells.  
- Outputs: `project_final/orbitals.pkl`, `project_final/maneuvers.pkl`, plus exploratory plots.

**B. Per-satellite modeling**
- Enter **`pipeline/<SAT>/`** (`SARAL`, `S3A`, `FY2D`).
- Run the notebooks to:
  - fit models,
  - produce predictions & residuals,
  - save intermediate CSV/PKL to `result/`.

**C. End-to-end LSTM (optional extension)**
- Open notebooks in **`end_to_end/`**.  
- Train with train/val/test, record the best model by F1@3d.  
- Export residuals & flags to `result/`.

**D. Evaluation & metrics**
- Open **`quantitative_calculation.ipynb`**  
  (file name used in repo may be `qutativ calculation.ipynb`; both refer to the same step).
- Run all cells to regenerate:
  - PR detail (`precision`, `recall`, `threshold`),
  - best **F1** and **F2** per model/window,
  - figures and tables under `evaluation/` and `result/`.

---

## What the main files do

- **`EDA.ipynb`** — Cleans raw data, harmonises time indices, computes median manoeuvre timestamps, and saves the two PKLs used downstream.  
- **`project_a.R`** — R environment for **ARIMA**: exports training data, fits models, and extracts residuals.  
- **`pipeline/*/*.ipynb`** — Implements/tunes **ARIMA/XGBoost/LSTM** variants per satellite; outputs residuals & saved artifacts.  
- **`end_to_end/*`** — Extended LSTM study with proper splits and hyperparam search; reports best config.  
- **`quantitative_calculation.ipynb`** — Builds **PR curves** and selects **best F1/F2 thresholds** for each model×satellite×window (1/3/5 days).

---

## Data notes

- **Source**: Internal archive provided by **Dr.David Shorten** (course supervisor).  
- **Ground truth**: Manoeuvre logs are used for evaluation only (matching windows of 1, 3, 5 days).  
- **Pickles**: `orbitals.pkl` (cleaned TLE/element time series) and `maneuvers.pkl` (processed manoeuvre timestamps).

---

## Attribution

- The function family named like **`compute_simple_matching_precision_recall_for_one_threshold`** used throughout this project is attributed to **Dr.Liang Zhao** (public GitHub).  
- Please retain this attribution in derivative work.

---

## Reproducibility & conventions

- **Windows**: matching windows used are **1, 3, 5 days**.  
- **Metrics**: PR curves with **best F1** (balanced) and **best F2** (recall-leaning) reported.  
- **Seeds**: set per notebook where supported (XGBoost/LSTM).  
- **Outputs**: CSVs with columns  
  `model, satellite, window_days, threshold, precision, recall [, f1, f2]`.

---

## Troubleshooting

- **“File not found: orbitals.pkl / maneuvers.pkl”**  
  Run **`EDA.ipynb`** first; confirm they were written to `project_a/`.
- **ARIMA flat forecasts / seasonality issues**  
  Check differencing settings in `project_a.R` and confirm series stationarity before fitting.
- **Mismatched folder name**  
  If your repo uses `satellite_data/`, search & replace references to `satellit_date/`.
