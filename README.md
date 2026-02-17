# Hispasonic — Second-Hand Synthesiser Market Analysis

[![Python](https://img.shields.io/badge/Python-3.9-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
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
├── jupyterlab_new.yaml                  # Conda environment (Python 3.9)
├── requirements.txt                     # Legacy pip requirements (Python 3.8)
└── README.md
```

---

## Quick Start

```bash
git clone https://github.com/albertjimrod/hispasonic.git
cd hispasonic

# Create and activate the conda environment
conda env create -f jupyterlab_new.yaml
conda activate jupyterlab_new

# Run notebooks in order
jupyter lab notebooks/01_etl.ipynb
jupyter lab notebooks/02_eda.ipynb
jupyter lab notebooks/03_supply_demand.ipynb
```

> **Note:** `data/processed/` is gitignored. Run `01_etl.ipynb` first to generate `hispasonic_unified.csv` before running the other notebooks.
>
> The environment file `jupyterlab_new.yaml` pins all dependencies (Python 3.9, pandas 2.2.3, scikit-learn 1.6.1, XGBoost 2.1.1, JupyterLab 4.4.4). A legacy `requirements.txt` is kept for reference but targets a Python 3.8 environment.

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
| `buy` | Int64 (0/1) | Active buy request (demand signal) |
| `change` | Int64 (0/1) | Swap/trade offer |
| `sell` | Int64 (0/1) | Item for sale (supply signal) |
| `price` | float64 | Asking price in euros (0 = free or price on request) |
| `gift` | Int64 (0/1) | Free item |
| `search` | Int64 (0/1) | Wanted/search listing (demand signal) |
| `repair` | Int64 (0/1) | Repair service listing |
| `parts` | Int64 (0/1) | Spare parts listing |
| `synt_brand` | object | Extracted brand name (249 unique values; `"-"` = unidentified) |
| `description` | object | Free-text listing description |
| `city` | object | City of the listing (49 unique values) |
| `published` | datetime | Original listing publication date |
| `expire` | datetime | Listing expiry date |
| `date_scrapped` | datetime | Date the listing was captured by the scraper |
| `seen` | Int64 | Total views accumulated by the listing |
| `source_file` | object | Source CSV filename (traceability) |

### Key data quality notes

- **2,059 rows (34.5%) are duplicates** by composite key — expected and intentional. They reflect listings that were still active when a later scrape ran and carry temporal information required by the supply/demand analysis in notebook 03.
- **3 null values** in `description` only; all other 16 columns are 100% complete.
- **Zeros in `price`** are legitimate (free items, gift listings, or "price on request") — do not treat as missing.
- The **2022-08-30 session** has a broken brand-extraction: all 760 listings are classified as `synt_brand = "-"`. Confirmed and documented in notebook 02.
- Three sessions (**2022-12-31, 2023-03-01, 2023-06-04**) have price recorded as 0 for all sell listings — a scraping artefact confirmed in notebook 02. These sessions must be excluded from any price-based analysis.

---

## Notebook 2 — Exploratory Data Analysis (`02_eda.ipynb`)

### Goal
Deep exploratory analysis of the unified dataset. The focus is on temporal evolution, price dynamics, brand and city patterns, engagement metrics, and correlation groups. All 18 figures are saved automatically to `reports/figures/`.

### Key findings

#### Temporal evolution (Figs 01–03)
- Listing volume peaks at ~800 listings/scrape in late 2022, then drops to ~260–430 from early 2023 onwards. A gap with no scraping is visible between Oct 2023 and May 2024. This irregular cadence must be accounted for in any temporal analysis.
- **Three scraping sessions (2022-12-31, 2023-03-01, 2023-06-04) have missing price data** (all prices recorded as 0 €) — a confirmed data quality artefact, not a real market event. The **2022-08-30 session shows a broken brand extraction** where all listings fall into the "-" bucket.
- Sell listings consistently represent **~90–95%** of all listing types at every point in time. Buy and Change together never exceed ~7%. The marketplace composition is structurally stable throughout the full period.
- The **median sell price** is remarkably stable at **~300–350 €** across all sessions. The mean (~520–820 €) is systematically inflated by a thin premium segment, confirming the distribution is right-skewed with a long tail.

#### Price analysis (Fig 04)
- Median sell price: **~250 €**. IQR: **€75–550**. Mean: ~637 € (distorted by outliers).
- At the 99th percentile the cap is **€5,500**. Above that, 47 listings (0.86%) are considered outliers and likely represent high-end vintage instruments.
- The price distribution is strongly **right-skewed**: most instruments trade below €500, but the tail extends legitimately to several thousand euros.

#### Brand analysis (Figs 05–08)
- The top entry by volume is **"-"** (~1,530 listings) — listings where no brand could be extracted from the free text. This is a brand-detection limitation, not a real brand.
- Among identifiable brands: **Roland** (~580), **Yamaha** (~420), **Korg** (~390) lead by volume. **Eurorack** appears in 4th place but represents a *format*, not a manufacturer — it captures modular listings without a specific maker name.
- **Hammond** (~1,400 €) leads median sell price by a large margin, followed by **Elektron** (~620 €) and **Moog** (~500 €). The brands with the highest listing counts (Roland, Yamaha, Korg) sit in the mid-range at ~300–350 €, reflecting their broad product catalogues.
- **Korg** shows the highest and most consistent median sell price over time (~300–550 €). **Roland** is the most price-stable brand (±50 € across the full period). **Yamaha** is the most volatile.
- The brand × scrape date heatmap reveals the **2022-08-30 anomaly** clearly: all named brands show 0 while "-" shows 760 — confirming a data pipeline failure on that date.

#### Geographic analysis (Figs 09–11)
- **Madrid** (~1,450 listings) and **Barcelona** (~1,040) together account for ~42% of all listings. **Girona** is a surprising third (~535), which may reflect province-level geographic tagging rather than genuine city activity.
- A **counterintuitive inverse relationship** exists between listing volume and median price: the highest-volume cities (Madrid ~215 €, Barcelona ~230 €) show the *lowest* median prices, while low-volume cities (Tarragona ~390 €, Sevilla ~370 €) show the *highest*. Greater competition in large markets compresses prices; smaller markets see more selective, higher-value listings.
- All top cities follow the same temporal pattern: peak activity late 2022, sharp drop in early 2023, partial recovery. **Girona** is an outlier — it collapsed to near zero after early 2023 and never recovered, likely a data capture issue rather than a real market exit.

#### Engagement — `seen` (Figs 12–14)
- The `seen` distribution is strongly **right-skewed** with a Pareto-like shape: most listings receive 100–300 views, while a small minority reaches 4,000–6,000+. Engagement is highly concentrated.
- Price vs views scatter shows **no strong positive correlation**. There is a weak moderate relationship (+0.37 per the correlation matrix), but visually the scatter is dominated by the dense low-price cluster. High-priced items attract a narrower, more targeted audience.
- Listings with price = 0 ("price on request") often attract high views — curiosity-driven traffic independent of price.

#### Correlation matrix (Fig 15)
- `sell` ↔ `buy` (−0.68) and `sell` ↔ `change` (−0.69): strong negatives by construction — the listing type flags are mutually exclusive. These reflect the data encoding scheme, not a market phenomenon.
- `price` ↔ `seen` = **+0.37**: the most meaningful correlation — higher-priced listings attract somewhat more views, likely because they represent more desirable or aspirational instruments.
- `repair` ↔ `parts` = **+0.37**: co-occurrence of repair and parts listings makes intuitive sense.
- `parts` ↔ `seen` = **+0.38**: parts listings attract above-average views, as a single spare part can satisfy multiple potential buyers simultaneously.
- Most other pairs are near zero, indicating **low multicollinearity** — a favourable property for downstream modelling.

#### Cross-group analysis (Figs 16–18)
- The brand × city heatmap confirms **Madrid as the dominant market for every brand**. Roland (145), Yamaha (122), Korg (112) and Eurorack (83) all peak there. **Elektron** is noticeably more concentrated in large cities (Madrid 27, Barcelona 22), consistent with its premium boutique positioning.
- Median price by brand and listing type reveals an anomaly: **Behringer "Change" listings show a median of ~1,875 €**, far above Behringer's typical sell price (~110 €). This suggests trade-up dynamics — sellers offering Behringer equipment as part of a bundle exchange for a single high-value item.
- The brand × time price chart confirms that **Korg commands the highest and most consistent median price** among the top brands, while Roland is the most price-stable and Yamaha the most volatile.

### Key takeaways

1. **Data quality issues are significant** — three sessions with zeroed prices and one with broken brand extraction must be excluded or imputed before any modelling.
2. **Hispasonic is structurally a selling platform** — over 90% of listings are sales, consistently across the full period.
3. **Prices are right-skewed with a stable median of ~250–350 €** — robust estimators or log-transformation are recommended for any regression task.
4. **High volume ≠ high value** — the brands with the most listings (Roland, Yamaha, Korg) are not the most expensive. Hammond, Elektron, and Moog lead on price with far fewer listings.
5. **Larger cities have lower prices** — the inverse volume-price relationship across cities reflects competitive dynamics, not geographic pricing differences.
6. **Engagement follows a Pareto distribution** — price is a weak predictor of views (+0.37). Most traffic concentrates on a small fraction of listings.
7. **Eurorack should be treated as a category, not a brand** — it aggregates hundreds of small module manufacturers and its statistics are not comparable to single-brand entries.
8. **Irregular scraping cadence** — volume trends over time are as much a function of scraping frequency as real market dynamics.

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
| Mean D/S ratio over time | **3.9%** |
| HHI (market concentration) | **1,638** (moderately concentrated) |
| Madrid + Barcelona volume share | **51.4%** |
| Madrid + Barcelona HHI contribution | **82.6%** |
| Fully visible cities (all thresholds met) | **5** of 15 |
| Cities with significant lagged correlation | **1** of 28 (Valencia only) |

### Analysis steps

#### 1. Supply and demand by city — structural imbalance (Fig 19)
Supply bars dominate demand in every single city. The demand signal is near-invisible at the chart scale. Four properties define the market structure:
- Supply and demand are structurally imbalanced across the entire geography
- Liquidity is concentrated in a few large hubs (Madrid, Barcelona)
- In medium and small cities, active demand is essentially absent
- The imbalance worsens proportionally as city volume decreases

#### 2. Market concentration — HHI and Pareto (Figs 19, inline charts)
The **Herfindahl-Hirschman Index (HHI = 1,638)** classifies the market as *moderately concentrated* (DOJ/FTC range: 1,500–2,500):
- **Madrid alone contributes 53.9%** of the HHI; Barcelona adds 28.6% — together 82.6%
- Girona appears as an anomalous 3rd contributor (~8.1%), likely reflecting supply-only batch listings rather than organic activity
- The remaining 12 cities collectively add only 9.3% — they are statistically irrelevant at national scale

The **Pareto curve** confirms that **6 cities (40%) account for 80% of total volume**: Madrid, Barcelona, Girona, Valencia, Zaragoza and Alicante. The step from Madrid to Barcelona alone adds 21 percentage points — the steepest single step on the curve.

#### 3. Visibility threshold and market attractiveness score (inline charts)
Cities are classified against three simultaneous criteria: market share > 2%, volume > 100 listings, active demand > 10 listings.

Only **5 cities pass all three filters**: Madrid, Barcelona, Valencia, Zaragoza and Alicante — representing **68% of total market volume**. Notable failure: **Girona** (3rd by volume, ~541 listings) is excluded because it has only 6 active demand listings. Its volume is supply-only activity — many sellers, almost no buyers.

The **Market Attractiveness Score** (35% Volume + 25% HHI weight + 30% D/S Ratio + 10% Visibility) ranks all 15 cities on a 0–100 scale:

| City | Score | Zone | Limiting factor |
|------|-------|------|----------------|
| Madrid | ~79.2 | MEDIUM | Low D/S ratio (3.49%) caps demand component |
| Barcelona | ~56.3 | MEDIUM | Even lower D/S ratio (2.84%) |
| Valencia | ~41 | LOW | Good D/S ratio but insufficient volume |
| Alicante | ~35 | LOW | Same profile as Valencia but smaller |
| Baleares | ~33 | LOW | Best D/S ratio (11.36%) but tiny volume |

No city reaches **HIGH (≥80)**. Madrid scores 79.2 — just below the threshold. The HIGH zone is structurally unreachable because no city simultaneously combines high volume, high HHI weight, D/S ratio > 10% and full visibility.

#### 4. D/S ratio ranking across cities (Fig 20)
Every bar is blue — every city is a buyer's market. Three clusters emerge by ratio level:
- **Cold (ratio < 3%):** Bizkaia (0.86%), Girona (1.12%), Barcelona (2.84%). Near-zero demand signal despite economic weight. Barcelona's large supply base dilutes its 28 demand listings to 2.84%.
- **Warm (3–10%):** Madrid (3.49%), Valencia, Zaragoza, Sevilla, Alicante, Granada. Most of the market lives here.
- **Above 10% — only Baleares (11.36%):** the single city closest to a seller's market; the only one above the 10% demand health threshold.

High-ratio outliers (Ciudad Real 57%, La Rioja 38%, Gipuzkoa 36%) are **small-denominator artefacts**: a handful of demand listings over near-zero supply. They are not actionable.

#### 5. Cross-sectional hypothesis test (Fig 21)
Pearson correlation between D/S ratio and median price across cities:
- **r = −0.291, p = 0.0529** — not statistically significant
- Direction is *negative* (counterintuitive): cities with more relative demand tend to have *lower* prices

Two explanations for the negative direction: (1) high-ratio cities are small markets trading cheap instruments — buyers seek affordable gear; (2) synthesiser prices are nationally referenced — local D/S ratio gives no pricing power because buyers can order from any city. **Local ratio analysis must be combined with brand/instrument-type data to understand price drivers.**

#### 6. Dual-axis confirmation (Fig 22)
**Bizkaia** is the most striking counterexample: lowest D/S ratio in the chart (0.86% — almost no buyers) yet one of the highest median prices (~€350). This makes the independence between local demand ratio and price visually undeniable. Price is driven by the composition of the local listing mix, not by local supply/demand dynamics.

#### 7. Temporal evolution of supply and demand (Figs 23–24)
Supply and demand are tracked per city across the 12 scrape dates. The demand line runs flat near zero for all cities throughout 2022–2024 — supply and demand never converge.

- **The early-2023 supply collapse is the dominant temporal event.** Supply halved synchronously across all cities simultaneously in January 2023. This synchrony confirms it is a platform-wide event (scraping frequency change or platform restructuring), not a genuine market contraction.
- **Girona's collapse:** supply peaked at ~125 in late 2022 then dropped to near zero after January 2023, never recovering. The resulting D/S ratio spike to 100% in July 2023 is a zero-denominator artefact (one demand listing against zero supply), not a real market event.
- **Valencia** is the only city that consistently maintains a slightly elevated D/S ratio (5–13%) relative to the others, foreshadowing its significance in the lagged correlation analysis.

#### 8. Global market evolution (Fig 25)
At national level, supply halved from ~750 to ~250 listings in January 2023 and never recovered. The mean D/S ratio is 3.9% with no trend toward equilibrium. The price time series shows two sparsity artefacts in early 2023 (near-zero median, then a sharp recovery) — these are **data quality events, not real market movements**.

#### 9. Passive demand proxy (Fig 26)
Mean views per city vs median price:
- **r = −0.003, p = 0.985** — essentially zero correlation
- Cheap listings attract more views, pulling the aggregate against any positive price signal
- Views reflect curiosity, not purchase intent; geographic distribution of viewers is unknown
- `seen` may be useful as a **listing-level engagement indicator** but loses all predictive signal when aggregated by city

#### 10. Lagged hypothesis test (Fig 27)
For each city with ≥4 scrape periods of non-zero data (28 of 49 cities pass this filter), Pearson correlation is computed between D/S ratio(t) and median price(t+1):
- **Valencia: r = 0.819, p = 0.002** — the only statistically significant result (robust: probability of observing r ≥ 0.819 by chance with n = 11 is 0.2%)
- **All other 27 cities:** not significant (p > 0.05)

Why only Valencia? It is the one city where three conditions coincide: sufficient temporal coverage with non-zero demand in most periods, a moderately consistent buyer base (27 demand listings spread over time), and enough price variance. Madrid and Barcelona have more data but their D/S ratio is so consistently low that there is no meaningful variation for the correlation to capture.

### Final conclusions

1. **Permanent buyer's market.** The global D/S ratio is 4.42% — for every 100 sell listings, only 4 active buyers exist. This condition holds consistently across all cities and all time periods without exception. The market would need to roughly double its active demand, or halve its supply, to approach equilibrium.
2. **Central hypothesis rejected.** D/S ratio does not predict price (r = −0.291, p = 0.0529). Synthesiser prices are determined by the instrument itself (brand, model, category) and by national reference pricing — not by local supply/demand pressure.
3. **Market concentrated in two cities.** Madrid + Barcelona account for 51.4% of volume and 82.6% of the HHI. Any national-level model is primarily shaped by these two markets. City-specific segmentation is methodologically required.
4. **Only 5 cities are strategically actionable** under the composite visibility threshold (Madrid, Barcelona, Valencia, Zaragoza, Alicante). Together they represent 68% of total market volume.
5. **No city justifies priority investment.** The highest composite score achieved is 79.2 (Madrid), just below the HIGH threshold of 80. The HIGH zone is structurally unreachable at current market conditions.
6. **Valencia is the only exception.** It is the only city where D/S ratio(t) predicts median price(t+1) with statistical significance (r = 0.819, p = 0.002). It is the only market in the dataset where the classic supply/demand pricing mechanism is empirically active.
7. **Views are not a price predictor** at the city level (r ≈ 0, p = 0.985). The `seen` metric measures passive curiosity, not purchase intent.
8. **The early-2023 structural break is the main methodological risk.** Supply dropped by half synchronously across all cities in January 2023 and never recovered. Any longitudinal model that spans this break should treat it as a regime change, not a continuous series.

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
| Language | Python 3.9 |
| Scraping | `requests`, `BeautifulSoup4` |
| Data | `pandas 2.2`, `numpy 2.0`, `pyarrow` |
| Visualisation | `matplotlib 3.9`, `seaborn 0.13`, `altair 5` |
| Statistics / ML | `scipy.stats`, `scikit-learn 1.6`, `XGBoost 2.1`, `shap`, `numba` |
| Environment | `JupyterLab 4.4`, `streamlit 1.50` |
| Package manager | `conda` (`jupyterlab_new.yaml`) |

---

## Author

**Alberto Jiménez** — [datablogcafe.com](https://datablogcafe.com) | [GitHub](https://github.com/albertjimrod)

**Repository:** [github.com/albertjimrod/hispasonic](https://github.com/albertjimrod/hispasonic)

---

## License

MIT License — feel free to use and modify.
