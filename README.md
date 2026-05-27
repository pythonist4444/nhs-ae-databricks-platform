# NHS A&E Performance Platform

A production-grade data engineering platform built entirely on **Microsoft Fabric**, ingesting, transforming, and analysing NHS England A&E waiting time statistics across 229 trusts and 36 months.

![Pipeline](https://img.shields.io/badge/pipeline-Bronze%20→%20Silver%20→%20Gold-blue)
![Stack](https://img.shields.io/badge/stack-Microsoft%20Fabric-purple)
![Data](https://img.shields.io/badge/data-NHS%20England-blue)

---

## Architecture

```
NHS England (public monthly CSV data)
              │
              ▼
┌─────────────────────────────────────────────────────────┐
│                Microsoft Fabric — OneLake                │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │                   Lakehouse                       │  │
│  │                                                   │  │
│  │  ┌──────────┐   ┌──────────┐   ┌─────────────┐  │  │
│  │  │  Bronze  │──▶│  Silver  │──▶│    Gold     │  │  │
│  │  │  (raw)   │   │(cleaned) │   │  (metrics)  │  │  │
│  │  └──────────┘   └──────────┘   └─────────────┘  │  │
│  │       │               │               │          │  │
│  │  Spark notebook  Spark notebook  Spark notebook  │  │
│  │  (ingest)        (transform)     (dim + facts)   │  │
│  └───────────────────────────────────────────────────┘  │
│                          │                              │
│              Fabric SQL Analytics Endpoint              │
│                          │                              │
│                   Power BI Report                       │
└─────────────────────────────────────────────────────────┘
```

---

## Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Platform | Microsoft Fabric | Unified lakehouse platform |
| Storage | OneLake | Single storage layer across all medallion zones |
| Compute | Fabric Spark (PySpark) | Notebook-based ingestion and transformation |
| Format | Delta Lake | ACID-compliant table format across all layers |
| Serving | Fabric SQL Analytics Endpoint | SQL-queryable Gold layer |
| Visualisation | Power BI (Fabric-native) | Semantic model and interactive reports |
| Version Control | GitHub | Notebook versioning and project documentation |

---

## Medallion Architecture

| Layer | Table | Rows | Description |
|-------|-------|------|-------------|
| Bronze | `bronze_ae_monthly` | 7,405 | Raw CSVs as-landed — all strings, immutable |
| Silver | `silver_ae_monthly` | 7,369 | Cleaned, typed, validated trust data |
| Silver | `silver_ae_rejected_rows` | 36 | TOTAL summary rows excluded from analysis |
| Gold | `gold_dim_trust` | 229 | One row per NHS trust — org code, name, parent |
| Gold | `gold_fct_ae_monthly` | 6,890 | One row per trust per month with all KPIs |

---

## Data Source

**NHS England — A&E Waiting Times and Activity**

Monthly statistics covering all NHS trusts in England:
- Total A&E attendances (Type 1, Type 2, Other, Booked)
- Attendances seen within 4 hours (4-hour target metric)
- Patients waiting 4–12 hours and 12+ hours from DTA (breach metrics)
- Emergency admissions via A&E

Source: [england.nhs.uk — A&E Waiting Times and Activity](https://www.england.nhs.uk/statistics/statistical-work-areas/ae-waiting-times-and-activity/)

Coverage: April 2021 – March 2024 (36 monthly files across 3 financial years)

---

## Key Metrics (Gold Layer)

| Metric | Column | Definition |
|--------|--------|-----------|
| 4-hour target % | `four_hr_target_pct` | (Total attendances − Over 4hrs) ÷ Total attendances × 100 |
| 12-hour breach rate | `twelve_hr_breach_rate` | 12+ hr DTA waits ÷ Total attendances × 100 |
| MoM attendance change | `mom_attendance_change_pct` | Month-on-month % change in total attendances |
| MoM 4hr target change | `mom_4hr_target_change` | Month-on-month change in 4-hour target % |

---

## Project Structure

```
nhs-ae-fabric-platform/
├── notebooks/
│   ├── 00_onelake_connect_test.ipynb   ← validate OneLake connection
│   ├── 01_bronze_ingest.ipynb          ← raw CSVs → Bronze Delta table
│   ├── 02_silver_clean.ipynb           ← Bronze → Silver Delta table
│   └── 03_gold_transforms.ipynb        ← Silver → Gold dim + fact tables
├── docs/
│   └── architecture.md                 ← detailed design decisions
└── data/
    └── raw/                            ← local CSVs (gitignored)
```

---

## Running the Pipeline

### Prerequisites
- Microsoft Fabric workspace (free 60-day trial at [app.fabric.microsoft.com](https://app.fabric.microsoft.com))
- `nhs_ae_lakehouse` Lakehouse created and attached to notebooks
- NHS A&E monthly CSVs uploaded to `Files/bronze/ae_monthly/` organised by year

### Execution order

Run notebooks in sequence inside your Fabric workspace:

1. `00_onelake_connect_test.ipynb` — validate OneLake connection and file listing
2. `01_bronze_ingest.ipynb` — ingest all 36 CSVs recursively → Bronze Delta table
3. `02_silver_clean.ipynb` — clean, cast, validate → Silver Delta table + rejected rows
4. `03_gold_transforms.ipynb` — build dim_trust and fct_ae_monthly → Gold Delta tables

> All notebooks run on Fabric's built-in Spark runtime. No external compute, no credentials, no cost beyond the Fabric trial.

---

## Design Decisions

See [docs/architecture.md](docs/architecture.md) for full rationale covering:
- Why Medallion Architecture
- Why Delta Lake over Parquet
- Why Fabric-native Spark over Azure Databricks
- Data quality approach across all three layers
- What was intentionally excluded and why

---


## Author

**Abdulafeez Fakorede** — Senior Data Engineer  
[LinkedIn](https://www.linkedin.com/in/abdulafeezfakorede/) · [GitHub](https://github.com/pythonist4444)
