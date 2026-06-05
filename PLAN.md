# Weather Analytics Data Warehouse — Execution Plan

**Document Version:** 1.0  
**Date:** June 5, 2026  
**Status:** Active  

> This document covers execution only. For project background see `README.md`, for business requirements see `requirements.md`, and for warehouse design see `architecture.md`.

---

## Phase Sequencing

Each phase must be fully complete before the next begins.

```
Phase 1 (Dataset Exploration)
    │
    └──► Phase 2 (Requirements Definition)
              │
              └──► Phase 3 (Warehouse Design)
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

## Phase 1 — Dataset Exploration

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
- [ ] Duplicate timestamp check result documented

---

## Phase 2 — Requirements Definition

**Objective:** Define and document the business requirements and analytical questions for the warehouse.  
**Deliverable:** `requirements.md`  
**Dependency:** Phase 1 complete

**Tasks:**
- Define the business problem (why a CSV is insufficient for analysis)
- Define audience and primary use cases
- Define scope: grain, date range, dimensions, and measurable facts
- Write 4 functional requirements, each with an acceptance criterion
- Write 3 non-functional requirements
- Write 23 business questions grouped by category, each mapped to required warehouse tables
- Define out-of-scope items

**Acceptance Criteria:**
- [ ] All sections of `requirements.md` complete
- [ ] 23 business questions documented with table mappings
- [ ] Each functional requirement includes an acceptance criterion

---

## Phase 3 — Implementation Steps

**Objective:** Create all SQL DDL for the database, staging, warehouse, and audit tables.  
**Deliverable:** `create_tables.sql`  
**Dependency:** Phase 2 complete

**Tasks:**
- Create the `WeatherDWH` database in SQL Server
- Write DDL for `stg_weather_raw` (all columns as `NVARCHAR` — see `architecture.md`)
- Write DDL for `dim_date`; populate all dates in the dataset range using a T-SQL tally-table script
- Write DDL for `dim_time`; insert all 24 hours (0–23) with time-of-day band labels
- Write DDL for `dim_weather_condition`
- Write DDL for `fact_weather_observation` including the computed column `apparent_temp_gap`
- Write DDL for `dq_rejected_rows`
- Write DDL for `etl_validation`
- Add indexes on all FK columns in `fact_weather_observation`
- Execute `create_tables.sql` in SSMS and verify all objects are created

**Season derivation logic for `dim_date` population:**
```sql
CASE
    WHEN month_number IN (3, 4, 5)   THEN 'Spring'
    WHEN month_number IN (6, 7, 8)   THEN 'Summer'
    WHEN month_number IN (9, 10, 11) THEN 'Autumn'
    ELSE 'Winter'
END AS season
```

**Acceptance Criteria:**
- [ ] `create_tables.sql` executes without errors in SSMS
- [ ] All 7 tables created: `stg_weather_raw`, `dim_date`, `dim_time`, `dim_weather_condition`, `fact_weather_observation`, `dq_rejected_rows`, `etl_validation`
- [ ] `dim_date` populated for the full dataset date range
- [ ] `dim_time` contains exactly 24 rows
- [ ] FK constraints and indexes in place on `fact_weather_observation`

---

## Phase 4 — SSIS Package 1: Staging Load

**Objective:** Build the SSIS package that reads `weatherHistory.csv` and loads all rows into `stg_weather_raw`.  
**Deliverable:** `Package1_Staging.dtsx`  
**Dependency:** Phase 3 complete; `stg_weather_raw` exists in `WeatherDWH`

**Tasks:**
- Create a new Integration Services project in Visual Studio (name: `WeatherDWH_SSIS`)
- Add a Flat File Connection Manager for `weatherHistory.csv` (comma-delimited; all columns as `DT_WSTR` length 500)
- Add an OLE DB Connection Manager targeting `WeatherDWH` on the local SQL Server instance
- Add an Execute SQL Task at the start of the control flow: `TRUNCATE TABLE stg_weather_raw` (ensures re-runability)
- Add a Data Flow Task: Flat File Source → Derived Column (add `stg_load_datetime = GETDATE()`) → OLE DB Destination (`stg_weather_raw`)
- After the Data Flow, insert a row into `etl_validation` recording `staging_row_count` and `execution_start`

**Acceptance Criteria:**
- [ ] Package executes without errors in Visual Studio debug mode
- [ ] `stg_weather_raw` row count equals the source CSV row count (excluding the header row)
- [ ] `stg_load_datetime` is populated on all rows
- [ ] Re-running the package produces the same row count (no duplicates)

---

## Phase 5 — SSIS Package 2: Warehouse Build

**Objective:** Apply DQ validation, load dimensions, and load the fact table from `stg_weather_raw`.  
**Deliverable:** `Package2_Warehouse.dtsx`  
**Dependency:** Phase 4 complete; `stg_weather_raw` is populated

**Tasks:**
- Add a new package `Package2_Warehouse` to the existing SSIS project
- Add a Data Flow Task reading all rows from `stg_weather_raw`
- Use a **Conditional Split** transformation to route rows: rows failing any DQ rule route to `dq_rejected_rows`; all other rows continue to the warehouse pipeline
- DQ rules to implement: null `formatted_date`; temperature outside −60 to +60°C; humidity outside 0–1; negative wind speed; pressure outside 870–1085 mb; negative visibility
- Load `dim_weather_condition` using a `MERGE` statement on `(summary, precip_type)` to avoid duplicate dimension rows
- Use OLE DB Lookup transformations to resolve `date_key`, `time_key`, and `weather_condition_key` for each valid row
- Add a Derived Column transformation to compute `apparent_temp_gap = apparent_temperature_c − temperature_c`
- Load valid, enriched rows into `fact_weather_observation`
- Update `etl_validation` with final counts: `dq_pass_count`, `dq_reject_count`, `fact_row_count`, `execution_end`, `execution_status`

**Acceptance Criteria:**
- [ ] Package executes without errors in Visual Studio debug mode
- [ ] `fact_weather_observation` count + `dq_rejected_rows` count = `stg_weather_raw` count
- [ ] All FK values in `fact_weather_observation` resolve to valid dimension keys (no orphans)
- [ ] `apparent_temp_gap` is populated on all fact rows
- [ ] `etl_validation` row updated with `'Success'` status
- [ ] Re-running both packages twice in sequence produces identical row counts

---

## Phase 6 — Validation and Testing

**Objective:** Verify that the ETL pipeline loaded the warehouse correctly and all quality checks pass.  
**Deliverable:** `test_results.md`  
**Dependency:** Phase 5 complete; both SSIS packages have executed successfully

**Validation Checks:**

| Check | SQL Approach | Expected Result |
|---|---|---|
| Staging row count matches CSV | `SELECT COUNT(*) FROM stg_weather_raw` | Equals source file row count |
| DQ reconciliation | `fact count + dq_rejected count` | Equals staging count |
| No duplicate observations | `GROUP BY date_key, time_key HAVING COUNT(*) > 1` | Zero rows |
| No null foreign keys in fact | `WHERE date_key IS NULL OR time_key IS NULL OR weather_condition_key IS NULL` | Zero rows |
| Orphan date_key | Left join fact → dim_date, filter unmatched | Zero rows |
| Orphan time_key | Left join fact → dim_time, filter unmatched | Zero rows |
| No invalid temperature in fact | `WHERE temperature_c < -60 OR temperature_c > 60` | Zero rows |
| No invalid humidity in fact | `WHERE humidity < 0 OR humidity > 1` | Zero rows |
| dim_time row count | `SELECT COUNT(*) FROM dim_time` | Exactly 24 |
| ETL audit record present | `SELECT * FROM etl_validation` | Rows present; status = 'Success'; counts reconcile |
| Idempotency | Run both packages twice; compare all counts | Identical before and after second run |

**Acceptance Criteria:**
- [ ] All 11 checks executed and documented in `test_results.md` with actual vs. expected values
- [ ] Zero failing checks

---

## Phase 7 — Analytical SQL

**Objective:** Write SQL queries answering all 23 business questions and create two reporting views.  
**Deliverable:** `analytical_queries.sql`  
**Dependency:** Phase 6 complete; all validation checks passed

**Tasks:**
- Create `analytical_queries.sql` with section headers for each question category (Descriptive, Trend, Comparative, Condition, Anomaly)
- Write one query per business question BQ-01 through BQ-23, each prefixed with a comment matching the question ID (e.g., `-- BQ-01`)
- Create view `vw_monthly_weather_summary`: monthly aggregates for temperature, humidity, wind speed, and precipitation count
- Create view `vw_annual_trends`: year-over-year averages for all key measures
- Spot-check at least 5 query results against `stg_weather_raw` to confirm accuracy

**Acceptance Criteria:**
- [ ] `analytical_queries.sql` contains all 23 queries, each with a `-- BQ-XX` comment
- [ ] All queries execute without errors in SSMS
- [ ] Views `vw_monthly_weather_summary` and `vw_annual_trends` created and return correct columns
- [ ] At least 5 queries validated against staging data

---

*End of Document — Execution Plan v1.0*
