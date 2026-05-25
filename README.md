# NHS A&E Performance Platform

A production-grade data engineering platform that ingests, transforms, and analyses NHS England A&E waiting time statistics using **Microsoft Fabric**, **Databricks**, and **dbt**.

![CI](https://github.com/YOUR_USERNAME/nhs-ae-databricks-platform/actions/workflows/ci.yml/badge.svg)

---

## Architecture

```
NHS England (public CSV data)
        │
        ▼
┌─────────────────────────────────────────────────────┐
│              Microsoft Fabric (OneLake)              │
│                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐   │
│  │  Bronze  │───▶│  Silver  │───▶│     Gold     │   │
│  │  (raw)   │    │(cleaned) │    │ (dbt models) │   │
│  └──────────┘    └──────────┘    └──────────────┘   │
│        │                │                           │
│   Databricks        Databricks           dbt-fabric │
│   (ingest)          (PySpark)         (dim + facts)  │
└─────────────────────────────────────────────────────┘
        │
        ▼
  Fabric SQL Endpoint
  (queryable via Power BI / SQL)
```

### Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Storage | Microsoft Fabric · OneLake | Unified lakehouse storage (Bronze/Silver/Gold) |
| Compute | Azure Databricks (in Fabric) | PySpark ingestion and transformation |
| Format | Delta Lake | ACID-compliant table format across all layers |
| Transform | dbt-fabric | SQL models, tests, and documentation (Gold layer) |
| CI/CD | GitHub Actions | dbt test on PR · dbt build on merge · docs deploy |
| Docs | dbt docs (GitHub Pages) | Auto-published data documentation |

---

## Project Structure

```
nhs-ae-databricks-platform/
├── notebooks/
│   ├── 00_onelake_connect_test.ipynb   ← validate OneLake connection
│   ├── 01_bronze_ingest.ipynb          ← raw CSV → Bronze Delta table
│   └── 02_silver_clean.ipynb           ← Bronze → Silver Delta table
├── dbt/nhs_ae/
│   ├── models/
│   │   ├── staging/                    ← stg_ae_monthly
│   │   ├── intermediate/               ← int_ae_monthly_cleaned
│   │   └── marts/                      ← dim_trust, fct_ae_monthly
│   └── tests/                          ← custom data quality tests
├── docs/
│   └── architecture.md                 ← detailed design decisions
└── data/raw/                           ← local CSVs (gitignored)
```

---

## Data Source

**NHS England — A&E Waiting Times and Activity**
Monthly data covering all NHS trusts in England, including:
- Total A&E attendances
- Attendances seen within 4 hours (4-hour target)
- 12-hour trolley waits (breach metric)
- Emergency admissions

Source: [england.nhs.uk/statistics/statistical-work-areas/ae-waiting-times-and-activity](https://www.england.nhs.uk/statistics/statistical-work-areas/ae-waiting-times-and-activity/)

Coverage: 2021–22 · 2022–23 · 2023–24

---

## Key Metrics (Gold Layer)

| Metric | Definition |
|--------|-----------|
| `four_hr_target_pct` | Attendances seen within 4 hrs ÷ total attendances |
| `twelve_hr_breach_rate` | 12-hr trolley waits ÷ total attendances |
| `mom_attendance_change` | Month-on-month % change in total attendances |
| `mom_target_change` | Month-on-month change in 4-hr target performance |

---

## Getting Started

### Prerequisites

- Microsoft Fabric workspace (free 60-day trial at [app.fabric.microsoft.com](https://app.fabric.microsoft.com))
- Python 3.9+
- dbt-fabric: `pip install dbt-fabric`

### dbt setup

```bash
cd dbt/nhs_ae
cp profiles.yml.example profiles.yml
# Edit profiles.yml with your Fabric SQL endpoint details
dbt debug       # confirm connection
dbt deps        # install packages
dbt run         # build all models
dbt test        # run all tests
dbt docs generate && dbt docs serve   # view docs locally
```

### Running notebooks

Open notebooks in order inside your Databricks environment connected to Fabric:

1. `00_onelake_connect_test.ipynb` — validate connection to OneLake
2. `01_bronze_ingest.ipynb` — ingest raw CSVs to Bronze Delta table
3. `02_silver_clean.ipynb` — clean and standardise to Silver Delta table

---

## Design Decisions

See [docs/architecture.md](docs/architecture.md) for full rationale on:
- Why Medallion Architecture
- Why Delta Lake over Parquet
- Why dbt for the Gold layer
- How Fabric + Databricks integration works
- Data quality approach

---

## Demo

🎥 [Watch the 2-minute walkthrough](#) ← add your YouTube link here

---

## Author

**Abdulafeez Fakorede** — Senior Data Engineer  
[LinkedIn](#) · [GitHub](#)
