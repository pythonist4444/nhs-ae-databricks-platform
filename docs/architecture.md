# Architecture — NHS A&E Performance Platform

## Overview

This platform processes NHS England A&E waiting time statistics through a three-layer Medallion Architecture built entirely on **Microsoft Fabric**. All compute runs on Fabric's built-in Spark runtime (PySpark notebooks), OneLake serves as the unified storage layer across all three medallion zones, and Power BI is connected natively via the Fabric SQL Analytics Endpoint.

No external compute, no Azure subscription, no third-party orchestration tools — everything runs inside a single Fabric workspace.

---

## Pipeline Summary

| Stage | Notebook | Input | Output | Rows |
|-------|----------|-------|--------|------|
| Ingest | `01_bronze_ingest` | 36 monthly CSVs | `bronze_ae_monthly` Delta table | 7,405 |
| Clean | `02_silver_clean` | `bronze_ae_monthly` | `silver_ae_monthly` + `silver_ae_rejected_rows` | 7,369 + 36 |
| Transform | `03_gold_transforms` | `silver_ae_monthly` | `gold_dim_trust` + `gold_fct_ae_monthly` | 229 + 6,890 |

---

## Why Medallion Architecture?

Medallion Architecture (Bronze → Silver → Gold) separates concerns across three clearly defined layers:

**Bronze** is an immutable copy of the source exactly as it arrived from NHS England. Every raw CSV file is read with `inferSchema=false` — all columns stored as strings, no casting, no transformation. If any downstream processing introduces a bug, the raw data is always available to reprocess from scratch. Files in the bronze folder are never modified after landing.

**Silver** is where cleaning and standardisation happens. All 22 source columns are renamed to snake_case for Delta Lake compatibility, date columns are parsed from the `MSitAE-MONTH-YEAR` NHS format, numeric columns are cast from string to LongType with suppressed values (`-`, `*`) handled explicitly rather than silently dropped. Invalid rows (the 36 TOTAL summary rows NHS includes at the bottom of each file) are written to a separate `silver_ae_rejected_rows` table rather than discarded — maintaining full auditability.

**Gold** is the business-ready layer. `gold_dim_trust` provides one row per NHS trust with organisation code, name, and parent organisation. `gold_fct_ae_monthly` provides one row per trust per month with all KPI metrics calculated, including month-on-month changes using PySpark window functions.

---

## Why Microsoft Fabric and OneLake?

OneLake acts as a single unified storage layer — no separate Azure Data Lake Storage account, no storage account keys, no service principals to configure. The Fabric Lakehouse provides a file explorer, Delta table registration, a SQL analytics endpoint, and a built-in Spark runtime all within one workspace.

Fabric's built-in Spark runtime connects to OneLake using simple relative paths (`Files/bronze/ae_monthly/`) rather than `abfss://` URIs with credential configuration. This removes an entire layer of infrastructure complexity without sacrificing any PySpark or Delta Lake capability.

Microsoft Fabric is actively being adopted across UK public sector organisations, making it directly relevant to NHS, local government, and central government data engineering roles.

---

## Why Fabric Spark over Azure Databricks?

Azure Databricks was the original plan for this project but was replaced by Fabric's built-in Spark for two reasons:

**Cost** — Databricks clusters on Azure incur compute charges even at minimum size. Fabric's built-in Spark is included in the trial at no additional cost, making the project fully reproducible without any spend.

**Integration simplicity** — Fabric Spark connects to OneLake natively without service principal setup, secret scope management, or `abfss://` path configuration. External Databricks would require all three, adding infrastructure complexity that doesn't contribute to the data engineering goals of this project.

In a production NHS trust environment with an existing Azure Databricks contract and an organisational Azure AD tenant, Databricks with Unity Catalog would be the appropriate choice — providing fine-grained governance, column-level lineage, and cross-workspace data sharing that Fabric's built-in Spark does not currently match. That trade-off is documented here deliberately.

---

## Why Delta Lake over Parquet?

Delta Lake adds four capabilities on top of plain Parquet that matter for this project:

**ACID transactions** — if the Silver cleaning notebook fails halfway through writing, the partial write is rolled back automatically. No corrupt Silver table, no manual cleanup.

**Time travel** — NHS England occasionally revises previously published monthly figures. Delta Lake's versioning means you can query the previous state of the Silver table before a revision was applied.

**Schema enforcement** — Delta rejects writes that don't match the registered schema. If a future NHS A&E file introduces a new column or changes a column type, the pipeline fails loudly at the Bronze write rather than silently propagating a schema change downstream.

**`OPTIMIZE` and `VACUUM`** — keep table read performance consistent as data volume grows across financial years.

---

## Data Quality Approach

Quality checks are applied at all three layers:

**Bronze (notebook-level)** — row count validation, null count per column, schema inspection, and source file tracking via `input_file_name()`. All 36 files are expected to have 22 columns and zero nulls on load.

**Silver (notebook-level)** — numeric columns validated against `^[0-9]+$` regex before casting; non-numeric values (suppressed data markers) set to null explicitly. TOTAL summary rows identified and written to `silver_ae_rejected_rows` rather than silently dropped. `period_date` parsed from `MSitAE-MONTH-YEAR` format with null check to catch any unparseable values.

**Gold (notebook-level)** — zero-attendance rows filtered before KPI calculation to prevent division by zero. Window function ordering validated by checking a single trust across all 36 months to confirm month-on-month lag is working correctly.

---

## KPI Definitions

| Metric | Formula | Notes |
|--------|---------|-------|
| `four_hr_target_pct` | (total_attendances − total_over_4hrs) ÷ total_attendances × 100 | NHS standard target is 95% |
| `twelve_hr_breach_rate` | patient_waited_12plus_hrs_dta ÷ total_attendances × 100 | Key performance concern post-COVID |
| `mom_attendance_change_pct` | (this month − last month) ÷ last month × 100 | Null for April 2021 (first month) |
| `mom_4hr_target_change` | this month four_hr_target_pct − last month four_hr_target_pct | Null for April 2021 (first month) |

---

## What Was Intentionally Excluded

| Feature | Rationale |
|---------|-----------|
| dbt-fabric | Evaluated but ruled out — authentication between dbt Cloud and personal Microsoft accounts without an Azure subscription is not supported. In a production deployment with an organisational Azure AD tenant, dbt-fabric would be the correct Gold transformation layer. |
| Unity Catalog | Azure Databricks-only feature. Not available in Fabric Spark. Would be the appropriate governance layer in a production Azure Databricks deployment. |
| Airflow orchestration | Out of scope — covered in Project 4 (Companies House ETL Pipeline) which focuses specifically on Airflow DAG patterns. |
| CI/CD pipeline | GitHub Actions workflows were removed when dbt was removed from the stack. In a production Fabric deployment, Microsoft Fabric Git integration provides native CI/CD for notebooks and Lakehouse objects. |
| Real-time ingestion | NHS A&E data is published monthly — a streaming architecture would be over-engineered and inappropriate for this use case. |
