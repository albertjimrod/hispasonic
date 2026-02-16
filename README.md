# Hispasonic — Second-Hand Synthesiser Market Analysis

[![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Pandas](https://img.shields.io/badge/Pandas-Data_Analysis-150458?style=flat-square&logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![SQLite](https://img.shields.io/badge/SQLite-Database-003B57?style=flat-square&logo=sqlite&logoColor=white)](https://sqlite.org)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

End-to-end data pipeline and market analysis built on scraped listings from [Hispasonic](https://www.hispasonic.com/), the largest Spanish-language music technology community. The project covers the full cycle from raw CSV files to supply/demand modelling across Spanish cities.

---

## Project Structure

```
hispasonic/
├── data/
│   ├── raw/                             # 12 source CSVs (Aug 2022 – May 2024)
│   └── processed/                       # Unified dataset (gitignored)
├── notebooks/
│   ├── 01_etl.ipynb                     # ETL — load, normalise, clean and unify all CSVs
│   ├── 02_eda.ipynb                     # EDA — temporal, price, brand, city, engagement
│   └── 03_supply_demand.ipynb           # Supply & demand analysis by city and price
├── reports/
│   ├── figures/                         # 27 auto-generated charts
│   └── README.md                        # Chart gallery
├── requirements.txt
└── README.md
```

---

## Quick Start

```bash
git clone https://github.com/albertjimrod/hispasonic.git
cd hispasonic

pip install -r requirements.txt

# Run notebooks in order
jupyter notebook notebooks/01_etl.ipynb
jupyter notebook notebooks/02_eda.ipynb
jupyter notebook notebooks/03_supply_demand.ipynb
```

> **Note:** `data/processed/` is gitignored. Run `01_etl.ipynb` first to generate `hispasonic_unified.csv` before running the other notebooks.

---

## Notebook 1 — ETL Pipeline (`01_etl.ipynb`)

### Goal
Load 12 heterogeneous CSV files scraped from Hispasonic across different dates, detect and resolve structural inconsistencies, and consolidate everything into a single clean dataset.

### Input
- **12 CSV files** in `data/raw/`, spanning August 2022 to May 2024
- File naming conventions vary: `2022_08_26.csv`, `hpw2024517.csv`, etc.
- Total raw rows: **5,962 listings**

### Step-by-step process

#### Step 1 — Column inspection
Each CSV is loaded with zero rows and its column list compared against a defined `CORE_COLUMNS` schema of 16 fields. The inspection reveals three types of extra columns present in different files:

| Extra column | Files affected | Action |
|---|---|---|
| `user` | 2022_08_26, 2022_08_30, 2022_10_01 onwards | Dropped (not in core schema) |
| `anon_user` | 2022_09_01 onwards | Dropped |
| `Unnamed: 0` | hpw2024517, hpw2024526 | Dropped (auto-generated index) |

All 12 files contain all 16 core columns — no column is missing from any file.

#### Step 2 — Load and normalise
Each CSV is loaded in full, non-core columns are dropped, and a `source_file` column is added for traceability. All 12 frames are then concatenated into a single unified DataFrame.

```
2022_08_26.csv  →  764 rows   │  2023_03_01.csv  →  270 rows
2022_08_30.csv  →  760 rows   │  2023_06_04.csv  →  261 rows
2022_09_01.csv  →  805 rows   │  2023_06_07.csv  →  261 rows
2022_10_01.csv  →  837 rows   │  2023_08_07.csv  →  425 rows
2022_11_01.csv  →  784 rows   │  hpw2024517.csv  →  264 rows
2022_12_31.csv  →  264 rows   │  hpw2024526.csv  →  267 rows
─────────────────────────────────────────────────────────────
Total                             5,962 rows × 17 columns
```

#### Step 3 — Data type fixing

| Column(s) | Original type | Converted to | Reason |
|---|---|---|---|
| `published`, `expire`, `date_scrapped` | `object` (string) | `datetime64[ns]` | Enable date arithmetic and time-series grouping |
| `price` | `int64` | `float64` (coerce errors) | Handle edge cases and future NaN compatibility |
| `seen` | `int64` | `Int64` (nullable) | Preserve NaN capability |
| `urgent`, `buy`, `change`, `sell`, `gift`, `search`, `repair`, `parts` | `int64` | `Int64` (nullable) | All are binary 0/1 flags |

#### Step 4 — Null analysis
After concatenation, only 3 nulls are found, all in the `description` column. All other columns are complete. The `price` column contains valid zeros (free/gift listings) which are intentional.

#### Step 5 — Duplicate detection
Using a composite key of `[description, price, published, city]`, **2,059 duplicate rows** are identified. These are listings that appear in multiple scrape snapshots (the same listing was still active when a later scrape ran). The duplicates are **intentionally retained** in the dataset because they carry temporal information — a listing appearing in 3 scrapes means it was active for 3 consecutive periods. The supply/demand analysis in notebook 03 relies on per-scrape-date counts, so removing duplicates would distort the temporal signal.

#### Step 6 — Output
The unified dataset is saved to `data/processed/hispasonic_unified.csv`.

```
Final shape:  5,962 rows × 17 columns
Date range:   2022-08-26 → 2024-05-26
Unique brands: 249
Unique cities:  49
Scrape dates:   12
```

### Dataset schema

| Column | Type | Description |
|---|---|---|
| `urgent` | Int64 (0/1) | Listing marked as urgent |
| `buy` | Int64 (0/1) | Active buy request |
| `change` | Int64 (0/1) | Swap/trade offer |
| `sell` | Int64 (0/1) | Item for sale (supply) |
| `price` | float64 | Asking price in euros |
| `gift` | Int64 (0/1) | Free item |
| `search` | Int64 (0/1) | Wanted/search listing (demand) |
| `repair` | Int64 (0/1) | Repair service listing |
| `parts` | Int64 (0/1) | Spare parts listing |
| `synt_brand` | object | Brand name (249 unique values) |
| `description` | object | Free-text listing description |
| `city` | object | City of the listing (49 unique values) |
| `published` | datetime | Original listing publication date |
| `expire` | datetime | Listing expiry date |
| `date_scrapped` | datetime | Date the listing was captured |
| `seen` | Int64 | Total views on the listing |
| `source_file` | object | Source CSV filename |

---

## Notebook 2 — Exploratory Data Analysis (`02_eda.ipynb`)

### Goal
Deep exploratory analysis of the unified dataset. The focus is on temporal evolution, price dynamics, brand and city patterns, engagement metrics, and correlation groups. All 18 figures are saved automatically to `reports/figures/`.

### Key findings

#### Temporal evolution (Figs 01–03)
- Listing volume shows a sharp drop from ~800 listings/scrape in late 2022 to ~260 in early 2023, then a partial recovery to ~400 before stabilising. This structural break in early 2023 is the most important temporal event in the dataset.
- Sell listings consistently represent ~92% of all listing types. Buy and search (demand signals) never exceed 5% combined.
- Median sell price fluctuates between €150 and €350 over the observation period, with the mean pulled significantly higher by extreme outliers (max price: €111,111 — likely a data entry error).

#### Price analysis (Fig 04)
- Median sell price: **€250**. Mean: €637 (distorted by outliers).
- At the 99th percentile the cap is €5,500. Above that, 47 listings (0.86%) are considered outliers.
- The price distribution is strongly right-skewed: most instruments trade between €75 and €550 (IQR), but the tail extends to very high values.

#### Brand analysis (Figs 05–08)
- The top brand by volume is `-` (listings with no brand assigned), followed by Roland, Korg, Moog and Behringer.
- Moog and Buchla command the highest median sell prices (€800–1,200+), reflecting their premium market positioning.
- Roland and Korg show the most stable volume over time, while smaller brands appear and disappear sporadically.
- The brand × scrape date heatmap confirms that market activity is brand-diverse but volume-concentrated in a handful of labels.

#### Geographic analysis (Figs 09–11)
- Madrid dominates with 1,461 listings (~24% of total), followed by Barcelona (~1,015) and Girona (~541).
- Median sell price varies significantly by city: Burgos, Navarra and Baleares show higher median prices, while La Rioja and Gipuzkoa trend lower — likely reflecting instrument category mix rather than genuine geographic pricing power.
- The top 6 cities by volume all follow the same temporal pattern: peak activity in late 2022, sharp drop in early 2023, partial recovery.

#### Engagement — `seen` (Figs 12–14)
- The `seen` distribution is heavily right-skewed: median ~387 views, mean ~790, max 25,152.
- Price vs views scatter shows no clear linear relationship — cheap listings attract high view counts (accessible to more buyers), while expensive items accumulate fewer but more targeted views.
- Moog and Buchla generate the highest median view counts, consistent with their community reputation and pricing.

#### Correlation matrix (Fig 15)
- `sell` and `price` show a weak positive correlation: sell listings tend to have a price assigned while buy/search/change listings often show zero.
- `seen` has near-zero correlation with all boolean listing-type columns — engagement is independent of listing category.
- The buy, search, change, repair and gift columns are all near-zero with each other, confirming they represent distinct, non-overlapping listing intents.

#### Cross-group analysis (Figs 16–18)
- The brand × city heatmap shows extreme concentration: Roland and Korg in Madrid and Barcelona account for a disproportionate share of all brand-city combinations.
- Median price by brand and listing type (sell/buy/change) reveals that buy requests for premium brands (Moog, Oberheim) reflect realistic market expectations — buyers are willing to pay close to sell prices.
- Price evolution by top 5 brands is volatile at the per-scrape level due to small sample sizes, but Roland and Korg maintain stable long-term medians.

### Figures generated

| Fig | File | Description |
|-----|------|-------------|
| 01 | `01_listing_volume_over_time.png` | Listing volume per scrape date |
| 02 | `02_price_evolution_over_time.png` | Median and mean sell price over time (with IQR band) |
| 03 | `03_listing_type_share_over_time.png` | Sell / Buy / Change / Search share over time |
| 04 | `04_price_distribution.png` | Price histogram and boxplot (sell listings, ≤p99) |
| 05 | `05_top_brands_by_listings.png` | Top 15 brands by listing count |
| 06 | `06_median_price_per_brand.png` | Median sell price per brand |
| 07 | `07_brand_evolution_over_time.png` | Top 8 brands — listing volume over time |
| 08 | `08_brand_time_heatmap.png` | Brand × scrape date heatmap |
| 09 | `09_top_cities_by_listings.png` | Top 15 cities by listing count |
| 10 | `10_median_price_per_city.png` | Median sell price per city |
| 11 | `11_city_evolution_over_time.png` | Top 6 cities — listing volume over time |
| 12 | `12_seen_distribution.png` | Views distribution |
| 13 | `13_price_vs_seen_scatter.png` | Price vs views scatter |
| 14 | `14_median_seen_per_brand.png` | Median views per brand |
| 15 | `15_correlation_matrix.png` | Correlation matrix — all numeric variables |
| 16 | `16_brand_city_heatmap.png` | Brand × City listing count heatmap |
| 17 | `17_price_by_brand_and_type.png` | Median price by brand and listing type |
| 18 | `18_price_evolution_by_brand.png` | Median sell price over time — top 5 brands |

---

## Notebook 3 — Supply & Demand Analysis (`03_supply_demand.ipynb`)

### Goal
Determine whether a relationship exists between the supply/demand balance per city and median sell prices. Test whether the D/S ratio at time *t* can predict prices at time *t+1*. All 9 figures are saved automatically to `reports/figures/`.

### Definitions

| Concept | Column(s) | Logic |
|---|---|---|
| **Supply** | `sell == 1` | Someone is offering an item for sale |
| **Demand** | `buy == 1` or `search == 1` | Someone is actively looking to buy |
| **D/S ratio** | demand / supply × 100 | Expressed as %. > 100% = seller's market |
| **Passive demand** | `seen` | Total views — proxy for passive interest |

### Hypotheses tested
1. **Cross-sectional:** Cities with a higher D/S ratio should show higher median sell prices.
2. **Temporal:** When the D/S ratio rises in a city at scrape *t*, prices should follow at scrape *t+1*.

### Key metrics

| Metric | Value |
|---|---|
| Total supply listings | 5,480 |
| Total demand listings | 242 |
| Global D/S ratio | **4.42%** |
| HHI (market concentration) | **1,638** (moderately concentrated) |
| Madrid + Barcelona volume share | **51.4%** |
| Madrid + Barcelona HHI contribution | **82.6%** |
| Fully visible cities (all thresholds met) | **5** of 15 |

### Analysis steps

#### 1. Supply and demand aggregation by city (Figs 19–22)
Cities are aggregated by supply count, demand count and median price. A minimum of 5 supply listings is required to include a city. The D/S ratio is computed as `(demand / supply) × 100`.

The **Visibility Threshold** classifies cities using three simultaneous criteria:
- Market share > 2%
- Total volume > 100 listings
- Active demand > 10 listings

Only **5 cities pass all three filters**: Madrid, Barcelona, Valencia, Zaragoza and Alicante — representing 68% of total market volume.

The **Herfindahl-Hirschman Index (HHI = 1,638)** classifies the market as moderately concentrated. Madrid and Barcelona account for 82.6% of that concentration despite representing only 51.4% of volume, due to the squared weighting in the HHI formula.

The **Market Attractiveness Score** (weighted composite: 35% Volume + 25% HHI + 30% D/S Ratio + 10% Visibility) produces a 0–100 ranking:
- No city reaches **HIGH** (≥80)
- Madrid (79.2) and Barcelona (56.3) reach **MEDIUM**
- All remaining 13 cities are **LOW**

#### 2. Temporal evolution by city (Figs 23–24)
Supply and demand are aggregated per city per scrape date. The five largest cities (Madrid, Barcelona, Girona, Valencia, Zaragoza) are tracked over time.

All cities show the same temporal pattern: peak supply in late 2022, sharp collapse in early 2023 (synchronous across all cities), partial recovery from mid-2023. **Girona** is an outlier: supply peaked at ~125 in late 2022 and then collapsed to near zero, never recovering. This produces an artefactual D/S ratio spike of 100% in July 2023 (zero supply + one demand listing = extreme ratio).

#### 3. Global market evolution (Fig 25)
At national level, supply halved from ~750 to ~250 listings in January 2023 and never fully recovered. The mean D/S ratio over the full period is **3.9%**. The price time series shows two anomalies caused by data sparsity in early 2023 (near-zero median followed by a sharp recovery) — these are data quality artefacts, not genuine market events.

#### 4. Cross-sectional hypothesis test (Fig 21)
Pearson correlation between D/S ratio and median price across cities:
- **r = -0.291, p = 0.0529** — not statistically significant (borderline)
- Direction is *negative* (counterintuitive): cities with higher relative demand tend to have *lower* prices
- Explanation: cities with the highest D/S ratios (Ciudad Real, La Rioja, Gipuzkoa) are small markets trading low-value instruments. Price is determined by instrument type, not local supply/demand pressure.

#### 5. Temporal (lagged) hypothesis test (Fig 27)
For each city with ≥4 scrape periods of non-zero data, Pearson correlation is computed between D/S ratio(t) and median price(t+1):
- **Valencia: r = 0.819, p = 0.002** — the only statistically significant result
- All other 27 cities: not significant (p > 0.05)
- Valencia is the only market where the classic economic mechanism (ratio leading price) is empirically active

#### 6. Passive demand proxy (Fig 26)
Mean views per city vs median price:
- **r = -0.003, p = 0.985** — essentially zero correlation
- Views measure curiosity, not purchase intent. Cheap listings attract more views, creating an inverse pull that cancels any positive price signal.

### Final conclusions

1. **Permanent buyer's market:** The market is structurally oversupplied at a global D/S ratio of 4.42%. This condition persists across all cities and all time periods without exception.
2. **Central hypothesis rejected:** D/S ratio does not predict price (r = -0.291, p = 0.0529). Synthesiser prices are set by the instrument, not by local supply/demand dynamics.
3. **Geographic concentration:** Two cities (Madrid + Barcelona) dominate 51% of volume and 83% of market concentration. Any national-level model is primarily shaped by these two markets.
4. **Only 5 cities are strategically actionable** under the composite visibility threshold.
5. **Valencia is the exception:** It is the only city where supply/demand ratio has demonstrated statistically significant predictive power over future prices (r = 0.819, p = 0.002).
6. **Views are not a price predictor** at the city aggregation level (r ≈ 0, p = 0.985).
7. **Early-2023 structural break** is the dominant temporal event and the main data quality risk for longitudinal modelling.

### Figures generated

| Fig | File | Description |
|-----|------|-------------|
| 19 | `19_supply_vs_demand_by_city.png` | Supply vs demand side by side — top 15 cities |
| 20 | `20_sd_ratio_ranking_by_city.png` | D/S ratio ranking — all markets are buyer's markets |
| 21 | `21_sd_ratio_vs_price_scatter.png` | D/S ratio vs median price scatter (with regression) |
| 22 | `22_sd_ratio_and_price_dual_axis.png` | D/S ratio and median price dual axis — top 12 cities |
| 23 | `23_supply_demand_over_time_by_city.png` | Supply and demand over time — top 5 cities |
| 24 | `24_sd_ratio_evolution_by_city.png` | D/S ratio evolution over time — top 5 cities |
| 25 | `25_global_supply_demand_ratio_price.png` | Global market: supply, demand, ratio and price over time |
| 26 | `26_seen_vs_price_by_city.png` | Mean views (passive demand) vs median price per city |
| 27 | `27_lagged_correlation_sd_ratio_vs_price.png` | Lagged correlation: does D/S ratio at (t) predict price at (t+1)? |

---

## Tech Stack

| Category | Tools |
|----------|-------|
| Scraping | `requests`, `BeautifulSoup` |
| Data | `pandas`, `numpy`, `SQLite` |
| Visualisation | `matplotlib`, `seaborn` |
| Statistics | `scipy.stats` |
| Environment | `Jupyter Notebooks` |

---

## Author

**Alberto Jiménez** — [datablogcafe.com](https://datablogcafe.com) | [GitHub](https://github.com/albertjimrod)

**Repository:** [github.com/albertjimrod/hispasonic](https://github.com/albertjimrod/hispasonic)

---

## License

MIT License — feel free to use and modify.
