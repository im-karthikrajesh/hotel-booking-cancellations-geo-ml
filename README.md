# hotel-booking-cancellations-geo-ml

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](#)
[![scikit-learn](https://img.shields.io/badge/ML-scikit--learn-F7931E?logo=scikitlearn&logoColor=white)](#)
[![Geo stack](https://img.shields.io/badge/Geo-GeoPandas%20%7C%20Folium-0E4B7F)](#)
[![Optuna](https://img.shields.io/badge/Tuning-Optuna-9cf)](#)
[![Colab Ready](https://img.shields.io/badge/Platform-Google%20Colab-F9AB00?logo=googlecolab&logoColor=white)](#)

A calibrated ML + geospatial pipeline that **predicts hotel booking cancellations** and explains **cross-country drivers** using World Bank's World Development Indicators(WDI) and Hofstede's cultural indicators.

> This repository supports my MSc Business Analytics Project *“Factors Driving Hotel Booking Cancellations: A Predictive and Geospatial Analysis.”*

---

## TL;DR — Highlights

- **Champion model:** isotonic-calibrated **XGBoost**  
  ROC-AUC **0.9559**, F1 **0.8454** at threshold **0.349**, Accuracy **0.88**, ECE **0.0052** (hold-out).  
- **Lift & stratification:** ~**52%** of all cancellations captured in the **top 20%** of scores; ~**89%** by **40%**; ~**99%** by **60%**.  
- **External context matters:** Group permutation indicates complementary signal from **country + WDI + Hofstede** blocks (beyond labels alone).  
- **Spatial structure:** Predicted risk shows significant Moran’s I; LISA maps identify HH/LL foci consistent with macro context.

---

## Repository Layout

```text
hotel-booking-cancellations-geo-ml/
├─ artifacts/
│  ├─ champion/
│  │  └─ champion_xgb.joblib
│  └─ model_results/
│     ├─ dummy_classifier_results.joblib
│     ├─ lightgbm_results.joblib
│     ├─ logistic_regression_results.joblib
│     ├─ mlp_classifier_results.joblib
│     ├─ random_forest_results.joblib
│     └─ xgboost_results.joblib
├─ notebooks/
│  ├─ notebook_01/
│  │  ├─ External_Data_Preparation_Notebook.ipynb
│  │  └─ README.md
│  ├─ notebook_02/
│  │  ├─ Comprehensive_Data_Preparation_and_Feature_Engineering_Workflow_Notebook.ipynb
│  │  └─ README.md
│  └─ notebook_03/
│     ├─ Predictive_Modelling_and_Geospatial_Analysis_Notebook.ipynb
│     └─ README.md
├─ reports/
│  ├─ figures/
│  │  ├─ bivariate_risk_inflation_pct.png
│  │  ├─ bivariate_risk_ivr.png
│  │  ├─ bivariate_risk_tourism_departures_per_1000.png
│  │  ├─ choropleth_pred_mean.png
│  │  ├─ correlation_matrix.png
│  │  ├─ cross_country_driver_scatter.png
│  │  ├─ Feature_summary.png
│  │  ├─ lisa_clusters_obs.png
│  │  ├─ lisa_clusters_pred.png
│  │  └─ risk_trend_top_origins.png
│  └─ maps/
│     └─ pred_risk_by_origin.html
├─ DATA_SOURCES.md
└─ README.md
```

- Each notebook folder includes a **focused README** with exact inputs, outputs, and run steps.

---

## What this project does

1) **Creates country-year context**  
   Filters WDI (2015–2017) and cleans Hofstede dimensions; standardises ISO-3, pivots WDI to wide form, and adds derived indicators (tourism per-capita, digital penetration, etc.).  
2) **Builds a modelling-ready table**  
   Cleans and engineers booking-level features; joins country-year context; applies **multi-stage imputation** (within-country interpolation → region/year medians → global fallback).  
3) **Trains & selects a champion**  
   Pipelines for LR / RF / XGBoost / LightGBM / MLP with **5-fold stratified CV**, single **20% hold-out**, imbalance handling, and **probability calibration** (sigmoid vs isotonic).  
4) **Explains predictions & maps risk**  
   SHAP (global/local), PDP/ICE, risk deciles & lift/capture. **GeoPandas + Folium** render global risk choropleths; **Moran’s I & LISA** quantify spatial autocorrelation and clusters.

---

## Notebooks

- **Notebook 1 — External Data Preparation: WDI & Hofstede**  
  Curates indicators, harmonises ISO-3, exports `wdi_long.csv`, `wdi_filtered.csv`, `hofstede.csv`. See [`notebooks/notebook_01/README.md`](notebooks/notebook_01/README.md).

- **Notebook 2 — Comprehensive Data Prep & Feature Engineering**  
  Cleans bookings, engineers features, integrates externals, imputes, and exports **`bk_full_cleaned.csv`**. See [`notebooks/notebook_02/README.md`](notebooks/notebook_02/README.md).

- **Notebook 3 — Predictive Modelling, Geospatial & Cross-Country Analysis**  
  Full training/eval, calibration & explainability, maps, spatial stats, **results registry** + **champion artifact**. See [`notebooks/notebook_03/README.md`](notebooks/notebook_03/README.md).

---

## Getting started

### Option A — Google Colab (fastest)
Open each notebook and **Run all**. When prompted, upload inputs or mount Drive. Outputs (artifacts/figures/maps) are written to the working directory structure shown above.

### Option B — Local environment
```bash
# Create environment and install dependencies
pip install -U numpy pandas scikit-learn xgboost lightgbm optuna   imbalanced-learn shap category-encoders   geopandas folium mapclassify libpysal esda   matplotlib seaborn joblib pyproj fiona shapely rtree
```
> **Geo stack note:** `geopandas` and friends may require GDAL/GEOS/PROJ system libraries.

**Data paths**  
- Primary modelling input: `data/processed/bk_full_cleaned.csv` (produced by Notebook 2).  
- Natural Earth boundaries are fetched at runtime by GeoPandas in Notebook 3.

---

## Artifacts & figures

- **Model registry:** `artifacts/model_results/*.joblib` (fitted pipeline + CV/hold-out metrics).  
- **Champion:** `artifacts/champion/champion_xgb.joblib` (calibrated model + threshold).  
- **Figures:** correlation heatmaps, ROC, reliability, SHAP, PDP/ICE, risk deciles & capture, LISA cluster plots.  
- **Maps:** `reports/maps/pred_risk_by_origin.html` (interactive Folium).

Selected results: **ROC-AUC 0.9559**, **F1 0.8454 @ 0.349**, **ECE 0.0052**, strong decile capture; significant, interpretable geo-patterns validated via **Moran’s I / LISA**.

---

## Reproducibility & governance

- Fixed seeds (`random_state=42`) and stratified splits across models.  
- Consistent pipeline idioms (ColumnTransformer, fit/transform discipline).  
- Results tracked to versioned artifacts and figures listed above.  
- See `DATA_SOURCES.md` for data licensing and links.

---

## Scope & research questions

1) **Which booking traits and external indicators** best predict cancellation?  
2) **How much do macro-demographics & culture** improve predictive accuracy and decision value?  
3) **What spatial patterns** exist in cancellation behaviour across origin countries?

---

## Acknowledgements

- **Primary dataset:** Hotel Booking Demand (see `DATA_SOURCES.md` for links/terms).  
- **External context:** World Development Indicators (World Bank) and Hofstede cultural dimensions.  
- Project executed as part of **Data-Driven Dissertation** (University of Nottingham, MSc Business Analytics).

---

**Questions or ideas?** Open an issue with a minimal reproducible example and the notebook section you’re referencing.
