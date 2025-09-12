# Notebook 2: Comprehensive Data Preparation & Feature Engineering

**Project:** *Factors Driving Hotel Booking Cancellations: A Predictive and Geospatial Analysis*  
**Path:** `notebooks/notebook 02/Comprehensive_Data_Preparation_and_Feature_Engineering_Workflow_Notebook.ipynb`

This notebook takes the raw booking data and the external context prepared in **Notebook 1**, performs rigorous **cleaning, harmonisation, feature engineering, imputation, and outlier handling**, and exports a single, modelling‑ready table.

> The list of source datasets and links is maintained in `DATA_SOURCES.md` (repo root).

---

## 1) Purpose & Scope

- Enforce **ISO‑3** country codes across all tables (bookings, WDI, Hofstede).  
- Engineer booking‑level features and **country‑year aggregates**.  
- Prepare WDI in **wide** country–year form with human‑readable names.  
- Clean Hofstede dimensions; run **MICE‑style** imputation (scikit‑learn `IterativeImputer` with BayesianRidge), with **region** and **global** fallbacks.  
- Multi‑stage **WDI imputation** (per‑country interpolation → region‑year median → year median → global median).  
- Create **derived ratios/indices** (digital penetration, tourism per‑capita, etc.).  
- Apply **outlier handling** (IQR winsorisation with skew logic; 99th‑percentile caps) and **transformations** (log1p, cube‑root).  
- Export the canonical dataset for modelling: **`bk_full_cleaned.csv`**.

---

## 2) Inputs (expected files)

| Logical name | Default path | Notes |
|---|---|---|
| Bookings | `hotel_bookings.csv` | Parses `reservation_status_date`; uses arrival date parts to build a single `arrival_date`. |
| WDI (long) | `wdi_long.csv` | Output from Notebook 1. Columns: `Country Name`, `Country Code`, `Indicator Name`, `Indicator Code`, `Year`, `Value`. |
| Hofstede | `hofstede.csv` | Output from Notebook 1 (cleaned). Columns typically include `country_code`, `country_name`, six dimension scores. |
| UN regions | `UNSD.csv` | **Semicolon** delimited; needs columns `ISO-alpha3 Code`, `Region Name` for region mapping during imputation. |

Missing files raise a **FileNotFoundError** with the resolved path.

---

## 3) Country Code Harmonisation (ISO‑3)

- **Bookings:** Convert ISO‑2 → ISO‑3 using `pycountry`. Special case `TMP` → `TLS`.  
- **Hofstede:** Fuzzy‑match non‑standard codes to official country names (`rapidfuzz`), then apply a **manual mapping** for edge cases (e.g., `BEF→BEL`, `SWF→CHE`).  
  - Drop sub‑national/language splits (e.g., `BEF`, `SWF`, `SWG`, `GEE`).  
  - Merge duplicates by **averaging numeric** and taking **first** for non‑numeric.  
- **WDI:** Remove non‑country aggregates, keep only valid ISO‑3.  
- Final **validation** confirms there are no invalid codes remaining.

---

## 4) Booking‑Level Cleaning & Feature Engineering

- Build `arrival_date` and `arrival_year`, drop original date parts.  
- Impute/cast: `children→0 (uint8)`, `country→'UNK' (category)`, `agent→'NONE' (category)`. Drop `company`.  
- Cast categoricals: `hotel, meal, market_segment, distribution_channel, reserved_room_type, assigned_room_type, deposit_type, customer_type, reservation_status`.  
- Core engineered features:
  - `total_nights = weekend + week_nights` (then drop components)  
  - `total_guests = adults + children + babies`  
  - `cancel_lag = arrival_date - reservation_status_date` (days)  
  - `lead_time_bin` (0‑7, 8‑30, 31‑90, 91‑180, >180)  
  - `season` (Winter/Spring/Summer/Autumn)  
  - Flags: `is_long_stay`, `flag_repeated_guest`, `flag_special_requests`, `flag_deposit_taken`, `flag_country_unknown`, plus outlier flags (`flag_lead_time_outlier`, `flag_cancel_lag_outlier`, `flag_adr_outlier`, `flag_negative_adr`).  
- **EDA visuals**: numeric distributions, cancellation rate by key categories, ADR vs cancellation, correlation heatmap.  
- **Country–year aggregates:** `bookings_count`, `cancellation_rate`.

---

## 5) WDI Preparation (long → wide, renamed)

- Rename columns to canonical: `country_name, country, indicator_name, indicator_code, year, value`.  
- Pivot to **wide** (`country, year` index) and cast `year` → `int`.  
- Rename 13 indicators to concise names:

| Code | New name |
|---|---|
| EN.POP.DNST | `population_density` |
| SP.POP.TOTL | `population_total` |
| SP.POP.1564.TO.ZS | `pop_15_64_pct` |
| SP.POP.65UP.TO.ZS | `pop_65_plus_pct` |
| SP.URB.TOTL.IN.ZS | `urban_pop_pct` |
| NY.GDP.PCAP.CD | `gdp_per_capita_usd` |
| NY.GDP.MKTP.KD.ZG | `gdp_growth_pct` |
| FP.CPI.TOTL.ZG | `inflation_pct` |
| SL.UEM.TOTL.ZS | `unemployment_pct` |
| IT.NET.USER.ZS | `internet_users_pct` |
| IT.CEL.SETS.P2 | `mobile_subs_per100` |
| ST.INT.DPRT | `tourism_departures` |
| ST.INT.RCPT.CD | `tourism_receipts_usd` |

- EDA: missingness bars, distributions, trends by year, correlation heatmap.

---

## 6) Hofstede Preparation

- Keep `country` (ISO‑3) and six dimensions: `pdi, idv, mas, uai, ltowvs, ivr` (cast to `float32`).  
- EDA: missingness, distributions, correlation.

---

## 7) Integration (country–year → bookings)

- Build a country–year table `ext` from booking aggregates and **LEFT JOIN**:
  1) WDI wide on `['country','arrival_year']` → check fraction with ≥1 missing WDI.  
  2) Hofstede on `['country']` → check fraction with ≥1 missing dim.  
- **LEFT JOIN back** to booking‑level on `['country','arrival_year']` to create `bk_full`.

---

## 8) WDI Imputation (multi‑stage)

Using `UNSD.csv` regions (`country` ↔ `region`):

1. **Per‑country** linear interpolation, then **ffill/bfill**.  
2. **Region–year median** for any remaining gaps.  
3. **Year median** fallback.  
4. **Global medians** (safety net).  
5. Cast all WDI features to `float32`.

A booking‑level flag **`booking_wdi_imputed`** is created (1 if its country–year originally had any missing WDI).

---

## 9) Derived Ratios / Indices (country–year)

Merged into `bk_full`:

- `tourism_departures_per_1000 = tourism_departures / (population_total/1000)`  
- `tourism_receipts_per_capita = tourism_receipts_usd / population_total`  
- `tourism_share_of_gdp = tourism_receipts_usd / (gdp_per_capita_usd * population_total)`  
- `real_gdp_growth_pct = gdp_growth_pct - inflation_pct`  
- `digital_penetration_index = mean(internet_users_pct, mobile_subs_per100)`  
- `pop_0_14_pct = 100 - pop_15_64_pct - pop_65_plus_pct`  
- `dependency_ratio = pop_0_14_pct / pop_15_64_pct`

---

## 10) Hofstede Imputation (country → bookings)

1. **MICE‑style** imputation at country‑level using `IterativeImputer(BayesianRidge, sample_posterior=True, max_iter=20)`.  
   - Round & clip to **[0,100]**.  
   - Country‑level flag kept; merged into both `ext` and `bk_full`.
2. **Region median** fill at booking‑level for any remaining gaps (via UNSD regions).  
3. **Global medians** for any still missing.  
4. Build unified booking flag **`booking_hofstede_imputed`** = union of PMM‑/region‑/global‑filled.  
5. Verification checks confirm region/global fills applied only where needed.

---

## 11) Outlier Handling & Transformations (booking‑level)

- **Domain corrections:**  
  - Negative `adr` → median `adr`.  
  - Negative `cancel_lag` → 0.
- **IQR winsorisation** (continuous vars): one‑sided or two‑sided based on **skew** (cap upper for right‑skew; lower for left‑skew; both for near‑symmetric).  
- **99th‑percentile caps** for low‑cardinality numeric features (`3 ≤ nunique ≤ 100`, excluding `babies`).  
- **Babies** capped to max **1**.  
- **Transformations:** `log1p` for variables with skew > 1; **cube‑root** if still > 1.  
- **High correlation review** (|ρ|>0.8) printed for inspection.

---

## 12) Outputs

- **`bk_full_cleaned.csv`** — modelling‑ready booking‑level table with:  
  - **Identifiers:** `country` (ISO‑3), `arrival_year`, `arrival_date`.  
  - **Target:** `is_canceled`.  
  - **Internal features:** `lead_time`, `lead_time_bin`, `season`, `total_nights`, `total_guests`, `is_long_stay`, `flag_special_requests`, `flag_deposit_taken`, `flag_country_unknown`, `adr` (cleaned), etc.  
  - **WDI features (imputed):** `population_density`, `population_total`, `pop_15_64_pct`, `pop_65_plus_pct`, `urban_pop_pct`, `gdp_per_capita_usd`, `gdp_growth_pct`, `inflation_pct`, `unemployment_pct`, `internet_users_pct`, `mobile_subs_per100`, `tourism_departures`, `tourism_receipts_usd`.  
  - **Derived ratios/indices:** `tourism_departures_per_1000`, `tourism_receipts_per_capita`, `tourism_share_of_gdp`, `real_gdp_growth_pct`, `digital_penetration_index`, `dependency_ratio`.  
  - **Hofstede dimensions (imputed):** `pdi`, `idv`, `mas`, `uai`, `ltowvs`, `ivr`.  
  - **Imputation flags:** `booking_wdi_imputed`, `booking_hofstede_imputed`.  
- File is also downloaded automatically in **Google Colab** via `files.download(...)`.

> Note: Certain features and flags are removed prior to export because of high correlation.

---

## 13) How to Run

### Google Colab
1. Open the notebook.  
2. Upload or mount the four inputs (Bookings, WDI long, Hofstede, UNSD).  
3. **Runtime ▸ Run all**.  
4. Download `bk_full_cleaned.csv` when prompted.

### Local Jupyter / VS Code
1. Python ≥ 3.9 with:
   ```bash
   pip install -U pandas numpy matplotlib seaborn tabulate pycountry rapidfuzz scikit-learn
   ```
2. Place the input CSVs next to the notebook, or adjust the `DATA_FILES` and UNSD paths.  
3. Run all cells. Check the working directory for `bk_full_cleaned.csv`.

---

## 14) Reproducibility & Notes

- `IterativeImputer(..., random_state=0, sample_posterior=True)` ensures **stable stochastic imputation**.  
- Most numeric columns are **downcast** to reduce memory footprint.  
- `UNSD.csv` must be **semicolon‑delimited** and include `ISO-alpha3 Code`, `Region Name`.  
- The notebook prints schema, missingness, memory usage, and correlation diagnostics for audit trails.
