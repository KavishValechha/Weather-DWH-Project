# Weather Analytics Data Warehouse — Project Plan

**Document Version:** 1.0  
**Date:** June 4, 2026  
**Author:** Data Warehouse Team  
**Status:** Active  

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Warehouse Architecture](#3-warehouse-architecture)
4. [Phase 1 — Dataset Exploration](#4-phase-1--dataset-exploration)
5. [Phase 2 — Requirements Definition](#5-phase-2--requirements-definition)
6. [Phase 3 — Warehouse Design](#6-phase-3--warehouse-design)
7. [Phase 4 — SSIS Package 1: Staging Load](#7-phase-4--ssis-package-1-staging-load)
8. [Phase 5 — SSIS Package 2: Warehouse Build](#8-phase-5--ssis-package-2-warehouse-build)
9. [Phase 6 — Validation and Testing](#9-phase-6--validation-and-testing)
10. [Phase 7 — Analytics](#10-phase-7--analytics)
11. [Phase 8 — Reporting](#11-phase-8--reporting)
12. [Deliverables Summary](#12-deliverables-summary)
13. [Dependencies and Sequencing](#13-dependencies-and-sequencing)

---

## 1. Project Overview

| Attribute | Detail |
|---|---|
| **Project Name** | Weather Analytics Data Warehouse |
| **Project Type** | Learning / Portfolio — Data Warehouse Implementation |
| **Goal** | Demonstrate end-to-end Data Warehouse design and implementation using industry-standard dimensional modeling, SSIS-based ETL, data quality validation, and analytical SQL development against a real-world weather dataset. |
| **Dataset** | `weatherHistory.csv` (~96,000 hourly weather observations) |
| **Modeling Approach** | Kimball Dimensional Modeling (Star Schema) |
| **Target Environment** | SQL Server Developer Edition (local), Visual Studio with SSIS, SSMS |

### Expected Deliverables

| # | Deliverable | Phase | Type |
|---|---|---|---|
| 1 | `eda_findings.md` | Phase 1 | Documentation |
| 2 | `requirements.md` | Phase 2 | Documentation |
| 3 | `create_tables.sql` | Phase 3 | SQL DDL |
| 4 | `Package1_Staging.dtsx` | Phase 4 | SSIS Package |
| 5 | `Package2_Warehouse.dtsx` | Phase 5 | SSIS Package |
| 6 | `test_results.md` | Phase 6 | Documentation |
| 7 | `analytical_queries.sql` | Phase 7 | SQL |
| 8 | Weather Analytics Dashboard | Phase 8 | Power BI Report |

---

## 2. Tech Stack

| Component | Tool / Technology | Purpose |
|---|---|---|
| **Database Engine** | Microsoft SQL Server Developer Edition (2019+) | Warehouse storage, query execution |
| **Database Management** | SQL Server Management Studio (SSMS) | DDL execution, query development, table inspection |
| **ETL Tool** | SQL Server Integration Services (SSIS) | Flat file ingestion, data transformation, dimension and fact loading |
| **SSIS Development** | Visual Studio 2019 / 2022 with SQL Server Data Tools (SSDT) | SSIS package authoring and debugging |
| **Query Development** | SSMS Query Editor | Analytical SQL, stored procedures, views |
| **Source Data** | `weatherHistory.csv` (flat file) | Raw weather observations |
| **Reporting (Optional)** | Power BI Desktop | Dashboard creation, visual analytics |
| **Documentation** | Markdown (`.md`) files | Project documentation, EDA findings, test results |
| **Version Control** | Git | Source control for all project artifacts |

### Environment Prerequisites

- SQL Server Developer Edition installed and running locally
- SSMS installed (version 18 or later)
- Visual Studio 2019 or 2022 with the **SQL Server Data Tools (SSDT)** extension installed
- SSIS extension for Visual Studio configured with OLE DB connection support
- Power BI Desktop installed (Phase 8 only)
- Git installed and repository initialized at project root

---

## 3. Warehouse Architecture

### 3.1 Modeling Approach

The warehouse follows the **Kimball Dimensional Modeling** methodology:

- A central **fact table** stores numeric measures at the grain of one observation per hour.
- Surrounding **dimension tables** store descriptive context for slicing and filtering.
- All dimensions use **integer surrogate keys** as primary keys.
- The fact table references dimensions exclusively via surrogate keys.
- Slowly Changing Dimension (SCD) handling is **Type 1** (overwrite) for this static historical dataset.

### 3.2 Star Schema Diagram

```
                        ┌─────────────────────┐
                        │      dim_date        │
                        │─────────────────────│
                        │ date_key (PK)        │
                        │ full_date            │
                        │ year                 │
                        │ quarter              │
                        │ month_number         │
                        │ month_name           │
                        │ week_number          │
                        │ day_of_month         │
                        │ day_name             │
                        │ is_weekend           │
                        │ season               │
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
   ┌──────────┴──────────┐         │         ┌──────────┴──────────┐
   │      dim_time        │         │         │ dim_weather_cond     │
   │─────────────────────│         │         │─────────────────────│
   │ time_key (PK)        │         │         │ weather_cond_key(PK)│
   │ hour_value           │         │         │ summary             │
   │ hour_label           │    ┌───┴────┐     │ precip_type         │
   │ time_of_day_band     │    │ FACT   │     │ daily_summary       │
   └──────────┬──────────┘    │ TABLE  │     └──────────┬──────────┘
              │               └────────┘                │
              └──────── fact_weather_observation ────────┘
                        │ observation_key (PK)
                        │ date_key (FK)
                        │ time_key (FK)
                        │ weather_condition_key (FK)
                        │ observation_datetime
                        │ temperature_c
                        │ apparent_temperature_c
                        │ apparent_temp_gap
                        │ humidity
                        │ wind_speed_kmh
                        │ wind_bearing_degrees
                        │ visibility_km
                        │ cloud_cover
                        │ pressure_millibars
                        │ etl_batch_id
                        │ etl_load_datetime
```

### 3.3 Grain Definition

> **Grain:** One row in `fact_weather_observation` represents **one weather observation recorded at a specific date and time (hour)**. This is the lowest level of detail available in the source dataset.

Each row is uniquely identified by the combination of:
- `date_key` (the calendar date)
- `time_key` (the hour of the observation)

### 3.4 Table Definitions

---

#### `stg_weather_raw` — Staging Table

Holds raw data exactly as extracted from `weatherHistory.csv`. No transformations are applied. Used as the starting point for DQ validation and warehouse loading.

| Column Name | SQL Data Type | Source Column | Notes |
|---|---|---|---|
| `stg_id` | INT IDENTITY | — | Surrogate staging row ID |
| `formatted_date` | NVARCHAR(50) | `Formatted Date` | Raw string; not yet parsed |
| `summary` | NVARCHAR(100) | `Summary` | Raw weather description |
| `precip_type` | NVARCHAR(20) | `Precip Type` | May be NULL in source |
| `temperature_c` | NVARCHAR(20) | `Temperature (C)` | Loaded as string; cast during DQ |
| `apparent_temperature_c` | NVARCHAR(20) | `Apparent Temperature (C)` | Loaded as string |
| `humidity` | NVARCHAR(20) | `Humidity` | Loaded as string |
| `wind_speed_kmh` | NVARCHAR(20) | `Wind Speed (km/h)` | Loaded as string |
| `wind_bearing_degrees` | NVARCHAR(20) | `Wind Bearing (degrees)` | Loaded as string |
| `visibility_km` | NVARCHAR(20) | `Visibility (km)` | Loaded as string |
| `cloud_cover` | NVARCHAR(20) | `Cloud Cover` | Loaded as string |
| `pressure_millibars` | NVARCHAR(20) | `Pressure (millibars)` | Loaded as string |
| `daily_summary` | NVARCHAR(500) | `Daily Summary` | Raw daily narrative |
| `stg_load_datetime` | DATETIME | — | Set by SSIS on load |
| `stg_batch_id` | INT | — | ETL batch identifier |

> All numeric columns are intentionally loaded as NVARCHAR in staging to prevent SSIS type conversion failures on dirty data. Casting is performed during DQ validation in Package 2.

---

#### `dim_date` — Date Dimension

| Column Name | SQL Data Type | Description |
|---|---|---|
| `date_key` | INT | Surrogate primary key (format: YYYYMMDD, e.g., 20160101) |
| `full_date` | DATE | Calendar date |
| `year` | SMALLINT | Calendar year (e.g., 2016) |
| `quarter` | TINYINT | Quarter number 1–4 |
| `quarter_label` | CHAR(2) | Label (Q1, Q2, Q3, Q4) |
| `month_number` | TINYINT | Month number 1–12 |
| `month_name` | NVARCHAR(10) | Month name (e.g., January) |
| `month_short` | CHAR(3) | Abbreviated month (e.g., Jan) |
| `week_number` | TINYINT | ISO week number 1–53 |
| `day_of_month` | TINYINT | Day of month 1–31 |
| `day_of_week` | TINYINT | Day of week 1 (Monday) to 7 (Sunday) |
| `day_name` | NVARCHAR(10) | Day name (e.g., Monday) |
| `day_short` | CHAR(3) | Abbreviated day (e.g., Mon) |
| `is_weekend` | BIT | 1 if Saturday or Sunday |
| `season` | NVARCHAR(10) | Spring / Summer / Autumn / Winter |

> `date_key` is the integer representation of the date (YYYYMMDD). This is the standard Kimball pattern for date dimension keys — it is human-readable, sorts naturally, and eliminates reliance on IDENTITY values.

---

#### `dim_time` — Time Dimension

| Column Name | SQL Data Type | Description |
|---|---|---|
| `time_key` | TINYINT | Surrogate primary key; equals hour value (0–23) |
| `hour_value` | TINYINT | Hour of the day 0–23 |
| `hour_label` | CHAR(8) | 12-hour label (e.g., 02:00 AM, 11:00 PM) |
| `time_of_day_band` | NVARCHAR(15) | Night (0–5), Morning (6–11), Afternoon (12–17), Evening (18–23) |
| `is_business_hours` | BIT | 1 if hour is between 8 and 17 inclusive |

> `dim_time` contains exactly 24 rows. It is a static dimension populated once via a `T-SQL INSERT` script during Phase 3, not via SSIS.

---

#### `dim_weather_condition` — Weather Condition Dimension

| Column Name | SQL Data Type | Description |
|---|---|---|
| `weather_condition_key` | INT IDENTITY | Surrogate primary key |
| `summary` | NVARCHAR(100) | Hourly weather condition label (e.g., "Partly Cloudy") |
| `precip_type` | NVARCHAR(20) | Precipitation type: rain, snow, or None |
| `daily_summary` | NVARCHAR(500) | Daily narrative description |
| `condition_created_datetime` | DATETIME | Row insert timestamp |

> The natural business key for this dimension is the combination of `summary` and `precip_type`. Deduplication is enforced during the SSIS MERGE load to prevent multiple surrogate keys for the same logical condition.

---

#### `fact_weather_observation` — Central Fact Table

| Column Name | SQL Data Type | Description |
|---|---|---|
| `observation_key` | BIGINT IDENTITY | Surrogate primary key |
| `date_key` | INT | FK → `dim_date.date_key` |
| `time_key` | TINYINT | FK → `dim_time.time_key` |
| `weather_condition_key` | INT | FK → `dim_weather_condition.weather_condition_key` |
| `observation_datetime` | DATETIME | Full parsed observation timestamp |
| `temperature_c` | DECIMAL(6,2) | Actual temperature in degrees Celsius |
| `apparent_temperature_c` | DECIMAL(6,2) | Feels-like temperature in degrees Celsius |
| `apparent_temp_gap` | DECIMAL(6,2) | Computed: `apparent_temperature_c - temperature_c` |
| `humidity` | DECIMAL(5,4) | Humidity ratio (0.0000 to 1.0000) |
| `wind_speed_kmh` | DECIMAL(7,2) | Wind speed in km/h |
| `wind_bearing_degrees` | SMALLINT | Wind direction in degrees (0–360) |
| `visibility_km` | DECIMAL(7,2) | Visibility in km |
| `cloud_cover` | DECIMAL(5,4) | Cloud cover ratio (0.0000 to 1.0000) |
| `pressure_millibars` | DECIMAL(8,2) | Atmospheric pressure in millibars |
| `etl_batch_id` | INT | ETL batch identifier for traceability |
| `etl_load_datetime` | DATETIME | Row insert timestamp |

---

#### `dq_rejected_rows` — Data Quality Rejection Log

| Column Name | SQL Data Type | Description |
|---|---|---|
| `rejection_key` | INT IDENTITY | Surrogate primary key |
| `stg_id` | INT | Reference to `stg_weather_raw.stg_id` |
| `formatted_date` | NVARCHAR(50) | Raw date value from staging |
| `rejection_reason_code` | NVARCHAR(20) | Coded reason (e.g., DQ_NULL_DATE, DQ_TEMP_RANGE) |
| `rejection_reason_desc` | NVARCHAR(200) | Human-readable rejection description |
| `raw_row_data` | NVARCHAR(2000) | Full concatenated raw row for inspection |
| `etl_batch_id` | INT | ETL batch identifier |
| `rejected_datetime` | DATETIME | Timestamp of rejection |

---

#### `etl_validation` — ETL Audit and Control Table

| Column Name | SQL Data Type | Description |
|---|---|---|
| `validation_key` | INT IDENTITY | Surrogate primary key |
| `batch_id` | INT | ETL execution batch identifier |
| `package_name` | NVARCHAR(100) | SSIS package name |
| `execution_start` | DATETIME | Package execution start time |
| `execution_end` | DATETIME | Package execution end time |
| `source_row_count` | INT | Total rows in source file |
| `staging_row_count` | INT | Rows loaded to `stg_weather_raw` |
| `dq_pass_count` | INT | Rows passing all DQ rules |
| `dq_reject_count` | INT | Rows failing DQ and written to `dq_rejected_rows` |
| `fact_row_count` | INT | Rows loaded to `fact_weather_observation` |
| `execution_status` | NVARCHAR(10) | Success / Failure / Partial |
| `error_message` | NVARCHAR(1000) | Error details if status is Failure |

---

## 4. Phase 1 — Dataset Exploration

**Objective:** Thoroughly understand the source data before any design or development work begins. Document all findings to inform warehouse design decisions.

**Estimated Effort:** 0.5 days

**Deliverable:** `eda_findings.md`

**Dependency:** Access to `weatherHistory.csv`

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T1-01 | Open and inspect CSV | Open `weatherHistory.csv` in Excel or a text editor. Verify encoding, delimiter, and header row. |
| T1-02 | Count total rows | Count total data rows excluding header. Expected: ~96,000. |
| T1-03 | Profile `Formatted Date` | Identify the date format string, timezone notation, and date range (min and max values). |
| T1-04 | Profile `Summary` | Identify all distinct values. Count occurrences per value. |
| T1-05 | Profile `Precip Type` | Identify all distinct values. Count NULL occurrences and non-null breakdown (rain vs. snow). |
| T1-06 | Profile numeric columns | For each measure column, compute: min, max, mean, and count of NULL values. |
| T1-07 | Detect duplicates | Check for duplicate `Formatted Date` values (same timestamp appearing more than once). |
| T1-08 | Detect nulls | Count NULL or empty values for every column. |
| T1-09 | Assess data quality issues | Document any anomalies: unexpected values, apparent outliers, encoding issues, inconsistent labels. |
| T1-10 | Document findings | Compile all observations into `eda_findings.md`. |

### Acceptance Criteria

- [ ] Total row count documented and matches file
- [ ] Date range (min date, max date) documented
- [ ] Null counts per column documented
- [ ] Distinct value counts for `Summary` and `Precip Type` documented
- [ ] Min, max, and mean documented for all 8 numeric measure columns
- [ ] Duplicate date check result documented
- [ ] `eda_findings.md` created in project root

---

## 5. Phase 2 — Requirements Definition

**Objective:** Formally define and document the business requirements, analytical questions, and scope boundaries for the warehouse.

**Estimated Effort:** 0.5 days

**Deliverable:** `requirements.md`

**Dependency:** Phase 1 complete (`eda_findings.md` available)

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T2-01 | Define business problem | Articulate why raw CSV data is insufficient for analysis. |
| T2-02 | Define audience | Identify the five user personas and their analytical needs. |
| T2-03 | Define scope | Document time period, grain, reporting dimensions, and measurable facts. |
| T2-04 | Write functional requirements | Document FR-01 through FR-07 with acceptance criteria. |
| T2-05 | Write non-functional requirements | Document NFR-01 through NFR-06. |
| T2-06 | Generate business questions | Write 37–40 business questions grouped by analytical category. |
| T2-07 | Map questions to tables | For every business question, identify the required warehouse tables. |
| T2-08 | Create traceability matrix | Map reports to source columns and ETL transformations. |
| T2-09 | Document out-of-scope items | List all items explicitly excluded from the project. |

### Acceptance Criteria

- [ ] `requirements.md` created with all 8 sections complete
- [ ] Minimum 37 business questions documented with Question IDs
- [ ] Every question includes required table mapping
- [ ] Functional and non-functional requirements each have acceptance criteria
- [ ] Out-of-scope section covers at least 8 items

---

## 6. Phase 3 — Warehouse Design

**Objective:** Design the complete star schema. Create all SQL DDL for staging, warehouse, DQ, and audit tables. Populate static reference data.

**Estimated Effort:** 1 day

**Deliverable:** `create_tables.sql`

**Dependency:** Phase 2 complete (`requirements.md` available)

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T3-01 | Create database | Create the `WeatherDWH` database in SQL Server. |
| T3-02 | Create staging schema | Write DDL for `stg_weather_raw`. |
| T3-03 | Design `dim_date` | Define all date dimension columns with data types. Write DDL. |
| T3-04 | Populate `dim_date` | Write a T-SQL loop or tally-table script to populate `dim_date` for all dates in the dataset range (e.g., 2006-01-01 to 2016-12-31). Include season derivation logic. |
| T3-05 | Design `dim_time` | Define all time dimension columns. Write DDL. |
| T3-06 | Populate `dim_time` | Write a T-SQL INSERT for all 24 hours (0–23) with labels and time-of-day bands. |
| T3-07 | Design `dim_weather_condition` | Define dimension columns and natural key constraint. Write DDL. |
| T3-08 | Design `fact_weather_observation` | Define all columns, foreign key constraints, and computed column `apparent_temp_gap`. Write DDL. |
| T3-09 | Create DQ table | Write DDL for `dq_rejected_rows`. |
| T3-10 | Create audit table | Write DDL for `etl_validation`. |
| T3-11 | Create indexes | Write index DDL for all FK columns on `fact_weather_observation` and business key columns on dimensions. |
| T3-12 | Test DDL in SSMS | Execute `create_tables.sql` in SSMS against `WeatherDWH`. Verify all tables and indexes are created successfully. |

### Acceptance Criteria

- [ ] `create_tables.sql` executes without errors in SSMS
- [ ] All 7 tables created: `stg_weather_raw`, `dim_date`, `dim_time`, `dim_weather_condition`, `fact_weather_observation`, `dq_rejected_rows`, `etl_validation`
- [ ] `dim_date` is populated for full date range
- [ ] `dim_time` contains exactly 24 rows
- [ ] All FK constraints defined on `fact_weather_observation`
- [ ] Indexes created on all FK columns

### Season Derivation Logic

```sql
-- Applied during dim_date population
CASE
    WHEN month_number IN (3, 4, 5)  THEN 'Spring'
    WHEN month_number IN (6, 7, 8)  THEN 'Summer'
    WHEN month_number IN (9, 10, 11) THEN 'Autumn'
    ELSE 'Winter'
END AS season
```

---

## 7. Phase 4 — SSIS Package 1: Staging Load

**Objective:** Build and execute the SSIS package that reads `weatherHistory.csv` and loads all rows into `stg_weather_raw`.

**Estimated Effort:** 0.5 days

**Deliverable:** `Package1_Staging.dtsx`

**Dependency:** Phase 3 complete (`create_tables.sql` executed; `stg_weather_raw` exists)

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T4-01 | Create SSIS project | Create a new Integration Services project in Visual Studio. Name: `WeatherDWH_SSIS`. |
| T4-02 | Configure Flat File Connection Manager | Add a Flat File Connection Manager pointed to `weatherHistory.csv`. Set delimiter (comma), header row skip, and column names from the header row. Set all column data types to `Unicode String [DT_WSTR]` with length 500. |
| T4-03 | Configure OLE DB Connection Manager | Add an OLE DB Connection Manager targeting the local SQL Server instance and the `WeatherDWH` database. |
| T4-04 | Add Execute SQL Task — Truncate Staging | Add an Execute SQL Task at the start of the control flow. SQL: `TRUNCATE TABLE stg_weather_raw`. This ensures re-runability. |
| T4-05 | Add Data Flow Task | Add a Data Flow Task connected after the truncate task. |
| T4-06 | Configure Flat File Source | Inside the Data Flow, add a Flat File Source using the Flat File Connection Manager. Map all 12 source columns. |
| T4-07 | Add Derived Column transformation | Add a Derived Column transformation. Add two columns: `stg_load_datetime` (GETDATE()) and `stg_batch_id` (package variable `@[User::BatchId]`, an INT incremented per run). |
| T4-08 | Configure OLE DB Destination | Add an OLE DB Destination targeting `stg_weather_raw`. Map all source columns to staging columns. |
| T4-09 | Add Execute SQL Task — Insert audit row | After the Data Flow Task, add an Execute SQL Task that inserts an initial row into `etl_validation` recording `staging_row_count` using `@@ROWCOUNT` or a SELECT COUNT from `stg_weather_raw`. |
| T4-10 | Configure package variables | Add SSIS package variable `@[User::BatchId]` (INT) and `@[User::SourceFilePath]` (String) as parameterized inputs. |
| T4-11 | Test execution | Execute the package in Visual Studio debug mode. Verify row count in `stg_weather_raw` matches source CSV row count. |

### Acceptance Criteria

- [ ] Package executes without errors in Visual Studio debug mode
- [ ] `stg_weather_raw` row count equals source CSV row count (excluding header)
- [ ] `stg_load_datetime` is populated on all rows
- [ ] `stg_batch_id` is populated on all rows
- [ ] Re-running the package produces the same row count (no duplicates)
- [ ] Initial row inserted into `etl_validation` with `staging_row_count`

### Row Count Validation Query

```sql
-- Run in SSMS after Package 1 execution
SELECT COUNT(*) AS staging_row_count FROM stg_weather_raw;
```

---

## 8. Phase 5 — SSIS Package 2: Warehouse Build

**Objective:** Build and execute the SSIS package that applies DQ validation, loads dimensions, and loads the fact table from `stg_weather_raw`.

**Estimated Effort:** 1.5 days

**Deliverable:** `Package2_Warehouse.dtsx`

**Dependency:** Phase 4 complete (`Package1_Staging.dtsx` successfully executed; `stg_weather_raw` populated)

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T5-01 | Create Package 2 | Add a new SSIS package to the existing project. Name: `Package2_Warehouse`. |
| T5-02 | Add DQ Validation Data Flow | Add a Data Flow Task for DQ validation. Source: `stg_weather_raw`. Use a **Conditional Split** transformation to route valid rows to the warehouse pipeline and invalid rows to `dq_rejected_rows`. |
| T5-03 | Configure DQ rules in Conditional Split | Implement all 7 DQ rules from FR-02. Each condition routes failing rows to the rejection output. |
| T5-04 | Load `dim_weather_condition` | Add an OLE DB Command or Execute SQL Task using `MERGE` to upsert distinct `(summary, precip_type)` combinations into `dim_weather_condition`. |
| T5-05 | Load `fact_weather_observation` | Add a Data Flow Task. Source: valid rows from the DQ Conditional Split. Use OLE DB Lookups to resolve `date_key` (from parsed date), `time_key` (from parsed hour), and `weather_condition_key` (from summary + precip_type). Destination: `fact_weather_observation`. |
| T5-06 | Compute `apparent_temp_gap` | Add a Derived Column transformation before the fact destination. Compute: `apparent_temperature_c - temperature_c`. |
| T5-07 | Load `dq_rejected_rows` | Route all Conditional Split rejection outputs to a separate OLE DB Destination targeting `dq_rejected_rows`. Populate `rejection_reason_code` and `rejection_reason_desc` using Derived Column. |
| T5-08 | Update `etl_validation` | After all data flows complete, add an Execute SQL Task to update the `etl_validation` row for this batch with: `dq_pass_count`, `dq_reject_count`, `fact_row_count`, `execution_end`, `execution_status`. |
| T5-09 | Add error handling | Wrap all tasks in a Sequence Container with an `OnError` event handler that updates `etl_validation.execution_status` to `'Failure'` and sets `error_message`. |
| T5-10 | Test execution | Execute Package 2 in Visual Studio debug mode. Verify counts in all tables. |

### DQ Conditional Split Conditions

| Output Name | Condition | Rejection Code |
|---|---|---|
| `DQ_NULL_DATE` | `ISNULL(formatted_date)` | `DQ_NULL_DATE` |
| `DQ_TEMP_RANGE` | `(CAST(temperature_c AS DT_R8) < -60) \|\| (CAST(temperature_c AS DT_R8) > 60)` | `DQ_TEMP_RANGE` |
| `DQ_HUMIDITY_RANGE` | `(CAST(humidity AS DT_R8) < 0) \|\| (CAST(humidity AS DT_R8) > 1)` | `DQ_HUMIDITY_RANGE` |
| `DQ_WIND_NEGATIVE` | `CAST(wind_speed_kmh AS DT_R8) < 0` | `DQ_WIND_NEGATIVE` |
| `DQ_PRESSURE_RANGE` | `(CAST(pressure_millibars AS DT_R8) < 870) \|\| (CAST(pressure_millibars AS DT_R8) > 1085)` | `DQ_PRESSURE_RANGE` |
| `DQ_VISIBILITY_NEGATIVE` | `CAST(visibility_km AS DT_R8) < 0` | `DQ_VISIBILITY_NEGATIVE` |
| `Valid` | All above conditions false | — |

### Acceptance Criteria

- [ ] Package executes without errors in Visual Studio debug mode
- [ ] `fact_weather_observation` + `dq_rejected_rows` count = `stg_weather_raw` count
- [ ] `dim_weather_condition` contains all distinct condition combinations
- [ ] All FK values in `fact_weather_observation` resolve to valid dimension keys
- [ ] `apparent_temp_gap` is populated on all fact rows
- [ ] `etl_validation` row updated with final counts and `'Success'` status
- [ ] Re-running both packages produces identical row counts (idempotent)

---

## 9. Phase 6 — Validation and Testing

**Objective:** Formally validate the end-to-end ETL pipeline output. Verify data integrity, row counts, referential integrity, and aggregate accuracy.

**Estimated Effort:** 0.5 days

**Deliverable:** `test_results.md`

**Dependency:** Phase 5 complete (both SSIS packages executed successfully)

### Test Cases

| Test ID | Test Name | Test Type | Validation SQL | Expected Result |
|---|---|---|---|---|
| TC-01 | Staging Row Count | Row Count | `SELECT COUNT(*) FROM stg_weather_raw` | Equals source CSV row count |
| TC-02 | Fact Row Count | Row Count | `SELECT COUNT(*) FROM fact_weather_observation` | <= staging row count; >= staging count minus DQ rejects |
| TC-03 | DQ Reconciliation | Row Count | `SELECT (SELECT COUNT(*) FROM fact_weather_observation) + (SELECT COUNT(*) FROM dq_rejected_rows)` | Equals `stg_weather_raw` count |
| TC-04 | Duplicate Observations | Duplicate Check | `SELECT date_key, time_key, COUNT(*) FROM fact_weather_observation GROUP BY date_key, time_key HAVING COUNT(*) > 1` | Zero rows returned |
| TC-05 | Null Foreign Keys in Fact | Null Check | `SELECT COUNT(*) FROM fact_weather_observation WHERE date_key IS NULL OR time_key IS NULL OR weather_condition_key IS NULL` | Zero |
| TC-06 | Orphan date_key | Referential Integrity | `SELECT COUNT(*) FROM fact_weather_observation f LEFT JOIN dim_date d ON f.date_key = d.date_key WHERE d.date_key IS NULL` | Zero |
| TC-07 | Orphan time_key | Referential Integrity | `SELECT COUNT(*) FROM fact_weather_observation f LEFT JOIN dim_time t ON f.time_key = t.time_key WHERE t.time_key IS NULL` | Zero |
| TC-08 | Orphan weather_condition_key | Referential Integrity | `SELECT COUNT(*) FROM fact_weather_observation f LEFT JOIN dim_weather_condition c ON f.weather_condition_key = c.weather_condition_key WHERE c.weather_condition_key IS NULL` | Zero |
| TC-09 | Temperature Range Check | Data Quality | `SELECT COUNT(*) FROM fact_weather_observation WHERE temperature_c < -60 OR temperature_c > 60` | Zero |
| TC-10 | Humidity Range Check | Data Quality | `SELECT COUNT(*) FROM fact_weather_observation WHERE humidity < 0 OR humidity > 1` | Zero |
| TC-11 | Wind Speed Non-Negative | Data Quality | `SELECT COUNT(*) FROM fact_weather_observation WHERE wind_speed_kmh < 0` | Zero |
| TC-12 | Pressure Range Check | Data Quality | `SELECT COUNT(*) FROM fact_weather_observation WHERE pressure_millibars < 870 OR pressure_millibars > 1085` | Zero |
| TC-13 | dim_date Coverage | Completeness | `SELECT MIN(full_date), MAX(full_date) FROM dim_date` | Covers full dataset date range |
| TC-14 | dim_time Row Count | Completeness | `SELECT COUNT(*) FROM dim_time` | Exactly 24 rows |
| TC-15 | Idempotency Test | Re-Runability | Execute both packages twice. Compare row counts before and after second run. | Identical row counts |
| TC-16 | ETL Audit Record | Auditability | `SELECT * FROM etl_validation ORDER BY validation_key DESC` | Rows present with `execution_status = 'Success'`; counts reconcile |
| TC-17 | Aggregate Spot Check | Aggregate Accuracy | `SELECT AVG(temperature_c) FROM fact_weather_observation` vs. manual CSV calculation | Values match within rounding tolerance |

### Acceptance Criteria

- [ ] All 17 test cases executed and results documented in `test_results.md`
- [ ] Zero failing test cases (all expected results met)
- [ ] `test_results.md` includes actual vs. expected values for every test

---

## 10. Phase 7 — Analytics

**Objective:** Write SQL queries that answer all 37 business questions defined in `requirements.md`. Create reporting views.

**Estimated Effort:** 1 day

**Deliverable:** `analytical_queries.sql`

**Dependency:** Phase 6 complete (all validation tests passed)

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T7-01 | Set up query file structure | Create `analytical_queries.sql` with section headers matching the 6 business question categories. Each query block prefixed with the Question ID comment (e.g., `-- BQ-01`). |
| T7-02 | Write Descriptive Analytics queries | BQ-01 through BQ-10. |
| T7-03 | Write Trend Analysis queries | BQ-11 through BQ-18. Use `GROUP BY` on `dim_date` columns. |
| T7-04 | Write Comparative Analysis queries | BQ-19 through BQ-25. Use `CASE WHEN` and conditional aggregation. |
| T7-05 | Write Weather Condition Analysis queries | BQ-26 through BQ-31. Use `GROUP BY` on `dim_weather_condition` columns. |
| T7-06 | Write Correlation Analysis queries | BQ-32 through BQ-36. Use binning (`CASE WHEN` bands) for scatter-equivalent grouping. |
| T7-07 | Write Anomaly Detection queries | BQ-37 through BQ-40. Use `PERCENTILE_CONT` window functions and threshold `WHERE` clauses. |
| T7-08 | Create `vw_monthly_weather_summary` view | Create a view joining fact + `dim_date` returning monthly aggregates for temperature, humidity, wind speed, visibility, and precipitation count. |
| T7-09 | Create `vw_annual_trends` view | Create a view returning year-over-year averages for all key measures. |
| T7-10 | Create `vw_condition_stats` view | Create a view returning average measures per weather condition summary. |
| T7-11 | Validate query results | Cross-validate at least 5 query results against manual spot-checks from `stg_weather_raw` or the source CSV. |

### Query File Structure

```sql
-- ============================================================
-- Weather Analytics Data Warehouse
-- Analytical Queries
-- analytical_queries.sql
-- ============================================================

-- ============================================================
-- SECTION 1: DESCRIPTIVE ANALYTICS
-- ============================================================

-- BQ-01: Highest recorded temperature and date
-- BQ-02: Lowest recorded temperature and date
-- ...

-- ============================================================
-- SECTION 2: TREND ANALYSIS
-- ============================================================
-- BQ-11 through BQ-18

-- ... and so on for all 6 sections
```

### Acceptance Criteria

- [ ] `analytical_queries.sql` contains all 37 queries, each prefixed with `-- BQ-XX`
- [ ] All queries execute without errors in SSMS
- [ ] At minimum, queries for BQ-01, BQ-11, BQ-19, BQ-26, BQ-32, BQ-37 are spot-checked and validated
- [ ] Three reporting views created: `vw_monthly_weather_summary`, `vw_annual_trends`, `vw_condition_stats`
- [ ] All views execute without errors and return expected columns

---

## 11. Phase 8 — Reporting

**Objective:** Connect Power BI Desktop to the warehouse and build a dashboard that visually answers the key business questions.

**Estimated Effort:** 0.5 days

**Deliverable:** Weather Analytics Dashboard (`.pbix` file)

**Dependency:** Phase 7 complete (analytical queries and views available)

> This phase is **optional** for the core learning project but is strongly recommended to demonstrate reporting readiness and end-to-end warehouse value.

### Tasks

| Task ID | Task | Description |
|---|---|---|
| T8-01 | Connect Power BI to SQL Server | Open Power BI Desktop. Use Get Data → SQL Server. Connect to `WeatherDWH`. Import `fact_weather_observation`, `dim_date`, `dim_time`, `dim_weather_condition`, and all three reporting views. |
| T8-02 | Verify relationships | In the Power BI Model view, verify relationships: `fact_weather_observation` → `dim_date` (via `date_key`), → `dim_time` (via `time_key`), → `dim_weather_condition` (via `weather_condition_key`). Set cardinality to Many-to-One. |
| T8-03 | Build Dashboard Page 1 — Overview | KPI cards: Total Observations, Date Range, Avg Temperature, Avg Humidity. Bar chart: Observations by Year. Line chart: Average Temperature by Month. |
| T8-04 | Build Dashboard Page 2 — Temperature Trends | Line chart: Monthly average temperature by year. Bar chart: Average temperature by season. Scatter plot: Temperature vs. Humidity. |
| T8-05 | Build Dashboard Page 3 — Weather Conditions | Donut chart: Precipitation type distribution. Bar chart: Top 10 most common weather summaries. Clustered bar: Average wind speed by weather condition. |
| T8-06 | Build Dashboard Page 4 — Anomalies | Table: Extreme heat events (BQ-37). Table: Extreme cold events (BQ-38). Table: Low visibility events (BQ-40). |
| T8-07 | Add slicers | Add slicers for Year, Season, Precipitation Type across all dashboard pages. |
| T8-08 | Validate dashboard values | Cross-validate at least 3 dashboard visuals against SSMS query results. |
| T8-09 | Save `.pbix` file | Save as `WeatherAnalyticsDashboard.pbix` in the project root. |

### Acceptance Criteria

- [ ] Power BI connects to `WeatherDWH` without errors
- [ ] All three dimension relationships are correctly defined in Power BI Model view
- [ ] Dashboard contains a minimum of 4 pages
- [ ] At least 3 visual values validated against SSMS analytical queries
- [ ] `.pbix` file saved to project root

---

## 12. Deliverables Summary

| # | Deliverable File | Phase | Location | Status |
|---|---|---|---|---|
| 1 | `eda_findings.md` | Phase 1 | Project Root | Pending |
| 2 | `requirements.md` | Phase 2 | Project Root | Pending |
| 3 | `create_tables.sql` | Phase 3 | Project Root | Pending |
| 4 | `Package1_Staging.dtsx` | Phase 4 | SSIS Project Folder | Pending |
| 5 | `Package2_Warehouse.dtsx` | Phase 5 | SSIS Project Folder | Pending |
| 6 | `test_results.md` | Phase 6 | Project Root | Pending |
| 7 | `analytical_queries.sql` | Phase 7 | Project Root | Pending |
| 8 | `WeatherAnalyticsDashboard.pbix` | Phase 8 | Project Root | Optional |

---

## 13. Dependencies and Sequencing

The phases are strictly sequential. No phase should begin until all acceptance criteria for the preceding phase are met.

```
Phase 1 (EDA)
    │
    └──► Phase 2 (Requirements)
              │
              └──► Phase 3 (Warehouse Design + DDL)
                        │
                        └──► Phase 4 (SSIS Package 1 — Staging)
                                  │
                                  └──► Phase 5 (SSIS Package 2 — Warehouse Build)
                                            │
                                            └──► Phase 6 (Validation and Testing)
                                                      │
                                                      └──► Phase 7 (Analytics — SQL)
                                                                │
                                                                └──► Phase 8 (Reporting — Power BI)
```

### Phase Dependency Summary

| Phase | Depends On | Gate Condition |
|---|---|---|
| Phase 1 | None | `weatherHistory.csv` available |
| Phase 2 | Phase 1 | `eda_findings.md` complete |
| Phase 3 | Phase 2 | `requirements.md` complete |
| Phase 4 | Phase 3 | `create_tables.sql` executed; `stg_weather_raw` exists |
| Phase 5 | Phase 4 | `stg_weather_raw` populated; all dimensions populated |
| Phase 6 | Phase 5 | Both SSIS packages executed successfully |
| Phase 7 | Phase 6 | All 17 test cases passed |
| Phase 8 | Phase 7 | `analytical_queries.sql` complete; views created |

---

*End of Document — Weather Analytics Data Warehouse Project Plan v1.0*
