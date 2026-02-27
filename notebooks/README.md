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
- Deduplicates by `tmdb_id` within each provider-market pair
- Standardizes origin country codes to ISO 3166-1 alpha-2
- Assigns recency buckets:
  - `Recent` — release year ≥ 2023
  - `Mid` — release year 2020–2022
  - `Legacy` — release year ≤ 2019
- Computes weighted rating: `vote_average × log(vote_count + 1)` — stored as integer ×10,000 for Power BI precision compatibility

> **Why log?** A logarithmic transform on `vote_count` attenuates the influence of titles with extremely high vote counts (blockbusters with 100K+ votes) without discarding low-voted titles entirely. This avoids the arbitrary prior required by Bayesian/IMDb-style weighted means while still penalizing titles with near-zero votes. The result is a quality signal that scales more fairly across catalog sizes as different as Apple TV+ (~2K titles) and Netflix (~61K titles).
>
> **Note on vote_count = 0:** Titles with no votes yield `log(0 + 1) = 0`, resulting in `weighted_rating = 0`. These titles are not excluded from the mart but are effectively penalized to zero — the quality metrics computed in the dashboard reflect this, with aggregate averages pulled down by unrated titles in proportion to their share of the catalog.

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
pip install -r requirements.txt
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
