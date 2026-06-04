# Business Requirements — Weather Analytics Data Warehouse

**Document Version:** 2.0  
**Date:** June 4, 2026  
**Status:** Approved  

---

## Table of Contents

1. [Business Problem](#1-business-problem)
2. [Purpose and Audience](#2-purpose-and-audience)
3. [Scope](#3-scope)
4. [Functional Requirements](#4-functional-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements)
6. [Business Questions](#6-business-questions)
7. [Report-to-Table Traceability](#7-report-to-table-traceability)
8. [Out of Scope](#8-out-of-scope)

---

## 1. Business Problem

Raw weather observation data stored in a flat CSV file is difficult to analyze directly for three reasons:

- **Flat, denormalized structure.** Date, time, weather conditions, and all numeric measures are packed into a single row. Slicing by year, season, or condition type requires repetitive transformations on every query.
- **No temporal hierarchy.** The `Formatted Date` column is a raw timestamp string. Grouping by month, quarter, or season requires string parsing at query time, which is error-prone and slow.
- **Inconsistent and missing values.** Fields like `Precip Type` contain nulls that propagate silently into aggregates. Without a validation layer, query results are unreliable.

A Data Warehouse with a Star Schema and an SSIS ETL pipeline resolves all three by separating concerns, enforcing data quality at ingestion, and pre-computing temporal hierarchies in a dedicated date dimension.

---

## 2. Purpose and Audience

This warehouse ingests, validates, and stores historical weather observations in a structured format suitable for analytical SQL queries. It is a learning project demonstrating Kimball Dimensional Modeling, SSIS ETL, staging layer design, and data quality enforcement.

| Persona | Primary Use Case |
|---|---|
| **Meteorologists** | Review historical weather patterns and investigate anomalies |
| **Climate Researchers** | Study long-term trends and seasonal variability |
| **Transportation Analysts** | Understand how visibility, wind, and precipitation affect safety |
| **Agricultural Analysts** | Correlate temperature, humidity, and precipitation with growing seasons |
| **Students and Educators** | Learn Data Warehouse design, ETL, and analytical SQL on a real dataset |

---

## 3. Scope

| Attribute | Detail |
|---|---|
| **Source File** | `weatherHistory.csv` (~96,000 hourly observations) |
| **Date Range** | Approximately 2006–2016 (confirmed during Phase 1) |
| **Grain** | One row = one weather observation at a specific date and hour |
| **Lowest Unit** | Hour |
| **Highest Unit** | Year |

**Dimensions available for analysis:**

| Dimension | Key Attributes |
|---|---|
| Date | Year, Quarter, Month, Season, Day Name, Is Weekend |
| Time | Hour, Time of Day Band (Morning / Afternoon / Evening / Night) |
| Weather Condition | Summary, Precipitation Type |

**Measurable facts stored per observation:**

Temperature (°C), Apparent Temperature (°C), Humidity, Wind Speed (km/h), Wind Bearing (degrees), Visibility (km), Cloud Cover, Pressure (millibars)

---

## 4. Functional Requirements

**FR-01 — Data Ingestion**  
The SSIS package `Package1_Staging.dtsx` must read `weatherHistory.csv` and load all rows into `stg_weather_raw`, preserving raw values without transformation.  
*Acceptance criterion:* Row count in `stg_weather_raw` equals the row count in the source CSV (excluding the header row).

**FR-02 — Data Quality Validation**  
Before any warehouse table is loaded, each row in staging must be validated. Rows that fail must be written to `dq_rejected_rows` with a rejection reason and excluded from the fact table.  
*DQ rules:* `Formatted Date` must be non-null and parseable; Temperature must be −60 to +60°C; Humidity must be 0.00–1.00; Wind Speed must be ≥ 0; Pressure must be 870–1085 mb; Visibility must be ≥ 0. Null `Precip Type` is permitted and treated as `'None'`.  
*Acceptance criterion:* `fact_weather_observation` count + `dq_rejected_rows` count = `stg_weather_raw` count.

**FR-03 — Analytical Reporting**  
The warehouse must support all 23 business questions in Section 6. All questions must be answerable using SQL queries against the warehouse layer — no query should need to reference `stg_weather_raw` directly.  
*Acceptance criterion:* All 23 queries in `analytical_queries.sql` execute without errors and return results.

**FR-04 — Re-Runnable ETL**  
Both SSIS packages must be safe to re-execute. `stg_weather_raw` must be truncated at the start of each run. Dimension tables must use upsert logic. The fact table must be truncated and reloaded on re-run.  
*Acceptance criterion:* Running both packages twice in sequence produces identical row counts in all warehouse tables.

---

## 5. Non-Functional Requirements

**NFR-01 — Re-Runability**  
The ETL pipeline must be idempotent. Executing the full pipeline multiple times against the same source file must produce identical results with no duplicate rows in any warehouse table.

**NFR-02 — Data Accuracy**  
The sum of rows in `fact_weather_observation` and `dq_rejected_rows` must always equal the total rows in `stg_weather_raw`. This reconciliation is the primary correctness check for the ETL pipeline.

**NFR-03 — Maintainability**  
All SQL DDL must be in a single file `create_tables.sql`. All analytical queries must be in `analytical_queries.sql`, each prefixed with the matching question ID (e.g., `-- BQ-01`). All table and column names must follow `snake_case` with no spaces or ambiguous abbreviations.

---

## 6. Business Questions

### 6.1 Descriptive Analytics

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-01 | What was the highest recorded temperature, and on which date did it occur? | `fact_weather_observation`, `dim_date` |
| BQ-02 | What was the lowest recorded temperature, and on which date did it occur? | `fact_weather_observation`, `dim_date` |
| BQ-03 | What is the overall average temperature across all observations? | `fact_weather_observation` |
| BQ-04 | What was the maximum wind speed ever recorded, and what weather condition was associated with it? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-05 | What was the minimum recorded visibility, and on which date did it occur? | `fact_weather_observation`, `dim_date` |
| BQ-06 | What is the distribution of precipitation types (rain, snow, none) across all observations? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-07 | Is there a relationship between temperature and humidity? Show average humidity grouped by temperature bands (below 0°C, 0–10°C, 10–20°C, above 20°C). | `fact_weather_observation` |
| BQ-08 | Does high wind speed correspond to lower visibility? Show average visibility grouped by wind speed bands (0–10, 10–20, 20–30, above 30 km/h). | `fact_weather_observation` |

---

### 6.2 Trend Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-09 | How did average temperature vary by calendar month across all years? | `fact_weather_observation`, `dim_date` |
| BQ-10 | How did average wind speed vary by year? | `fact_weather_observation`, `dim_date` |
| BQ-11 | What is the average temperature by season (Spring, Summer, Autumn, Winter)? | `fact_weather_observation`, `dim_date` |
| BQ-12 | How does average temperature differ between weekdays and weekends? | `fact_weather_observation`, `dim_date` |

---

### 6.3 Comparative Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-13 | Compare average temperature, humidity, and wind speed on rainy days versus non-rainy days. | `fact_weather_observation`, `dim_weather_condition` |
| BQ-14 | Compare average temperature between Summer (June–August) and Winter (December–February). | `fact_weather_observation`, `dim_date` |
| BQ-15 | Compare average wind speed during daytime hours (6 AM–6 PM) versus nighttime hours. | `fact_weather_observation`, `dim_time` |
| BQ-16 | Which year had the highest average temperature, and which had the lowest? | `fact_weather_observation`, `dim_date` |

---

### 6.4 Weather Condition Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-17 | What are the top 10 most frequently occurring weather condition summaries? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-18 | What percentage of total observations recorded precipitation (rain or snow)? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-19 | What is the average visibility for each weather condition summary? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-20 | What is the most common weather condition during morning hours (6 AM–12 PM) versus evening hours (6 PM–12 AM)? | `fact_weather_observation`, `dim_weather_condition`, `dim_time` |

---

### 6.5 Anomaly Detection

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-21 | Identify observations where temperature exceeded the 99th percentile — extreme heat events. On which dates did they occur? | `fact_weather_observation`, `dim_date` |
| BQ-22 | Identify observations where temperature fell below the 1st percentile — extreme cold events. On which dates did they occur? | `fact_weather_observation`, `dim_date` |
| BQ-23 | Identify observations where visibility was below 1 km — dense fog or severe weather. How many occurred, and in which months were they most concentrated? | `fact_weather_observation`, `dim_date` |

---

## 7. Report-to-Table Traceability

| Question Category | Questions | Fact Table | Dimensions Used | Key Source Columns |
|---|---|---|---|---|
| Descriptive Analytics | BQ-01 to BQ-08 | `fact_weather_observation` | `dim_date`, `dim_weather_condition` | All measure columns; `Formatted Date`; `Summary`; `Precip Type` |
| Trend Analysis | BQ-09 to BQ-12 | `fact_weather_observation` | `dim_date` | `Formatted Date`; temperature; wind speed |
| Comparative Analysis | BQ-13 to BQ-16 | `fact_weather_observation` | `dim_date`, `dim_time`, `dim_weather_condition` | `Precip Type`; `Formatted Date`; all measure columns |
| Weather Condition Analysis | BQ-17 to BQ-20 | `fact_weather_observation` | `dim_weather_condition`, `dim_time` | `Summary`; `Precip Type`; `Formatted Date` |
| Anomaly Detection | BQ-21 to BQ-23 | `fact_weather_observation` | `dim_date` | Temperature; visibility; `Formatted Date` |

---

## 8. Out of Scope

| Item | Rationale |
|---|---|
| **Real-Time or Streaming Data** | Source is a static historical CSV file. No live feeds are required. |
| **Weather Forecasting or Machine Learning** | The warehouse is for descriptive and analytical reporting only. No predictive models are in scope. |
| **External Weather API Integration** | Only `weatherHistory.csv` is used as the data source. |
| **Automated Scheduling or Orchestration** | SSIS packages are executed manually. No SQL Server Agent jobs or pipeline automation are in scope. |
| **Cloud Deployment** | The warehouse runs on a local SQL Server Developer Edition instance only. |

---

*End of Document — Business Requirements v2.0*
