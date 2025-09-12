# Notebook 1: External Data Preparation: WDI & Hofstede

**Project:** *Factors Driving Hotel Booking Cancellations: A Predictive and Geospatial Analysis*  
**Path:** `notebooks/notebook 01/External_Data_Preparation_Notebook.ipynb`

This notebook prepares **country‑level external context**:
- Cleans the **Hofstede cultural dimensions** table.
- Filters and reshapes **World Development Indicators (WDI)** to the target study window.
- Exports **join‑ready CSVs** for downstream feature engineering, modelling, and geospatial analysis.

> Source links and licensing are listed in `DATA_SOURCES.md` (repo root).


---

## 1) Purpose & Scope

The notebook standardises external data so that each booking record (guest origin) can be augmented with:
- **Cultural dimensions** (Hofstede: PDI, IDV, MAS, UAI, LTO, IVR).
- **Macro indicators** (WDI: GDP per capita, inflation, unemployment, internet users, tourism, etc.) for **2015–2017**.

Outputs are designed to be **idempotent** and **joinable** using ISO‑3 country codes and year.


---

## 2) Inputs (expected files)

| Logical name | Default path | Notes |
|---|---|---|
| `bookings` | `hotel_bookings.csv` | Only previewed in this notebook (to check date parsing). Not required for exports. |
| `wdi` | `WDICSV.csv` | Standard WDI export with columns: `Country Name`, `Country Code`, `Indicator Name`, `Indicator Code`, `1960 … 2024` years. |
| `hofstede` | `hofstede.csv` | Raw Hofstede table. Notebook drops an `Unnamed: 0` column if present and rewrites to `hofstede.csv`. |

> File existence is validated at load. Missing files raise a **FileNotFoundError** with the resolved path.

---

## 3) Indicators included (WDI)

The notebook filters **WDI** to the following **Indicator Codes** and years **2015–2017**:

| Code | Description |
|---|---|
| `ST.INT.DPRT` | International tourism departures |
| `NY.GDP.PCAP.CD` | GDP per capita (current US$) |
| `NY.GDP.MKTP.KD.ZG` | GDP growth (annual %) |
| `FP.CPI.TOTL.ZG` | Inflation, consumer prices (annual %) |
| `SL.UEM.TOTL.ZS` | Unemployment (% of labour force) |
| `IT.NET.USER.ZS` | Internet users (% of population) |
| `IT.CEL.SETS.P2` | Mobile cellular subscriptions (per 100 people) |
| `SP.POP.TOTL` | Total population |
| `SP.URB.TOTL.IN.ZS` | Urban population (% of total) |
| `EN.POP.DNST` | Population density (people per sq. km of land area) |
| `SP.POP.1564.TO.ZS` | Population ages 15–64 (% of total) |
| `SP.POP.65UP.TO.ZS` | Population ages 65+ (% of total) |
| `ST.INT.RCPT.CD` | International tourism receipts (current US$) |


---

## 4) Processing pipeline

1. **Environment & helpers**  
   - Imports (`pandas`, `numpy`, `Path`, `IPython.display.display`) and a reusable `load_dataset()` with date parsing.
2. **Data ingestion & preview**  
   - Loads all configured CSVs; prints shapes & dtypes (including parsed dates).  
   - Displays `head()` for quick inspection.
3. **Hofstede cleaning**  
   - Drops the column **`Unnamed: 0`** if present.  
   - Keeps country identifiers and the six dimension columns.
4. **WDI filtering & reshape**  
   - Subsets rows to the 13 indicator codes above.  
   - Keeps only: `Country Name`, `Country Code`, `Indicator Name`, `Indicator Code`, plus **`2015`–`2017`**.  
   - **Melts** to long format → columns: `Country Name`, `Country Code`, `Indicator Name`, `Indicator Code`, `Year`, `Value`.
5. **Exports**  
   - Writes three CSVs in the **current working directory**:  
     - `hofstede.csv` (cleaned)  
     - `wdi_filtered.csv` (wide 2015–2017 columns kept)  
     - `wdi_long.csv` (tidy long)  
   - If running in **Google Colab**, triggers `files.download()` for each export.


---

## 5) Outputs & schemas

### `hofstede.csv` (cleaned)
```
Country (optional) : str   # may be present depending on source
Country Code       : str   # ISO-3 (recommended key name to unify later)
PDI, IDV, MAS,
UAI, LTO, IVR      : float
```
> The raw file name is reused for the cleaned export.

### `wdi_filtered.csv` (wide)
```
Country Name  : str
Country Code  : str   # ISO-3 (join key)
Indicator Name: str
Indicator Code: str
2015, 2016, 2017 : float
```

### `wdi_long.csv` (tidy)
```
Country Name  : str
Country Code  : str   # ISO-3 (join key)
Indicator Name: str
Indicator Code: str
Year          : str   # '2015'/'2016'/'2017' (string in current code)
Value         : float
```


---

## 6) How to run

### Google Colab
1. Open the notebook in Colab.  
2. Upload the three input CSVs (or mount Google Drive).  
3. **Runtime ▸ Run all**.  
4. Download the three exported CSVs when prompted.

### Local Jupyter / VS Code
1. Python ≥ 3.9 with:
   ```bash
   pip install -U pandas numpy
   ```
2. Place `hotel_bookings.csv`, `WDICSV.csv`, and `hofstede.csv` next to the notebook (or update `DATA_FILES` paths).  
3. Run all cells. Find exports in the working directory.

---

## 7) Changelog

- **v1.0 (current):** Initial external data prep for Hofstede + WDI (2015–2017), clean & exports (wide + long).
