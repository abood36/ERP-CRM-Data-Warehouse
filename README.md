# ERP-CRM-Data-Warehouse

A complete Data Engineering pipeline built using SQL Server and Python, designed to collect, process, and analyze large-scale datasets efficiently. This repository contains the SQL (T-SQL) objects and supporting Python helpers to populate and maintain a data warehouse fed by ERP and CRM systems.

- Primary language: T-SQL (SQL Server)
- Supporting automation / orchestration: Python (ETL helpers, optional runners)
- Goal: provide a reproducible, production-ready pipeline for ingesting, transforming, and serving ERP/CRM data for analytics and reporting.

---

## Table of contents

- Overview
- Key features
- Architecture (high level)
- Prerequisites
- Repository layout (suggested / typical)
- Quick start
- Configuration
- Running pipelines and scheduling
- Monitoring & logging
- Testing
- Common SQL snippets
- Contributing
- License
- Contact

---

## Overview

This project centralizes extract-transform-load (ETL) logic for ERP and CRM sources into a managed SQL Server-based Data Warehouse. Core processing and transformations are implemented in T-SQL (stored procedures, views, staging tables, and dimensional modeling). Python is used for lightweight orchestration, connectivity helpers, and optional ad-hoc loaders.

Use cases:
- Centralize ERP and CRM operational data for analytics
- Maintain slowly changing dimensions (SCD)
- Build star schemas / facts for BI consumption
- Provide auditable, repeatable ETL steps

---

## Key features

- T-SQL based ETL logic and schema objects optimized for SQL Server
- Staging area design and deduplication patterns
- SCD Type 1 / Type 2 implementations and utilities
- Python helpers for data ingestion, validation, and orchestration (pyodbc / sqlalchemy)
- Configurable connection and environment settings
- Examples for scheduling with SQL Server Agent or a scheduler of your choice
- Logging and simple alert patterns based on execution results

---

## Architecture (high level)

1. Source connectors (ERP/CRM exports, APIs, flat files) feed raw data into an ingestion layer.
2. Staging schemas hold raw, minimally transformed data.
3. Transformations and business rules are applied in T-SQL (stored procedures).
4. Data is loaded into dimensional models (dimensions + fact tables).
5. Downstream consumers (BI tools, analytics, reporting) query the dimensional model.

ASCII diagram:

Source systems (ERP/CRM, files, APIs)
        ↓
 Ingest (Python / bulk load / SSIS / BCP)
        ↓
 Staging schema (stg_*)
        ↓
 Transformations (T-SQL stored procedures)
        ↓
 Dimensional model (dim_*, fact_*)
        ↓
 BI / Reporting / ML

---

## Prerequisites

- Microsoft SQL Server 2016+ (or compatible)
- SQL Server Management Studio (SSMS) recommended
- SQL Server Agent (for scheduling jobs) or an external scheduler (Airflow, cron, etc.)
- Python 3.8+ (optional, for helpers)
  - Recommended Python packages: pyodbc, pandas, sqlalchemy, python-dotenv (or configparser)
- Access credentials with permissions to create schemas, tables, procedures and to run jobs

Optional:
- Docker (to run a SQL Server instance locally for development)
- BCP / sqlcmd tools for bulk loads

---

## Repository layout (typical)

Note: adjust paths below to match this repository's actual structure. Example structure you can adopt:

- sql/
  - ddl/
    - create_database.sql
    - schemas.sql
    - tables/
  - dml/
    - staging_loads.sql
    - transforms/
  - sprocs/
    - etl_master_proc.sql
    - scd_helpers.sql
  - views/
  - seeds/
- python/
  - connectors/
    - load_from_api.py
    - load_from_file.py
  - runners/
    - etl_runner.py
  - requirements.txt
- docs/
  - data_dictionary.md
  - architecture.md
- tests/
  - sql/
  - integration/
- README.md

---

## Quick start

1. Clone the repository

   ```
   git clone https://github.com/abood36/ERP-CRM-Data-Warehouse.git
   cd ERP-CRM-Data-Warehouse
   ```

2. Create or prepare the target SQL Server instance and credentials.

3. Apply DDL (create database, schemas, tables). Example using sqlcmd:

   ```
   sqlcmd -S <SERVER_NAME> -U <USER> -P <PASSWORD> -i sql/ddl/create_database.sql
   sqlcmd -S <SERVER_NAME> -U <USER> -P <PASSWORD> -i sql/ddl/schemas.sql
   ```

4. Deploy stored procedures, views and transforms:

   ```
   sqlcmd -S <SERVER_NAME> -U <USER> -P <PASSWORD> -i sql/sprocs/etl_master_proc.sql
   ```

5. (Optional) Set up Python environment and run a loader:

   ```
   python -m venv .venv
   source .venv/bin/activate    # On Windows: .venv\Scripts\activate
   pip install -r python/requirements.txt
   python python/runners/etl_runner.py --config config/config.ini
   ```

---

## Configuration

Use an environment-safe configuration file or environment variables to store connection strings and secrets. Example config/config.ini:

```ini
[database]
server = your_sql_server
database = dwh_db
username = dwh_user
password = your_password
driver = ODBC Driver 17 for SQL Server

[logging]
log_table = etl.audit_log
log_level = INFO
```

Secure secrets using:
- Windows Credential Manager / Azure Key Vault / HashiCorp Vault
- CI/CD secrets for deployment
- Do NOT commit credentials to Git

---

## Running pipelines and scheduling

- SQL Server Agent job: create jobs that call the master ETL stored procedure or call staging loader procedures.
- Airflow / Prefect: wrap Python runners or use mssql hooks/operators to trigger SQL stored procedures.
- Cron / scheduler: run Python scripts or sqlcmd invocations on schedule.

Example SQL Server Agent command to execute a stored proc:

```sql
EXEC etl.etl_master_run @run_date = CONVERT(date, GETDATE());
```

---

## Monitoring & Logging

- Implement an audit table (e.g., etl.audit_log) that stores start/end times, status, rows processed, error messages.
- Each stored procedure should write status rows before and after execution.
- For Python jobs, log to stdout and/or to the database audit table.
- Set up alerts on repeated failures (email/slack/webhook).

---

## Testing

- Unit test T-SQL via tSQLt (if you want T-SQL unit tests).
- Integration tests: run small sample loads into a dedicated test database and assert expected row counts and values.
- Python: use pytest for any helper scripts.

---

## Common SQL snippets

SCD Type 2 upsert pattern (simplified):

```sql
-- assume dim_customer and stg_customer exist
MERGE dim.d_customer AS target
USING stg.stg_customer AS source
ON target.business_key = source.business_key
WHEN MATCHED AND (
    ISNULL(target.col1,'') <> ISNULL(source.col1,'')
    OR ISNULL(target.col2,'') <> ISNULL(source.col2,'')
)
THEN
    -- expire current version
    UPDATE SET is_current = 0, end_date = GETDATE()
WHEN NOT MATCHED BY TARGET
THEN
    INSERT (business_key, col1, col2, start_date, end_date, is_current)
    VALUES (source.business_key, source.col1, source.col2, GETDATE(), '9999-12-31', 1);
```

Bulk load via BCP (example):

```
bcp dbo.stg_table in data/file.csv -S server -d db -U user -P pass -c -t','
```

---

## Troubleshooting

- Login/Permissions: Ensure service account has CREATE/INSERT/UPDATE/EXECUTE privileges.
- Performance: Add indexing to staging and dimension tables, use minimal logging (bulk-logged/simple recovery during bulk loads), partition large fact tables.
- Transaction timeouts: break large operations into batches.
- Connectivity issues from Python: ensure ODBC driver and firewall rules are correct.

---

## Contributing

Contributions are welcome. Suggested workflow:

1. Open an issue to discuss large changes or new features.
2. Fork the repo and create a feature branch.
3. Add tests (T-SQL/unit/integration) when applicable.
4. Submit a pull request with a clear description and migration notes.

Please follow the repository's coding and naming conventions for SQL objects and scripts.

---

## License

Specify your license here (e.g., MIT). If none, add one before publishing.

---

## Contact

Maintainer: abood36  
Repository: https://github.com/abood36/ERP-CRM-Data-Warehouse

If you need help with deployment or architecture advice, open an issue or raise a discussion in the repo.
