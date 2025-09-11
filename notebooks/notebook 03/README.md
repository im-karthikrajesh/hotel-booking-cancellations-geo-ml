# Notebook 3: Predictive Modelling, Geospatial & Cross-Country Analysis

**Project:** *Factors Driving Hotel Booking Cancellations: A Predictive and Geospatial Analysis*  
**Path:** `notebooks/notebook 03/Predictive_Modelling_Geospatial_and_Cross_Country_Analysis_Notebook.ipynb`

This notebook consumes the modelling‑ready dataset exported by **Notebook 2**, builds robust classification models, selects and calibrates a **champion**, explains predictions, and delivers geospatial & cross‑country insights.

> The list of source datasets and links is maintained in `DATA_SOURCES.md` (repo root).

---

## 1) Purpose & Scope

- Load the modelling‑ready table **`bk_full_cleaned.csv`** (from Notebook 2).  
- Enforce **target‑leakage controls** and derive temporal **cyclical encodings**.  
- Build a **ColumnTransformer** preprocessor (scaling + OHE + target encoding).  
- Train & tune (Optuna) **Logistic Regression, Random Forest, XGBoost, LightGBM, MLP**, plus a **Dummy** baseline.  
- Evaluate with **5‑fold stratified CV** and a **single 20% hold‑out** (ROC‑AUC, Accuracy, Precision, Recall, F1).  
- Pick a **champion** (by ROC‑AUC), **optimise threshold** for F1, assess **calibration** (sigmoid, isotonic), and persist artifacts.  
- Provide **explainability** (SHAP global/local; PDP/ICE) and **risk segmentation** (deciles, lift & capture curves).  
- Produce **geospatial** outputs (static choropleth, interactive Folium) and **spatial statistics** (Moran’s I & LISA).  
- Run **cross‑country driver** correlations with **BH‑FDR** control.

---

## 2) Inputs (expected files)

| Logical name | Default path in code | Recommended repo path | Notes |
|---|---|---|---|
| Cleaned bookings (modelling‑ready) | `/content/bk_full_cleaned.csv` | `data/processed/bk_full_cleaned.csv` | Output of Notebook 2. Update `DATA_PATH` if you place it elsewhere. |
| World boundaries (Natural Earth, 50m “admin_0 map_units”) | downloaded at runtime | – | Used for choropleths and LISA; fetched from official S3 URL by GeoPandas. |

Missing inputs will raise a **FileNotFoundError** (for local CSV) or a **download error** (for Natural Earth) at the point of access.

---

## 3) Target‑Leakage Controls

The following variables are **dropped** before modelling as they leak target information or are post‑outcome:

- `reservation_status`, `reservation_status_date`, `assigned_room_type`, `bookings_count`, `cancellation_rate`.

A standalone check confirms any such features would individually achieve suspiciously high AUC — hence removal.

---

## 4) Feature Matrix & Temporal Engineering

- `arrival_date` → extract **month** and **weekday**; add **cyclical encodings**:  
  - `arr_month_sin`, `arr_month_cos`, `arr_weekday_sin`, `arr_weekday_cos`.  
- Cast temporal categories (`arrival_year`) and selected engineered flags to appropriate dtypes.  
- Define **X** and **y** (`is_canceled`) from the vetted dataframe.

---

## 5) Split & Preprocessing Pipeline

- **Split:** Stratified 80/20 train/test with fixed seed (`random_state=42`).  
- **Preprocessor:** `ColumnTransformer` with three branches:  
  - Numeric → `StandardScaler`  
  - Low‑cardinality categorical (≤10 levels) → `OneHotEncoder(handle_unknown='ignore')`  
  - High‑cardinality categorical → `TargetEncoder(smoothing=0.1)`

Utilities ensure **train/test column alignment** before inference.

---

## 6) Models, Tuning & Evaluation Strategy

### Models
- **DummyClassifier** (stratified) — baseline.  
- **Logistic Regression** — tune penalty/solver/C/tol/intercept/class weights.  
- **Random Forest** — tune trees, depth, min split/leaf, max features, class weights.  
- **XGBoost** — tune depth, learning rate, subsample/colsample, gamma, min_child_weight, L1/L2; `tree_method='hist'`.  
- **LightGBM** — tune n_estimators, num_leaves, depth, subsample/colsample, L1/L2.  
- **MLPClassifier** — tune hidden layers/units, activation, solver, learning‑rate schedule, early stopping. Uses `RandomOverSampler` for class imbalance.

### Evaluation
- **5‑fold Stratified CV** (`random_state=42`), scoring on: ROC‑AUC, Accuracy, Precision, Recall, F1.  
- **Single hold‑out** results mirrored on the same metrics.  
- **ROC curves** plotted on the hold‑out for all trained pipelines.

### Imbalance handling
- **XGBoost:** `scale_pos_weight = (#neg / #pos)` computed on training set.  
- **LR, RF, LightGBM:** `class_weight='balanced'`.  
- **MLP:** `RandomOverSampler` inside the pipeline.

---

## 7) Results Registry (persist fitted pipelines + metrics)

All fitted models and their metric dicts are saved as **joblib** bundles for reuse/comparison:

- Directory in code: `model_results/` (overwritten if `REFRESH_RESULTS=True`).  
- Each file stores: `{'pipeline': fitted_pipeline, 'cv': {...}, 'holdout': {...}}`.

> You can point the registry directly to the repo’s `artifacts/model_results/` by editing `RESULTS_DIR` (see §11).

---

## 8) Champion Selection, Thresholding & Calibration

1. **Champion by ROC‑AUC** on hold‑out; then optimise **F1 threshold** via the PR curve.  
2. Persist **raw champion** (example: XGBoost):  
   - `champion_xgb.joblib` → `{'model': best_xgb, 'threshold': <float>}`.  
3. **Calibration check:** compare **Raw / Sigmoid / Isotonic** (`CalibratedClassifierCV`) by **ECE**, **Brier**, and ROC‑AUC; flag over‑fit if train‑ECE ≪ hold‑out‑ECE.  
4. The **final calibrated winner** is stored in‑memory as `final_artifact = {'model': ..., 'threshold': ...}` for downstream risk & geo analysis.

> Optional: persist the calibrated champion as `champion_final.joblib` for deployment.

---

## 9) Explainability & Risk Segmentation

- **SHAP** (TreeExplainer): global **Top‑K mean |SHAP|**, **beeswarm**, **heatmap**; **waterfall** for the highest‑risk instance.  
- **PDP/ICE** for selected numeric drivers; **category‑effect** bars for categoricals.  
- **Risk buckets:** hold‑out probabilities → equal‑sized **deciles**; report event rates, **lift**, and **cumulative capture**.

Figures render inline; save to disk if needed (examples in §12 table).

---

## 10) Geospatial & Cross‑Country Analysis

- Build a **per‑country** summary on the hold‑out: bookings, observed cancellation rate, **Bayesian‑shrunk** rate, mean predicted risk, lift, booking share.  
- **Static choropleth** (GeoPandas) and **interactive Folium** map of predicted risk by **origin country**.  
- **Rankings:** Top/Bottom origins by predicted risk (with minimum‑n stability) and **top contributors** to predicted positives.  
- **Trends:** risk trend over years for top‑volume origins.  
- **Cross‑country drivers:** compute **Spearman** and **bookings‑weighted Pearson** correlations vs predicted risk; control **FDR** with Benjamini–Hochberg (q≤0.10).  
- **Spatial autocorrelation:** Global **Moran’s I** and **LISA** clusters using **KNN (k=5)** and **9,999 permutations**.

---

## 11) Directory Constants & Feature Toggles (optional)

Set these near the top of the notebook to align outputs with your repo layout:

```python
# Inputs
DATA_PATH   = 'data/processed/bk_full_cleaned.csv'

# Output roots
from pathlib import Path
RESULTS_DIR = Path('artifacts/model_results')  # instead of 'model_results/'
OUT_DIR_FIG = 'reports/figures/geospatial'     # instead of 'figs_geo/'
OUT_DIR_MAP = 'reports/maps'                   # instead of 'maps_geo/'
```

Feature toggles:

```python
REFRESH_RESULTS = True  # overwrite model_results/*.joblib after training
FORCE_RERUN     = True  # recompute LISA plots if guards are set
```

---

## 12) Outputs

> File(s) produced by the notebook.

| Generated file(s) | Produced by section | Default path in code |
|---|---|---|
| `model_results/*.joblib` (per‑model dict with `pipeline`, `cv`, `holdout`) | Results Registry | `model_results/` |
| `logistic_regression_results.joblib` | Results Registry | `model_results/logistic_regression_results.joblib` |
| `random_forest_results.joblib` | Results Registry | `model_results/random_forest_results.joblib` |
| `xgboost_results.joblib` | Results Registry | `model_results/xgboost_results.joblib` |
| `lightgbm_results.joblib` | Results Registry | `model_results/lightgbm_results.joblib` |
| `mlp_classifier_results.joblib` | Results Registry | `model_results/mlp_classifier_results.joblib` |
| `dummy_classifier_results.joblib` | Results Registry | `model_results/dummy_classifier_results.joblib` |
| **Champion (raw)** `champion_xgb.joblib` (`{'model','threshold'}`) | Champion Selection | `./champion_xgb.joblib` |
| (Optional) **Champion (calibrated)** `champion_final.joblib` | Calibration Check | _not written by default_ |
| `correlation_matrix.png` | Correlation Checks | `./correlation_matrix.png` |
| `Feature_summary.png` (fallback `feature_summary.html`) | Feature Table | `./Feature_summary.png` |
| `figs_geo/choropleth_pred_mean.png` | Geospatial – static map | `figs_geo/` |
| `maps_geo/pred_risk_by_origin.html` | Geospatial – interactive | `maps_geo/` |
| `figs_geo/risk_trend_top_origins.png` | Risk trends | `figs_geo/` |
| `figs_geo/bivariate_risk_<driver>.png` (≤3 drivers) | Bivariate maps | `figs_geo/` |
| `figs_geo/lisa_clusters_pred.png` | LISA (predicted risk) | `figs_geo/` |
| `figs_geo/lisa_clusters_obs.png` (if created) | LISA (observed shrunk) | `figs_geo/` |

**Inline (not saved by default):** ROC curves, reliability diagrams, SHAP beeswarm/heatmap/waterfall, PDP/ICE grids, risk‑bucket charts. Add explicit `plt.savefig(...)` calls if you want files.

---

## 13) How to Run

### Google Colab
1. Open the notebook.  
2. Ensure **`bk_full_cleaned.csv`** is available (upload or mount Drive).  
3. **Runtime ▸ Run all**.  
4. Download artifacts if needed (files appear in the Colab working directory).

### Local Jupyter / VS Code
1. Python ≥ 3.9 with dependencies (see §14).  
2. Place `bk_full_cleaned.csv` at `data/processed/` or update `DATA_PATH`.  
3. Run cells in order through: **EDA → Leakage → Features → Split & Preprocess → Model blocks (LR/RF/XGB/LGBM/MLP) → Results Registry → Champion → Calibration → Risk & Explainability → Geospatial & Cross‑country**.  
4. Verify artifacts in the folders listed in §12.

---

## 14) Dependencies

Install into a fresh environment (or Colab):

```bash
pip install optuna category_encoders shap xgboost lightgbm imbalanced-learn \
            geopandas folium mapclassify dataframe-image libpysal esda \
            numpy pandas scikit-learn matplotlib seaborn joblib \
            pyproj fiona shapely rtree
```

> **Geo stack note:** `geopandas` and friends may require GDAL/GEOS/PROJ system libraries. Colab usually works out of the box.

---

## 15) Reproducibility & Notes

- Global seed: `random_state=42` across splits and models.  
- CV: **StratifiedKFold(n_splits=5, shuffle=True, random_state=42)**.  
- XGBoost imbalance via **`scale_pos_weight = neg/pos`** computed on the training split.  
- **BH‑FDR** for cross‑country tests at **q ≤ 0.10**.  
- **Moran/LISA:** **KNN k=5**, **9,999 permutations**.  
- **Runtime considerations:** Optuna tuning & spatial statistics can be compute‑intensive; lower `n_trials` or `timeout` if needed.

---

## 16) Caveats

- **Do not re‑introduce** leaky fields (`reservation_status*`, etc.) into the modelling matrix.  
- The final **calibrated** champion is in memory by default — **persist it** if you plan to deploy.  
- Natural Earth data is fetched live, requires internet access during geospatial steps.
