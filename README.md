# Hispasonic Web Scraping & Data Analysis

[![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Pandas](https://img.shields.io/badge/Pandas-Data_Analysis-150458?style=flat-square&logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![SQLite](https://img.shields.io/badge/SQLite-Database-003B57?style=flat-square&logo=sqlite&logoColor=white)](https://sqlite.org)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

End-to-end solution for automated web scraping and data analysis from [Hispasonic](https://www.hispasonic.com/), a Spanish-speaking music technology community.

---

## Features

- **Ethical Web Scraping** - Rate-limited extraction of 10,000+ posts using BeautifulSoup
- **SQLite Storage** - Structured database with data integrity validation
- **EDA & Analytics** - User activity trends, engagement metrics, topic popularity
- **Reproducible** - Documented Jupyter notebooks for every stage

---

## Tech Stack

| Category | Tools |
|----------|-------|
| Scraping | `requests`, `BeautifulSoup` |
| Data | `pandas`, `numpy`, `SQLite` |
| Analysis | `Jupyter Notebooks` |

---

## Quick Start

```bash
# Clone
git clone https://github.com/albertjimrod/hispasonic.git
cd hispasonic

# Install dependencies
pip install -r requirements.txt

# Run notebooks in hispaok/
jupyter notebook
```

---

## Project Structure

```
hispasonic/
├── hispaok/
│   ├── 01_from_web_to_csv_togit.ipynb  # Scraping notebook
│   ├── csv/                             # Raw scraped CSVs (source)
│   └── images/                          # Scraping visualizations
├── data/
│   ├── raw/                             # 12 source CSVs (2022–2024)
│   └── processed/                       # Unified dataset (gitignored)
├── notebooks/
│   ├── 01_etl.ipynb                     # ETL — load, normalize, unify all CSVs
│   ├── 02_eda.ipynb                     # EDA — temporal, price, brand, city analysis
│   └── 03_supply_demand.ipynb           # Supply & demand analysis by city and price
├── reports/
│   └── figures/                         # Auto-generated charts from EDA
├── requirements.txt
└── README.md
```

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Rate limiting | Request delays + user-agent rotation |
| HTML structure changes | Modular design for quick fixes |
| Missing data | Validation & preprocessing pipeline |

---

## Author

**Alberto Jiménez** - [datablogcafe.com](https://datablogcafe.com) | [GitHub](https://github.com/albertjimrod)

---

## License

MIT License - feel free to use and modify.
