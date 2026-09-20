# MLB Statcast Analytics Pipeline on Databricks

A end-to-end data pipeline and dashboard built on **Databricks Free Edition**, ingesting MLB Statcast pitch-by-pitch data and transforming it into analyst-ready tables using the **Medallion Architecture** (Bronze → Silver → Gold).

## Motivation

This project was built to gain hands-on experience with the core Databricks toolset (Delta Lake, PySpark, Lakeview Dashboards, Genie) using a real-world, high-volume dataset (MLB Statcast), with the goal of building a portfolio piece relevant to data analyst / data engineer roles.

## Architecture

```
pybaseball (Statcast API)
        │
        ▼
┌───────────────┐   raw, unmodified, string-typed
│  Bronze layer │   mlb.bronze.statcast_raw
└───────┬───────┘
        │  type casting, null/outlier filtering, dedup
        ▼
┌───────────────┐   analysis-ready, cleaned
│  Silver layer │   mlb.silver.statcast_clean
└───────┬───────┘
        │  groupBy aggregations
        ▼
┌───────────────┐   pitcher/batter summaries + player names
│  Gold layer   │   mlb.gold.pitcher_summary_named
│               │   mlb.gold.batter_summary_named
│               │   mlb.gold.pitcher_category_summary
└───────┬───────┘
        ▼
┌───────────────────────┐
│  Lakeview Dashboard    │  Pitcher sheet / Batter sheet
│  (+ Genie NL queries)  │
└───────────────────────┘
```

## Tech Stack

- **Platform**: Databricks Free Edition (serverless compute, Unity Catalog)
- **Data source**: [pybaseball](https://github.com/jldbc/pybaseball) (Statcast pitch-level data)
- **Processing**: PySpark, Delta Lake
- **Visualization**: Databricks Lakeview Dashboards, Genie (natural-language query generation)
- **Language**: Python, SQL

## Pipeline Details

### Bronze — Raw ingestion
Statcast data is pulled via `pybaseball.statcast()` and written to Delta **as-is, with all columns cast to string**. This avoids type-inference failures on pandas' mixed NaN/object columns, and keeps the Bronze layer faithful to the source — any reinterpretation of types happens explicitly downstream, not implicitly on ingest.

### Silver — Cleaning
- Casts key columns (`game_date`, `release_speed`, `launch_speed`, `launch_angle`, `release_spin_rate`, etc.) to their proper types
- Deduplicates on the natural pitch-level key: `game_pk` + `at_bat_number` + `pitch_number`
- Filters out null/out-of-range values (e.g. `release_speed` outside 30–110 mph)
- Normalizes inconsistent pitch-type labels (e.g. `FT`→`SI`, `FA`→`FF`)
- Adds a derived `pitch_category` column, grouping raw Statcast pitch codes into `Fastball` / `Breaking Ball` / `Offspeed` / `Other`

### Gold — Aggregation
- `pitcher_summary` / `batter_summary`: per-player aggregates (avg exit velocity, avg spin rate, pitch counts, etc.), split by pitcher and batter perspective
- `pitcher_category_summary`: per-pitcher aggregates by `pitch_category`
- `player_lookup`: MLBAM ID → player name mapping via `pybaseball.playerid_reverse_lookup`, joined into the summary tables (`*_named`) via left join so unmatched IDs don't drop rows

## Dashboard

The dashboard is split into two pages to reflect the two natural analytical perspectives in baseball data:

- **Pitcher sheet**: pitch count / pitch mix (`pitch_category` breakdown) / spin rate vs. release speed scatter / velocity leaderboard
- **Batter sheet**: exit velocity leaderboard / launch angle distribution / batted-ball count

All ranking widgets apply a minimum sample-size filter (e.g. `batted_ball_count >= 20`) to avoid small-sample outliers dominating the leaderboards.

Several widgets were built using **Genie** (natural-language to visualization), with explicit prompts specifying aggregation, chart type, axis, sort order, and formatting to reduce ambiguity.

## Key Design Decisions

| Decision | Reasoning |
|---|---|
| String-typed Bronze layer | Keeps raw ingestion resilient to upstream schema/type drift; all typing decisions live in Silver |
| Medallion architecture | Industry-standard pattern; separates raw storage, cleaning, and business-level aggregation into auditable layers |
| Left join for player names | Preserves aggregate rows even when a player ID can't be resolved to a name |
| Minimum sample-size filters | Prevents statistically unreliable results (e.g. 1 batted ball) from dominating rankings |
| English category labels | Keeps the dashboard usable for non-Japanese-speaking reviewers |

## Screenshots

_(add `docs/dashboard_pitcher.png` and `docs/dashboard_batter.png` here)_

## Future Work

- Automate the Bronze→Silver→Gold pipeline as a scheduled Databricks Job
- Add a batted-ball outcome prediction model with MLflow experiment tracking
- Extend to a full-season dataset with incremental (rather than full-overwrite) loads
