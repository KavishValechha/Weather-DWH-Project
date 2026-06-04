# Weather Analytics Data Warehouse — Project Plan

**Document Version:** 2.0  
**Date:** June 4, 2026  
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
10. [Phase 7 — Analytical SQL](#10-phase-7--analytical-sql)
11. [Deliverables Summary](#11-deliverables-summary)
12. [Phase Sequencing](#12-phase-sequencing)

---

## 1. Project Overview

| Attribute | Detail |
|---|---|
| **Project Name** | Weather Analytics Data Warehouse |
| **Goal** | Demonstrate end-to-end Data Warehouse design using Kimball Dimensional Modeling, SSIS ETL, staging layer design, data quality enforcement, and analytical SQL on a real weather dataset |
| **Dataset** | `weatherHistory.csv` (~96,000 hourly weather observations) |
| **Modeling Approach** | Kimball Star Schema |
| **Environment** | SQL Server Developer Edition (local), Visual Studio with SSIS, SSMS |

**Deliverables:**

| # | File | Type |
|---|---|---|
| 1 | `eda_findings.md` | Documentation |
| 2 | `requirements.md` | Documentation |
| 3 | `create_tables.sql` | SQL DDL |
| 4 | `Package1_Staging.dtsx` | SSIS Package |
| 5 | `Package2_Warehouse.dtsx` | SSIS Package |
| 6 | `test_results.md` | Documentation |
| 7 | `analytical_queries.sql` | SQL |

---

## 2. Tech Stack

| Tool | Purpose |
|---|---|
| **SQL Server Developer Edition** | Warehouse storage and query execution (free for development) |
| **SSMS** | DDL execution, query development, table inspection |
| **Visual Studio + SSDT** | SSIS package authoring and debugging |
| **SSIS** | CSV ingestion, data transformation, dimension and fact loading |
| **Git** | Version control for all project files |

---

## 3. Warehouse Architecture

### Modeling Approach

The warehouse uses **Kimball Dimensional Modeling**:

- A central fact table stores all numeric measures at the grain of one hourly observation.
- Dimension tables store descriptive context for filtering and grouping.
- All dimensions use integer surrogate keys. The fact table references dimensions only via surrogate keys.

### Grain

> **One row in `fact_weather_observation` = one weather observation recorded at a specific date and hour.**

### Star Schema

```
              ┌──────────────────┐
              │    dim_date      │
              │──────────────────│
              │ date_key (PK)    │
              │ full_date        │
              │ year             │
              │ quarter          │
              │ month_number     │
              │ month_name       │
              │ day_name         │
              │ is_weekend       │
              │ season           │
              └────────┬─────────┘
                       │
       ┌───────────────┼───────────────┐
       │               │               │
┌──────┴───────┐        │     ┌─────────┴──────────┐
│   dim_time   │        │     │ dim_weather_cond    │
│──────────────│        │     │────────────────────│
│ time_key(PK) │   ┌────┴───┐ │ condition_key (PK) │
│ hour_value   │   │  FACT  │ │ summary            │
│ hour_label   │   │ TABLE  │ │ precip_type        │
│ time_of_day  │   └────────┘ └────────────────────┘
└──────────────┘
                  fact_weather_observation
                  ├─ observation_key (PK)
                  ├─ date_key (FK)
                  ├─ time_key (FK)
                  ├─ weather_condition_key (FK)
                  ├─ observation_datetime
                  ├─ temperature_c
                  ├─ apparent_temperature_c
                  ├─ apparent_temp_gap
                  ├─ humidity
                  ├─ wind_speed_kmh
                  ├─ wind_bearing_degrees
                  ├─ visibility_km
                  ├─ cloud_cover
                  ├─ pressure_millibars
                  └─ etl_load_datetime
```

### Table Justifications

| Table | Why It Exists |
|---|---|
| `stg_weather_raw` | Staging landing zone — see "Why Staging Exists" below |
| `dim_date` | Enables grouping by year, month, quarter, season, and weekday without parsing timestamps at query time |
| `dim_time` | Enables time-of-day banding (Morning / Afternoon / Evening / Night) needed for BQ-15 and BQ-20; populated once via T-SQL, not SSIS |
| `dim_weather_condition` | Normalizes free-text `Summary` and `Precip Type` strings into a single deduplicated reference table; eliminates string redundancy across 96,000 fact rows |
| `fact_weather_observation` | Central table storing all 8 numeric measures at the hourly grain; the source for all 23 business questions |
| `dq_rejected_rows` | Captures rows that fail data quality rules so they can be inspected without polluting the warehouse |
| `etl_validation` | Simple audit log recording row counts per ETL run, enabling reconciliation between staging and warehouse |

### Why Staging Exists

The staging table `stg_weather_raw` is not a copy of the CSV. It is a deliberate landing zone that separates three concerns:

1. **Ingestion isolation.** SSIS loads the raw file once into staging. If something breaks downstream in Package 2, the data is already in the database and Package 1 does not need to re-run against the file.
2. **DQ validation.** All data quality checks run against staging rows, not against the source file. Rejected rows are written to `dq_rejected_rows` without touching the warehouse tables.
3. **Re-runability.** Staging is truncated at the start of every run. This makes the pipeline safe to re-execute — Package 2 always reads from the same clean staging snapshot.

Without staging, SSIS would read the CSV and write directly to the fact table in a single pass. This gives no checkpoint for validation, no rejection log, and no way to re-run the warehouse load independently of the file read.

### Table Definitions

---

#### `stg_weather_raw`

All columns loaded as `NVARCHAR` to prevent SSIS type conversion failures on dirty data. Type casting is applied in Package 2 after DQ validation.

| Column | Type | Notes |
|---|---|---|
| `stg_id` | INT IDENTITY | Staging row identifier |
| `formatted_date` | NVARCHAR(50) | Raw timestamp string from CSV |
| `summary` | NVARCHAR(100) | Hourly weather description |
| `precip_type` | NVARCHAR(20) | May be NULL in source |
| `temperature_c` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `apparent_temperature_c` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `humidity` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `wind_speed_kmh` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `wind_bearing_degrees` | NVARCHAR(20) | Cast to SMALLINT during DQ |
| `visibility_km` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `cloud_cover` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `pressure_millibars` | NVARCHAR(20) | Cast to DECIMAL during DQ |
| `daily_summary` | NVARCHAR(500) | Daily narrative description |
| `stg_load_datetime` | DATETIME | Set by SSIS on load |

---

#### `dim_date`

Populated once via a T-SQL script covering the full dataset date range. Not loaded by SSIS.

| Column | Type | Description |
|---|---|---|
| `date_key` | INT | YYYYMMDD integer (e.g., 20160101) — standard Kimball pattern |
| `full_date` | DATE | Calendar date |
| `year` | SMALLINT | Calendar year |
| `quarter` | TINYINT | 1–4 |
| `month_number` | TINYINT | 1–12 |
| `month_name` | NVARCHAR(10) | e.g., January |
| `day_name` | NVARCHAR(10) | e.g., Monday |
| `is_weekend` | BIT | 1 if Saturday or Sunday |
| `season` | NVARCHAR(10) | Spring / Summer / Autumn / Winter |

---

#### `dim_time`

Contains exactly 24 rows (one per hour). Populated once via T-SQL `INSERT`.

| Column | Type | Description |
|---|---|---|
| `time_key` | TINYINT | Hour value 0–23 (serves as both PK and natural key) |
| `hour_value` | TINYINT | Hour 0–23 |
| `hour_label` | CHAR(8) | e.g., 02:00 AM |
| `time_of_day_band` | NVARCHAR(15) | Night (0–5) / Morning (6–11) / Afternoon (12–17) / Evening (18–23) |

---

#### `dim_weather_condition`

Deduplicated on `(summary, precip_type)`. Loaded via MERGE in Package 2.

| Column | Type | Description |
|---|---|---|
| `weather_condition_key` | INT IDENTITY | Surrogate PK |
| `summary` | NVARCHAR(100) | Hourly condition label (e.g., "Partly Cloudy") |
| `precip_type` | NVARCHAR(20) | rain / snow / None |

---

#### `fact_weather_observation`

| Column | Type | Description |
|---|---|---|
| `observation_key` | BIGINT IDENTITY | Surrogate PK |
| `date_key` | INT | FK → `dim_date` |
| `time_key` | TINYINT | FK → `dim_time` |
| `weather_condition_key` | INT | FK → `dim_weather_condition` |
| `observation_datetime` | DATETIME | Full parsed timestamp |
| `temperature_c` | DECIMAL(6,2) | Actual temperature °C |
| `apparent_temperature_c` | DECIMAL(6,2) | Feels-like temperature °C |
| `apparent_temp_gap` | DECIMAL(6,2) | `apparent_temperature_c − temperature_c` |
| `humidity` | DECIMAL(5,4) | 0.0000 to 1.0000 |
| `wind_speed_kmh` | DECIMAL(7,2) | km/h |
| `wind_bearing_degrees` | SMALLINT | 0–360 |
| `visibility_km` | DECIMAL(7,2) | km |
| `cloud_cover` | DECIMAL(5,4) | 0.0000 to 1.0000 |
| `pressure_millibars` | DECIMAL(8,2) | mb |
| `etl_load_datetime` | DATETIME | Row insert timestamp |

---

#### `dq_rejected_rows`

| Column | Type | Description |
|---|---|---|
| `rejection_key` | INT IDENTITY | Surrogate PK |
| `stg_id` | INT | Reference to `stg_weather_raw.stg_id` |
| `formatted_date` | NVARCHAR(50) | Raw date value for inspection |
| `rejection_reason_code` | NVARCHAR(20) | e.g., DQ_NULL_DATE, DQ_TEMP_RANGE |
| `rejection_reason_desc` | NVARCHAR(200) | Human-readable description |
| `rejected_datetime` | DATETIME | Timestamp of rejection |

---

#### `etl_validation`

| Column | Type | Description |
|---|---|---|
| `validation_key` | INT IDENTITY | Surrogate PK |
| `batch_id` | INT | ETL run identifier |
| `execution_start` | DATETIME | Package start time |
| `execution_end` | DATETIME | Package end time |
| `staging_row_count` | INT | Rows loaded to `stg_weather_raw` |
| `dq_pass_count` | INT | Rows passing all DQ rules |
| `dq_reject_count` | INT | Rows written to `dq_rejected_rows` |
| `fact_row_count` | INT | Rows loaded to `fact_weather_observation` |
| `execution_status` | NVARCHAR(10) | Success / Failure |

---

## 4. Phase 1 — Dataset Exploration

**Objective:** Understand the source data before any design work begins.  
**Deliverable:** `eda_findings.md`  
**Dependency:** `weatherHistory.csv` available

**Tasks:**
- Open the CSV and verify delimiter, encoding, and header row
- Count total rows (expected ~96,000)
- Identify the full date range (min and max `Formatted Date`)
- Profile `Summary` and `Precip Type`: distinct values and null counts
- Profile all 8 numeric columns: min, max, mean, and null count
- Check for duplicate timestamps
- Document all findings in `eda_findings.md`

**Acceptance Criteria:**
- [ ] Total row count documented
- [ ] Date range (min, max) documented
- [ ] Null counts per column documented
- [ ] Min, max, and mean for all 8 measure columns documented
- [ ] Duplicate check result documented

---

## 5. Phase 2 — Requirements Definition

**Objective:** Define and document the business requirements and analytical questions for the warehouse.  
**Deliverable:** `requirements.md`  
**Dependency:** Phase 1 complete

**Tasks:**
- Define the business problem (why a CSV is insufficient)
- Define audience and primary use cases
- Define scope: grain, date range, dimensions, and measurable facts
- Write 4 functional requirements with acceptance criteria
- Write 3 non-functional requirements
- Write 23 business questions grouped by category
- Map each question to required warehouse tables
- Define out-of-scope items

**Acceptance Criteria:**
- [ ] All sections of `requirements.md` complete
- [ ] 23 business questions documented with table mappings
- [ ] Functional and non-functional requirements each include an acceptance criterion

---

## 6. Phase 3 — Warehouse Design

**Objective:** Create all SQL DDL for the database, staging, warehouse, and audit tables.  
**Deliverable:** `create_tables.sql`  
**Dependency:** Phase 2 complete

**Tasks:**
- Create the `WeatherDWH` database
- Write DDL for `stg_weather_raw`
- Write DDL for `dim_date`; populate all dates in the dataset range using a T-SQL tally-table script including season derivation
- Write DDL for `dim_time`; populate 24 rows (hours 0–23) with time-of-day bands
- Write DDL for `dim_weather_condition`
- Write DDL for `fact_weather_observation` including the computed column `apparent_temp_gap`
- Write DDL for `dq_rejected_rows`
- Write DDL for `etl_validation`
- Add indexes on all FK columns in `fact_weather_observation`
- Execute `create_tables.sql` in SSMS and verify all objects are created

**Season derivation:**
```sql
CASE
    WHEN month_number IN (3, 4, 5)   THEN 'Spring'
    WHEN month_number IN (6, 7, 8)   THEN 'Summer'
    WHEN month_number IN (9, 10, 11) THEN 'Autumn'
    ELSE 'Winter'
END AS season
```

**Acceptance Criteria:**
- [ ] `create_tables.sql` executes without errors
- [ ] All 7 tables created
- [ ] `dim_date` populated for full date range; `dim_time` contains exactly 24 rows
- [ ] FK constraints and indexes in place on `fact_weather_observation`

---

## 7. Phase 4 — SSIS Package 1: Staging Load

**Objective:** Build the SSIS package that reads `weatherHistory.csv` and loads all rows into `stg_weather_raw`.  
**Deliverable:** `Package1_Staging.dtsx`  
**Dependency:** Phase 3 complete; `stg_weather_raw` exists

**Tasks:**
- Create a new Integration Services project in Visual Studio (`WeatherDWH_SSIS`)
- Add a Flat File Connection Manager for `weatherHistory.csv` (comma-delimited, all columns as `DT_WSTR` length 500)
- Add an OLE DB Connection Manager targeting `WeatherDWH` on the local SQL Server instance
- Add an Execute SQL Task to `TRUNCATE TABLE stg_weather_raw` at the start of the control flow (ensures re-runability)
- Add a Data Flow Task: Flat File Source → Derived Column (add `stg_load_datetime = GETDATE()`) → OLE DB Destination (`stg_weather_raw`)
- After the Data Flow, insert a row into `etl_validation` with the staging row count and execution start time

**Acceptance Criteria:**
- [ ] Package executes without errors in Visual Studio debug mode
- [ ] `stg_weather_raw` row count equals source CSV row count (excluding header)
- [ ] `stg_load_datetime` populated on all rows
- [ ] Re-running the package produces the same row count

---

## 8. Phase 5 — SSIS Package 2: Warehouse Build

**Objective:** Apply DQ validation, load dimensions, and load the fact table from `stg_weather_raw`.  
**Deliverable:** `Package2_Warehouse.dtsx`  
**Dependency:** Phase 4 complete; `stg_weather_raw` populated

**Tasks:**
- Add a new package `Package2_Warehouse` to the SSIS project
- Add a Data Flow Task reading from `stg_weather_raw`
- Use a **Conditional Split** transformation to route rows: valid rows continue to the warehouse pipeline; rows failing any DQ rule are routed to `dq_rejected_rows` with a rejection reason code
- DQ rules to implement in the Conditional Split: null `formatted_date`; temperature outside −60 to +60°C; humidity outside 0–1; negative wind speed; pressure outside 870–1085 mb; negative visibility
- Load `dim_weather_condition` using a MERGE statement on `(summary, precip_type)` to avoid duplicates
- Use OLE DB Lookup transformations to resolve `date_key`, `time_key`, and `weather_condition_key` for each valid row
- Add a Derived Column transformation to compute `apparent_temp_gap = apparent_temperature_c − temperature_c`
- Load valid, enriched rows into `fact_weather_observation`
- Update `etl_validation` with final counts (`dq_pass_count`, `dq_reject_count`, `fact_row_count`, `execution_end`, `execution_status`)

**Acceptance Criteria:**
- [ ] Package executes without errors
- [ ] `fact_weather_observation` count + `dq_rejected_rows` count = `stg_weather_raw` count
- [ ] All FK values in `fact_weather_observation` resolve to valid dimension keys
- [ ] `apparent_temp_gap` populated on all fact rows
- [ ] `etl_validation` row updated with `'Success'` status
- [ ] Re-running both packages produces identical row counts

---

## 9. Phase 6 — Validation and Testing

**Objective:** Verify that the ETL pipeline loaded the warehouse correctly.  
**Deliverable:** `test_results.md`  
**Dependency:** Phase 5 complete

**Validation Checks:**

| Test | SQL | Expected |
|---|---|---|
| Staging row count matches CSV | `SELECT COUNT(*) FROM stg_weather_raw` | Equals source row count |
| DQ reconciliation | `fact count + dq_rejected count` | Equals staging count |
| No duplicate observations | `GROUP BY date_key, time_key HAVING COUNT(*) > 1` | Zero rows |
| No null foreign keys in fact | `WHERE date_key IS NULL OR time_key IS NULL OR weather_condition_key IS NULL` | Zero rows |
| Orphan date_key check | Left join fact → dim_date, filter unmatched | Zero rows |
| Orphan time_key check | Left join fact → dim_time, filter unmatched | Zero rows |
| No invalid temperature in fact | `WHERE temperature_c < -60 OR temperature_c > 60` | Zero rows |
| No invalid humidity in fact | `WHERE humidity < 0 OR humidity > 1` | Zero rows |
| dim_time row count | `SELECT COUNT(*) FROM dim_time` | Exactly 24 |
| ETL audit record | `SELECT * FROM etl_validation` | Rows present; status = 'Success'; counts reconcile |
| Idempotency | Run both packages twice; compare counts | Identical before and after second run |

**Acceptance Criteria:**
- [ ] All 11 checks documented in `test_results.md` with actual vs. expected values
- [ ] Zero failing checks

---

## 10. Phase 7 — Analytical SQL

**Objective:** Write SQL queries that answer all 23 business questions and create three reporting views.  
**Deliverable:** `analytical_queries.sql`  
**Dependency:** Phase 6 complete; all validation checks passed

**Tasks:**
- Create `analytical_queries.sql` with section headers for each question category
- Write one query per business question (BQ-01 through BQ-23), each prefixed with the question ID comment (e.g., `-- BQ-01`)
- Create view `vw_monthly_weather_summary`: monthly aggregates for temperature, humidity, wind speed, and precipitation count
- Create view `vw_annual_trends`: year-over-year averages for all key measures
- Spot-check at least 5 query results against `stg_weather_raw` to confirm accuracy

**Acceptance Criteria:**
- [ ] `analytical_queries.sql` contains all 23 queries, each with a `-- BQ-XX` comment
- [ ] All queries execute without errors in SSMS
- [ ] Views `vw_monthly_weather_summary` and `vw_annual_trends` created and return correct columns
- [ ] At least 5 queries validated against staging data

---

## 11. Deliverables Summary

| # | File | Phase |
|---|---|---|
| 1 | `eda_findings.md` | Phase 1 |
| 2 | `requirements.md` | Phase 2 |
| 3 | `create_tables.sql` | Phase 3 |
| 4 | `Package1_Staging.dtsx` | Phase 4 |
| 5 | `Package2_Warehouse.dtsx` | Phase 5 |
| 6 | `test_results.md` | Phase 6 |
| 7 | `analytical_queries.sql` | Phase 7 |

---

## 12. Phase Sequencing

Each phase must be fully complete before the next begins.

```
Phase 1 (Dataset Exploration)
    │
    └──► Phase 2 (Requirements Definition)
              │
              └──► Phase 3 (Warehouse Design + DDL)
                        │
                        └──► Phase 4 (SSIS Package 1 — Staging Load)
                                  │
                                  └──► Phase 5 (SSIS Package 2 — Warehouse Build)
                                            │
                                            └──► Phase 6 (Validation and Testing)
                                                      │
                                                      └──► Phase 7 (Analytical SQL)
```

---

*End of Document — Weather Analytics Data Warehouse Project Plan v2.0*
