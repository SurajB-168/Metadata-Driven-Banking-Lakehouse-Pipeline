# NeoBank Lakehouse: Metadata-Driven Banking Data Pipeline on Databricks

An end-to-end data engineering project that ingests banking data from **SQL Server** (via JDBC) and **CSV files** (via Auto Loader) into a **Medallion (Bronze → Silver → Gold) Lakehouse** on Databricks. The pipeline is **metadata-driven**: each table's load strategy, primary key and watermark column are stored in control tables, not hardcoded in the notebooks. It supports incremental loads, per-run audit logging, email notifications and an executive dashboard.
---

## Table of Contents
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Key Features](#key-features)
- [Data Sources](#data-sources)
- [Metadata Framework](#metadata-framework)
- [Pipeline Walkthrough](#pipeline-walkthrough)
- [Gold Layer Tables](#gold-layer-tables)
- [Dashboard](#dashboard)
- [Repository Structure](#repository-structure)
- [How to Run](#how-to-run)
- [What I Added](#what-i-added)
- [Known Limitations and Roadmap](#known-limitations-and-roadmap)
- [Author](#author)

---

## Architecture

```
┌──────────────────┐      JDBC (incremental, watermark-based)
│   SQL Server     │ ──────────────────────────────┐
│ customers        │                               │
│ accounts         │                               ▼
│ transactions     │                     ┌──────────────────┐
│ branches         │                     │  BRONZE (Delta)  │  raw, append-only,
└──────────────────┘                     │  banking.bronze  │  + insert_timestamp
                                         └────────┬─────────┘
┌──────────────────┐      Auto Loader              │
│ Blob / Volume    │ ──────────────────────────────┤
│ credit_bureau    │   (cloudFiles, availableNow)  │
│ payment_gateway  │                               ▼
└──────────────────┘                     ┌──────────────────┐
                                         │  SILVER (Delta)  │  FULL / APPEND / MERGE
                                         │  banking.silver  │  driven by metadata
                                         └────────┬─────────┘
                                                  │
                                                  ▼
                                         ┌──────────────────┐
                                         │   GOLD (Delta)   │  business aggregates
                                         │  banking.gold    │
                                         └────────┬─────────┘
                                                  │
                          ┌───────────────────────┼───────────────────────┐
                          ▼                       ▼                       ▼
                 Lakeview Dashboard      Email run summary        Audit table
                                                                (pipeline_runs)

        Control plane: banking.metadata (tables, table_parameters,
                       table_watermarks, pipeline_runs)
```

<!-- TODO: Replace with an architecture diagram image, e.g. ![Architecture](docs/architecture.png) -->

## Tech Stack

| Area | Tools |
|---|---|
| Platform | Databricks (Unity Catalog, Volumes, Jobs / Workflows) |
| Processing | PySpark, Spark SQL, Structured Streaming (Auto Loader) |
| Storage | Delta Lake |
| Sources | SQL Server (JDBC), CSV files in Unity Catalog Volumes |
| Security | Databricks Secret Scopes |
| Orchestration | Databricks Jobs with task values |
| Reporting | Databricks Lakeview (AI/BI) Dashboard |
| Notifications | Python `smtplib` (HTML email) |

## Key Features

- **Metadata-driven ingestion:** adding a table means inserting rows into control tables, with no pipeline code changes.
- **Three load strategies** configured per table: `FULL` (overwrite), `APPEND`, and `MERGE` (Delta upsert on the primary key).
- **Incremental loading** with a high-watermark stored per table in `metadata.table_watermarks`.
- **Two ingestion patterns:** JDBC batch reads from SQL Server and file ingestion with **Auto Loader** (`cloudFiles`, `trigger(availableNow=True)`, checkpointed).
- **Operational audit trail:** every table load per layer per run is logged in `metadata.pipeline_runs` with start and end time, status, record count and error message.
- **Secrets management:** database credentials are stored as a JSON secret in a Databricks Secret Scope. No credentials are in code.
- **Run notification:** an HTML email summarises the status of every table load at the end of a run.
- **Analytics layer:** five gold tables feed an executive dashboard.

## Data Sources

| Source | System | Tables / Files | Load Type | Watermark Column |
|---|---|---|---|---|
| SQL Server | `banking` schema | `customers` | MERGE | `updated_at` |
| SQL Server | `banking` schema | `accounts` | MERGE | `updated_at` |
| SQL Server | `banking` schema | `transactions` | APPEND | `txn_timestamp` |
| SQL Server | `banking` schema | `branches` | FULL | none |
| Blob / Volume (CSV) | Landing volume | `credit_bureau_reports` | MERGE | `bureau_pull_date` |
| Blob / Volume (CSV) | Landing volume | `payment_gateway_logs` | APPEND | `processed_timestamp` |

The dataset is synthetic. It includes a **historical load** and **incremental batches** (SQL insert scripts and `*_incremental.csv` files) so incremental logic can be demonstrated across runs.

## Metadata Framework

All behaviour is controlled by four Delta tables in `banking.metadata`:

| Table | Purpose |
|---|---|
| `tables` | Registry of logical tables: source system, schema, path, layer, active flag, load order |
| `table_parameters` | Key/value config per table: `load_type`, `primary_key`, `watermark_column` |
| `table_watermarks` | Last successfully processed watermark value per table |
| `pipeline_runs` | Audit log: run id, table, layer, timings, status, row count, error message |

## Pipeline Walkthrough

1. **Setup** (`01_Setup_Metadata`): creates the catalog, schemas, volume and control tables, then seeds the metadata and initial watermarks.
2. **Read table list** (`01_Read_Tables_List`): filters `metadata.tables` by `source_system` and `active_flag`, ordered by `load_order`, and publishes the list as a Databricks Jobs **task value**.
3. **Read table parameters** (`02_Read_Table_Parameters`): fetches the load configuration for one table and publishes it as a task value.
4. **Source → Bronze** (`03_Source_to_Bronze`):
   - SQL Server: reads only rows newer than the last watermark over JDBC, using credentials from the secret scope.
   - Blob: streams new CSV files with Auto Loader.
   - Appends to `banking.bronze.<table>` with an `insert_timestamp`.
5. **Bronze → Silver** (`04_Bronze_to_Silver`): applies the configured strategy (overwrite / append / Delta `MERGE`), advances the watermark, and writes the audit record in a `finally` block.
6. **Silver → Gold** (`01_Silver_to_Gold_Driver`): logs the audit entry and runs the matching notebook in `gold_transformations/`.
7. **Notify** (`04_Email_Notification`): builds an HTML summary from `pipeline_runs` for the run and emails it.

## Gold Layer Tables

| Table | Description |
|---|---|
| `customer_360` | One row per customer: branch, account count, total balance, transaction count and amount, latest credit score and risk grade, and a value segment (`HIGH_VALUE` / `MEDIUM_VALUE` / `LOW_VALUE`) |
| `branch_performance` | Branch-level customers, deposits and transactions |
| `transaction_channel_summary` | Transaction volumes by channel / gateway / device |
| `daily_bank_kpi` | Daily transaction totals alongside bank-wide customer, balance and credit metrics |
| `risk_customer_summary` | Customer counts, average credit score and overdue exposure by risk grade |

## Dashboard

A Lakeview dashboard (`05_Dashboard/NeoBank_Dashboard.lvdash.json`) built on the gold tables, with KPI counters (customers, deposits, transactions, high-risk customers), customer segment and credit risk distributions, branch performance, and device/gateway transaction breakdowns.

<!-- TODO: Add screenshots, e.g. ![Dashboard](docs/dashboard.png) -->

## Repository Structure

```
.
├── 00_Source_Files/
│   ├── 01_SQL_Server/        # DDL, historical inserts, incremental inserts
│   └── 02_Blob/              # credit bureau + payment gateway CSVs (initial + incremental)
├── 01_Setup_Metadata/        # control tables, seed metadata, metadata checks
├── 02_Source_to_Silver/      # secret scope setup, metadata readers, bronze + silver loads
├── 03_Silver_to_Gold/
│   ├── 01_Silver_to_Gold_Driver.py
│   └── gold_transformations/ # one notebook per gold table
├── 04_Email_Notification/    # run summary email
├── 05_Dashboard/             # Lakeview dashboard export
└── README.md
```

## How to Run

**Prerequisites:** a Databricks workspace with Unity Catalog, a SQL Server instance reachable from Databricks, and permission to create a catalog and secret scope.

1. **Create the source data:** run the scripts in `00_Source_Files/01_SQL_Server/` on SQL Server (`01_Create_Tables` → `02_Insert_Historical_data`). Upload the initial CSVs to the Volume folders created by the setup notebook (`credit_bureau_reports/`, `payment_gateway_logs/`).
2. **Configure secrets:** edit and run `00_Setup_Secret_Scope.py` with your own connection details. Never commit real credentials.
3. **Set up metadata:** run `01_Setup_Metadata.sql`, then `02_Check_Metadata.sql` to verify.
4. **Create a Databricks Job** with tasks in this order: read tables list → read table parameters → source to bronze → bronze to silver → silver to gold → email notification. Pass `run_id`, `table_metadata` and `table_parameters` between tasks using task values or job parameters.
5. **First run:** the historical load. **Second run:** load `03_Incrementat_data.sql` and the `*_incremental.csv` files, then re-run the job to see the watermark-based incremental behaviour.
6. **Import the dashboard:** import `NeoBank_Dashboard.lvdash.json` into Databricks Lakeview.
7. **Email:** store a Gmail app password in the secret scope and change the sender and recipient addresses in the notification notebook.

## What I Added

> Fill this section with **your real contributions**. Interviewers will ask, and specific, honest answers are what make this project credible. Examples to replace or delete:

- [ ] Added data cleaning in Silver (type casting, null handling, deduplication before MERGE)
- [ ] Added data quality checks and a quarantine table for rejected records
- [ ] Masked PII (PAN, email, phone) in Silver / Gold
- [ ] Fixed audit logging bug (Bronze rows logged with the wrong layer)
- [ ] Implemented SCD Type 2 for credit bureau history
- [ ] Deployed with Databricks Asset Bundles
- [ ] Extended the dataset or added a new source / gold table

## Known Limitations and Roadmap

Being upfront about what would be improved for production use:

- **Silver layer is light:** it mainly adds audit columns. Type standardisation, deduplication and data quality rules are planned.
- **MERGE assumes one row per key per batch:** duplicates within a batch would fail the merge, so a "latest record per key" step is needed.
- **Idempotency:** re-running Bronze before Silver can re-append rows because the watermark advances only after Silver.
- **Credit bureau history:** MERGE overwrites prior records, so history is lost. SCD Type 2 or append would preserve it.
- **Gold tables are fully rebuilt** on every run (`CREATE OR REPLACE`), not incrementally.
- **No dimensional model, automated tests, CI/CD or infrastructure-as-code** yet.
- **Watermarks are stored as strings;** typed comparisons would be safer.
- **Notification** uses SMTP with a personal Gmail account; a workspace-native alert or service account is preferable.

## Author

**Your Name**
[LinkedIn](https://linkedin.com/in/your-profile) · [GitHub](https://github.com/your-username) · your.email@example.com

## Acknowledgements

Based on the banking capstone from the [DataBeli Databricks course](https://github.com/databeli/databricks_course) by Narender Kumar.
