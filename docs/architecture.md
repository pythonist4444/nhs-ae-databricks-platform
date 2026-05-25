# Architecture — NHS A&E Performance Platform

## Overview

This platform processes NHS England A&E waiting time statistics through a three-layer Medallion Architecture, built entirely on **Microsoft Fabric**. All compute runs on Fabric's built-in Spark runtime (PySpark notebooks), with OneLake as the unified storage layer and dbt-fabric handling the Gold SQL transformation layer.

---

## Why Medallion Architecture?

Medallion Architecture (Bronze → Silver → Gold) separates concerns cleanly:

- **Bronze** is an immutable copy of the source. If downstream processing has a bug, the raw data is always there to reprocess from. Files in bronze are never modified after landing.
- **Silver** is cleaned and standardised. Trust codes are normalised, columns renamed to snake_case, date columns cast correctly, and invalid rows written to a separate rejected_rows table rather than silently dropped.
- **Gold** is business-ready. dbt models in the Gold layer define the KPIs, join dimensions to facts, and apply business logic (4-hour target calculation, breach rates). Only Gold is exposed to consumers.

---

## Why Microsoft Fabric + OneLake?

OneLake acts as a single unified storage layer — no separate Azure Data Lake Storage account, no storage account keys, no service principals to manage. The Fabric Lakehouse provides a file explorer, Delta table registration, a SQL endpoint, and a built-in Spark runtime all from one workspace.

Fabric's built-in Spark runtime means notebooks connect to OneLake natively using simple relative paths (`Files/bronze/ae_monthly/`) rather than `abfss://` URIs with credential configuration. This removes an entire layer of infrastructure complexity without sacrificing any PySpark or Delta Lake capability.

For a portfolio project, Fabric's 60-day free trial (F64 capacity) eliminates infrastructure cost entirely while demonstrating a platform that is actively adopted across UK public sector organisations.

---

## Why Fabric Spark over External Compute?

Azure Databricks was considered but ruled out for this project for two reasons:

1. **Cost** — Databricks clusters on Azure incur compute charges even at minimum size. Fabric's built-in Spark is included in the trial with no additional cost.
2. **Integration** — Fabric Spark connects to OneLake without any credential configuration. External Databricks would require service principal setup, secret scope management, and `abfss://` path configuration — adding infrastructure complexity that doesn't contribute to the data engineering goals of this project.

In a production NHS trust environment with an existing Azure Databricks contract, external Databricks with Unity Catalog would be the appropriate choice. That trade-off is acknowledged in the "What's Not in This Project" section below.

---

## Why Delta Lake (not Parquet)?

Delta Lake gives ACID transactions on top of Parquet files. In practice for this project this means:

- If the Silver cleaning notebook fails halfway through writing, the partial write is rolled back — no corrupt Silver table
- Time travel lets us query previous versions of the data (useful when NHS England publishes revised monthly figures)
- Schema enforcement catches unexpected column changes in new monthly files before they propagate downstream
- `OPTIMIZE` and `VACUUM` commands keep table performance consistent as data volume grows

---

## Why dbt for the Gold Layer?

The Silver layer is written by PySpark notebooks. Anything more complex — multi-table joins, KPI definitions, dimension modelling — is easier to write, test, and document in SQL via dbt. Specifically:

- Every model has `not_null`, `unique`, and `relationships` tests defined in YAML — broken data surfaces immediately
- `dbt docs generate` produces a queryable data catalogue published automatically to GitHub Pages via CI
- The separation between PySpark (heavy transformation) and dbt (business logic) mirrors how production data teams split responsibilities between data engineers and analytics engineers

---

## How Fabric Spark Notebooks Connect to OneLake

Fabric notebooks use `mssparkutils.fs` for file operations and `spark.read` for data access. Paths are relative to the attached Lakehouse — no credentials, no mount points, no configuration:

```python
# List files
mssparkutils.fs.ls("Files/bronze/ae_monthly/2021-22/")

# Read CSVs
df = spark.read.option("header", "true").csv("Files/bronze/ae_monthly/")

# Write Delta table
df.write.format("delta").mode("overwrite").save("Tables/silver_ae_monthly")
```

The Lakehouse is attached to the notebook via the Fabric UI (Add lakehouse → select nhs_ae_lakehouse). Authentication is handled automatically by the Fabric workspace identity.

---

## Data Quality Approach

Three layers of quality checks:

1. **Notebook-level** (Bronze ingest): schema validation, row count checks, null rate logging per column
2. **Notebook-level** (Silver clean): reject rows that fail trust code format or date casting — written to a separate `rejected_rows` Delta table rather than silently dropped
3. **dbt-level** (Gold): `not_null`, `unique`, `relationships`, and custom `accepted_values` tests on every mart model — all run on every PR via GitHub Actions

---

## What's Not in This Project (and Why)

| Feature | Why excluded |
|---------|-------------|
| Unity Catalog | Azure Databricks-only feature; not available in Fabric Spark. Would be the correct governance layer in a production Azure Databricks deployment. |
| Airflow orchestration | Out of scope — see Project 4 (Companies House ETL) for Airflow DAG patterns. |
| Power BI semantic model | Covered in Project 3 (Fabric Lakehouse — MHCLG data) which focuses on the full Fabric BI stack including Dataflow Gen2 and semantic models. |
| Real-time ingestion | NHS A&E data is monthly batch — a streaming architecture would be inappropriate and over-engineered for this use case. |
| Azure Databricks | Replaced by Fabric's built-in Spark for cost and integration reasons. See "Why Fabric Spark over External Compute" above. |
