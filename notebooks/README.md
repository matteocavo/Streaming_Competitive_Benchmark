# 📓 Notebooks

This folder contains the three-stage data pipeline that transforms raw TMDB API data into the analytical mart consumed by the Power BI dashboard.

---

## Pipeline Overview

```
TMDB API
    ↓
01_ingestion.ipynb     → data/raw/
    ↓
02_cleaning.ipynb      → data/clean/
    ↓
03_build_mart.ipynb    → data/mart/
    ↓
Power BI Dashboard
```

---

## 01_ingestion.ipynb — Data Collection

Collects catalog data from the TMDB API for all provider-market combinations.

**Inputs:** TMDB API key (environment variable `TMDB_API_KEY`)

**Logic:**
- Iterates over a 5 providers × 7 markets grid (35 combinations)
- Queries `/discover/movie` and `/discover/tv` endpoints with provider watch region filters
- Handles pagination (up to 500 pages per query)
- Applies rate limiting to respect TMDB API quotas

**Output:** `data/raw/` — one Parquet file per provider/market/media_type combination

---

## 02_cleaning.ipynb — Data Preparation

Transforms raw API responses into a normalized, deduplicated dataset.

**Inputs:** `data/raw/`

**Logic:**
- Flattens nested JSON fields (genres, origin countries, production companies)
- Normalizes provider names to canonical form (e.g. `"amazon_prime"` → `"Amazon Prime Video"`)
- Deduplicates titles by `(tmdb_id, media_type)` and availability by `(tmdb_id, media_type, provider_id, country)`; provider IDs are then mapped to a canonical name, which leaves duplicate rows for Paramount Plus US
- Standardizes origin country codes to ISO 3166-1 alpha-2
- Assigns recency buckets:
  - `Recent` — release year ≥ 2021
  - `Mid` — release year 2012–2020
  - `Legacy` — release year ≤ 2011
- Computes `vote_weight = log(1 + vote_count)` per title. It is not used by the mart's `weighted_rating`.

> **Note on `weighted_rating` (notebook 03):** the mart computes `sum(vote_average × vote_count) / sum(vote_count)` per provider–market pair over rows with a vote count and an average. Titles with `vote_count = 0` carry zero weight and are excluded in effect; they are not scored as 0. The raw vote count is the weight, so heavily voted titles dominate. This is a popularity-weighted rating and has not been validated as a measure of catalog quality.

**Output:** `data/clean/titles_clean.parquet`, `data/clean/provider_availability_clean.parquet`

---

## 03_build_mart.ipynb — Analytical Mart Construction

Builds the three analytical tables used by the Power BI dashboard.

**Inputs:** `data/clean/`

**Output tables:**

### provider_catalog_mart
Title-level grain: one row per `(tmdb_id, canonical_provider, country)`.

| Column | Type | Description |
|---|---|---|
| `tmdb_id` | int | TMDB unique identifier |
| `media_type` | str | `movie` or `tv` |
| `canonical_provider` | str | Normalized provider name |
| `country` | str | Market (ISO 3166-1 alpha-2) |
| `title` | str | Title name |
| `release_year` | int | Year of release |
| `recency_bucket` | str | Legacy / Mid / Recent |
| `primary_genre` | str | Dominant genre |
| `genres` | str | Full genre list |
| `origin_country` | str | Production country |
| `original_language` | str | Original language code |
| `vote_average` | float | TMDB rating |
| `vote_count` | int | Number of votes |
| `popularity` | float | TMDB popularity score |
| `runtime_min` | float | Runtime in minutes |

### provider_market_metrics
Provider × market grain: one row per `(canonical_provider, country)`.

| Column | Type | Description |
|---|---|---|
| `canonical_provider` | str | Provider name |
| `country` | str | Market |
| `catalog_size` | int | Total titles |
| `unique_titles` | int | Deduplicated titles |
| `recent_pct` | float | Share of Recent content |
| `recency_intensity_score` | float | Vote-count weighted recency composite |
| `genre_concentration_idx` | float | Herfindahl index on primary genre |
| `intl_diversification_score` | float | Non-local % × log(distinct countries) |
| `weighted_rating` | float | Avg quality-adjusted rating |

### overlap_matrix
Provider-pair grain: one row per `(country, provider_a, provider_b)`.

| Column | Type | Description |
|---|---|---|
| `country` | str | Market |
| `provider_a` | str | First provider |
| `provider_b` | str | Second provider |
| `intersection` | int | Shared titles (by tmdb_id) |
| `union` | int | Total unique titles across both |
| `jaccard` | float | Jaccard similarity coefficient |
| `titles_a` | int | Catalog size of provider_a |
| `titles_b` | int | Catalog size of provider_b |

---

## Setup

```bash
python -m pip install requests pandas numpy pyarrow jupyter
export TMDB_API_KEY="your_api_key_here"
```

Run notebooks in order:

```bash
jupyter notebook 01_ingestion.ipynb
jupyter notebook 02_cleaning.ipynb
jupyter notebook 03_build_mart.ipynb
```

> A free TMDB API key is required — register at [themoviedb.org](https://www.themoviedb.org/settings/api).

---

*Matteo Cavo — 2026*
