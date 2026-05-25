# Architecture — NHS A&E Performance Platform

## Overview

This platform processes NHS England A&E waiting time statistics through a three-layer Medallion Architecture, combining Microsoft Fabric (OneLake lakehouse storage), Azure Databricks (PySpark transformation), and dbt (SQL modelling and documentation).

---

## Why Medallion Architecture?

Medallion Architecture (Bronze → Silver → Gold) separates concerns cleanly:

- **Bronze** is an immutable copy of the source. If downstream processing has a bug, the raw data is always there to reprocess from. Files in bronze are never modified after landing.
- **Silver** is cleaned and standardised. Schema drift between years of NHS data is resolved here, trust codes are normalised, and columns are renamed consistently. This layer is trusted for ad-hoc analysis.
- **Gold** is business-ready. dbt models in the Gold layer define the KPIs, join dimensions to facts, and apply business logic (4-hour target calculation, breach rates). Only Gold is exposed to consumers.

---

## Why Microsoft Fabric + OneLake?

OneLake acts as a single unified storage layer — no separate Azure Data Lake Storage account, no storage account keys to manage. The Fabric Lakehouse UI provides a file explorer, Delta table registration, and a SQL endpoint all from one place.

For a portfolio project, Fabric's 60-day free trial (F64 capacity) eliminates infrastructure cost entirely while demonstrating a platform that is actively adopted across UK public sector organisations.

---

## Why Delta Lake (not Parquet)?

Delta Lake gives ACID transactions on top of Parquet files. In practice for this project this means:

- If the Silver cleaning notebook fails halfway through writing, the partial write is rolled back — no corrupt Silver table
- Time travel lets us query previous versions of the data (useful when NHS data revisions are published)
- Schema enforcement catches unexpected column changes in new monthly files before they propagate downstream

---

## Why dbt for the Gold Layer?

The Silver layer is written by PySpark notebooks. Anything more complex — multi-table joins, KPI definitions, dimension modelling — is easier to write, test, and document in SQL via dbt. Specifically:

- Every model has `not_null`, `unique`, and `relationships` tests defined in YAML — broken data surfaces immediately
- `dbt docs generate` produces a queryable data catalogue published automatically to GitHub Pages via CI
- The separation between PySpark (heavy transformation) and dbt (business logic) mirrors how production data teams split responsibilities between data engineers and analytics engineers

---

## How Fabric + Databricks Integration Works

Databricks notebooks run inside the Fabric workspace with OneLake as the backing storage. The notebook reads from and writes to `abfss://` paths on OneLake, which are also visible as files and Delta tables in the Fabric Lakehouse explorer.

Authentication uses the Fabric workspace identity — no service principals or storage account keys are required in the notebooks for local development.

---

## Data Quality Approach

Three layers of quality checks:

1. **Notebook-level** (Bronze ingest): schema validation, row count checks, null rate logging per column
2. **Notebook-level** (Silver clean): reject rows that fail trust code format, date cast failures written to a separate `rejected_rows` table
3. **dbt-level** (Gold): `not_null`, `unique`, `relationships`, and custom `accepted_values` tests on every mart model — all run on every PR via GitHub Actions

---

## What's Not in This Project (and Why)

| Feature | Why excluded |
|---------|-------------|
| Unity Catalog | Azure Databricks-only; not available when Databricks runs inside Fabric. Would be added in a full Azure deployment. |
| Airflow orchestration | Out of scope for this project — see Project 4 (Companies House) for Airflow patterns. |
| Power BI semantic model | Covered in Project 3 (Fabric Lakehouse — MHCLG data) which focuses on the full Fabric BI stack. |
| Real-time ingestion | NHS A&E data is monthly batch — streaming would be inappropriate. |
