# Reports — EDA Figures

Charts generated automatically by [`notebooks/02_eda.ipynb`](../notebooks/02_eda.ipynb).
Run the notebook to regenerate all figures.

---

## Temporal evolution

| Chart | Description |
|-------|-------------|
| ![](figures/01_listing_volume_over_time.png) | Listing volume per scrape date |
| ![](figures/02_price_evolution_over_time.png) | Median and mean sell price over time (with IQR band) |
| ![](figures/03_listing_type_share_over_time.png) | Sell / Buy / Change / Search share over time |

---

## Price analysis

| Chart | Description |
|-------|-------------|
| ![](figures/04_price_distribution.png) | Price histogram and boxplot (sell listings, ≤p99) |

---

## Brand analysis

| Chart | Description |
|-------|-------------|
| ![](figures/05_top_brands_by_listings.png) | Top 15 brands by listing count |
| ![](figures/06_median_price_per_brand.png) | Median sell price per brand |
| ![](figures/07_brand_evolution_over_time.png) | Top 8 brands — listing volume over time |
| ![](figures/08_brand_time_heatmap.png) | Brand × scrape date heatmap |

---

## Geographic analysis

| Chart | Description |
|-------|-------------|
| ![](figures/09_top_cities_by_listings.png) | Top 15 cities by listing count |
| ![](figures/10_median_price_per_city.png) | Median sell price per city |
| ![](figures/11_city_evolution_over_time.png) | Top 6 cities — listing volume over time |

---

## Engagement

| Chart | Description |
|-------|-------------|
| ![](figures/12_seen_distribution.png) | Views distribution |
| ![](figures/13_price_vs_seen_scatter.png) | Price vs views scatter |
| ![](figures/14_median_seen_per_brand.png) | Median views per brand |

---

## Correlations and cross-group analysis

| Chart | Description |
|-------|-------------|
| ![](figures/15_correlation_matrix.png) | Correlation matrix — all numeric variables |
| ![](figures/16_brand_city_heatmap.png) | Brand × City listing count heatmap |
| ![](figures/17_price_by_brand_and_type.png) | Median price by brand and listing type |
| ![](figures/18_price_evolution_by_brand.png) | Median sell price over time — top 5 brands |
