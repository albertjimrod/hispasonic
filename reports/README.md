# Reports — Figure Gallery

All charts are generated automatically by the project notebooks and saved to `figures/`.
Re-run each notebook to regenerate its figures.

| Notebook | Figures | Source |
|----------|---------|--------|
| `02_eda.ipynb` | 01–18 | EDA — temporal, price, brand, city, engagement, correlations |
| `03_supply_demand.ipynb` | 19–27 | Supply & demand analysis by city and price |

---

## Temporal Evolution

> Listing activity and price trends across the 12 scrape dates (Aug 2022 – May 2024).

| # | Chart | Key insight |
|---|-------|-------------|
| 01 | ![](figures/01_listing_volume_over_time.png) | Supply peaks at ~800 listings/scrape in late 2022 then drops to ~260–430. Irregular cadence: sessions missing Oct 2023 – May 2024. |
| 02 | ![](figures/02_price_evolution_over_time.png) | Median sell price stable at ~300–350 € across the full period. Mean (~520–820 €) inflated by a thin premium segment. Three sessions show price = 0 — a data quality artefact, not a real market event. |
| 03 | ![](figures/03_listing_type_share_over_time.png) | Sell listings represent ~90–95% of all types consistently. Marketplace structure unchanged between 2022 and 2024. |

---

## Price Analysis

> Distribution, outliers and spread of sell listing prices.

| # | Chart | Key insight |
|---|-------|-------------|
| 04 | ![](figures/04_price_distribution.png) | Median ~€250, IQR €75–550. Strongly right-skewed: most listings under €500, but a legitimate tail reaches €5,500+. 47 listings (0.86%) above p99 excluded from view. |

---

## Brand Analysis

> Volume, price positioning and temporal activity by brand.

| # | Chart | Key insight |
|---|-------|-------------|
| 05 | ![](figures/05_top_brands_by_listings.png) | "-" leads (~1,530) — a brand-detection gap, not a real brand. Roland (~580), Yamaha (~420), Korg (~390) are the top identifiable brands. Eurorack is a format, not a manufacturer. |
| 06 | ![](figures/06_median_price_per_brand.png) | Hammond (~€1,400) leads median price, followed by Elektron (~€620) and Moog (~€500). High-volume brands (Roland, Yamaha, Korg) cluster at €300–350. |
| 07 | ![](figures/07_brand_evolution_over_time.png) | Roland and Korg maintain the most stable volume over time. All brands drop sharply after late 2022 in line with reduced scraping frequency. |
| 08 | ![](figures/08_brand_time_heatmap.png) | The 2022-08-30 session shows all named brands at 0 while "-" spikes to 760 — a confirmed brand-extraction failure on that date. Roland is the most consistently present brand across all sessions. |

---

## Geographic Analysis

> Listing volume and price distribution across Spanish cities.

| # | Chart | Key insight |
|---|-------|-------------|
| 09 | ![](figures/09_top_cities_by_listings.png) | Madrid (~1,450) and Barcelona (~1,040) account for ~42% of all listings. Girona is a disproportionate third — possibly a province-level labelling artefact. |
| 10 | ![](figures/10_median_price_per_city.png) | Inverse volume-price relationship: high-volume cities (Madrid ~€215, Barcelona ~€230) show the *lowest* median prices; low-volume cities (Tarragona ~€390, Sevilla ~€370) show the *highest*. Competition compresses prices in large markets. |
| 11 | ![](figures/11_city_evolution_over_time.png) | All top cities follow the same pattern: peak late 2022, sharp drop early 2023, partial recovery. Girona collapses to near zero after early 2023 and does not recover — likely a data capture issue. |

---

## Engagement

> Analysis of the `seen` (views) variable as a passive demand proxy.

| # | Chart | Key insight |
|---|-------|-------------|
| 12 | ![](figures/12_seen_distribution.png) | Pareto-like engagement: most listings receive 100–300 views; a small minority reaches 4,000–6,000+. Engagement is highly concentrated. |
| 13 | ![](figures/13_price_vs_seen_scatter.png) | No strong price–views relationship. Dense low-price cluster dominates. High-priced items attract fewer but more targeted views. Listings at price = 0 often attract high traffic (curiosity effect). |
| 14 | ![](figures/14_median_seen_per_brand.png) | Premium brands generate higher median view counts, consistent with their aspirational appeal among the Hispasonic community. |

---

## Correlations and Cross-Group Analysis

> Numeric correlation structure and combined brand × city × type breakdowns.

| # | Chart | Key insight |
|---|-------|-------------|
| 15 | ![](figures/15_correlation_matrix.png) | `price` ↔ `seen` = +0.37 (most meaningful pair). `sell` ↔ `buy`/`change` strongly negative by construction (mutually exclusive flags). Most pairs near zero — low multicollinearity. |
| 16 | ![](figures/16_brand_city_heatmap.png) | Madrid dominates every brand column. Elektron is notably concentrated in large cities. Behringer has broad but modest national presence. |
| 17 | ![](figures/17_price_by_brand_and_type.png) | Behringer "Change" listings show an anomalous median of ~€1,875 — likely trade-up dynamics (bundle of Behringer items exchanged for a single high-end instrument). Elektron leads sell-type median at ~€620. |
| 18 | ![](figures/18_price_evolution_by_brand.png) | Korg commands the highest and most consistent median price (~€300–550 €). Roland is the most price-stable brand. Yamaha is the most volatile. Price = 0 artefact visible for all brands in Dec 2022 – Jun 2023. |

---

## Supply & Demand Analysis

> Generated by [`notebooks/03_supply_demand.ipynb`](../notebooks/03_supply_demand.ipynb)
> **Definitions:** Supply = `sell == 1` · Demand = `buy == 1` or `search == 1` · D/S ratio = demand / supply × 100

| # | Chart | Key insight |
|---|-------|-------------|
| 19 | ![](figures/19_supply_vs_demand_by_city.png) | Supply bars dominate in every city. Demand is nearly invisible at this scale — global D/S ratio is 4.42%. |
| 20 | ![](figures/20_sd_ratio_ranking_by_city.png) | Every city is a buyer's market. No city reaches the 100% balanced line. Ciudad Real tops the ranking at 57% due to a near-zero supply denominator. |
| 21 | ![](figures/21_sd_ratio_vs_price_scatter.png) | Pearson r = −0.291, p = 0.0529. The central hypothesis is not confirmed: higher D/S ratio does not lead to higher prices. |
| 22 | ![](figures/22_sd_ratio_and_price_dual_axis.png) | Price and ratio move independently. Bizkaia has the lowest ratio yet one of the highest median prices — instrument mix drives price, not local demand. |
| 23 | ![](figures/23_supply_demand_over_time_by_city.png) | The demand line is flat near zero for all cities throughout 2022–2024. Girona's supply collapsed entirely after early 2023. |
| 24 | ![](figures/24_sd_ratio_evolution_by_city.png) | All cities stay in the 0–15% band. Girona shows a spike to 100% in Jul 2023 — a zero-denominator artefact, not a real demand event. |
| 25 | ![](figures/25_global_supply_demand_ratio_price.png) | Supply halved in Jan 2023 and never recovered. Mean D/S ratio: 3.9%. Price anomalies in early 2023 are data sparsity artefacts. |
| 26 | ![](figures/26_seen_vs_price_by_city.png) | Pearson r = −0.003, p = 0.985. Views have zero predictive power over price at city level. |
| 27 | ![](figures/27_lagged_correlation_sd_ratio_vs_price.png) | Valencia is the only city with a statistically significant lagged correlation (r = 0.819, p = 0.002). All other 27 cities are not significant. |
