# Business Requirements — Weather Analytics Data Warehouse

**Document Version:** 1.0  
**Date:** June 4, 2026  
**Author:** Data Warehouse Team  
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

Raw weather observation data, as collected from sensors and meteorological instruments, presents significant analytical challenges when consumed directly in its source form:

- **Flat, denormalized structure.** A single CSV or text file containing ~96,000 rows mixes temporal attributes (date, time), categorical attributes (weather summary, precipitation type), and continuous measures (temperature, humidity, pressure) in one flat structure. This makes slicing and aggregating data difficult without heavy, repetitive SQL transformations at query time.

- **No temporal hierarchy.** The raw `Formatted Date` field is a timestamp string. Analysts cannot natively filter or group by year, month, quarter, season, or day-of-week without parsing and transforming that string on every query. This increases query complexity and degrades performance at scale.

- **Inconsistent and missing values.** Fields such as `Precip Type` contain null values for observations where precipitation did not occur. Without a staging validation layer, nulls propagate into analytical queries and produce misleading aggregates (e.g., incorrect averages or counts).

- **No categorical code management.** Weather condition labels (`Summary`, `Daily Summary`) are free-text strings stored inline with every row. This leads to high storage redundancy, inconsistent casing or spelling across observations, and no ability to group or filter conditions hierarchically.

- **No audit or lineage trail.** Raw files provide no mechanism to track when data was loaded, which rows were rejected due to quality failures, how many records were processed, or whether the data has changed between loads. This makes troubleshooting and compliance impossible.

- **Not optimized for analytical queries.** Analytical workloads—aggregations, rolling averages, seasonal comparisons, percentile calculations—require a query-optimized structure. A flat CSV cannot support indexed lookups, pre-computed aggregates, or join-based filtering across dimensions efficiently.

A purpose-built Data Warehouse with a structured Star Schema resolves all of the above. It separates concerns between temporal, categorical, and measurable data, enforces data quality at ingestion, supports repeatable ETL, and delivers a query-ready foundation for analytical reporting.

---

## 2. Purpose and Audience

### Purpose

This Data Warehouse is designed to ingest, validate, store, and expose historical weather observation data in a structured, query-optimized format. It supports descriptive analytics, trend analysis, comparative studies, and anomaly detection across multiple weather dimensions over the full historical period covered by the dataset.

The warehouse serves as a learning reference implementation demonstrating industry-standard Data Warehousing practices: Kimball Dimensional Modeling, SSIS-based ETL pipelines, staging layer design, data quality enforcement, and analytical SQL development.

### Intended Audience

| Persona | Role | Primary Use Case |
|---|---|---|
| **Meteorologists** | Domain experts analyzing atmospheric conditions | Review historical weather patterns, validate observation consistency, investigate anomalies |
| **Climate Researchers** | Academic or institutional researchers | Study long-term climate trends, seasonal variability, and extreme weather frequency |
| **Transportation Analysts** | Logistics and infrastructure planners | Understand how visibility, wind, and precipitation impact transportation safety and planning |
| **Agricultural Analysts** | Farming and crop management teams | Correlate temperature, humidity, and precipitation with growing seasons and harvest planning |
| **Students and Educators** | Data engineering and analytics learners | Study Data Warehouse architecture, ETL design, SQL analytics, and dimensional modeling using a real dataset |

---

## 3. Scope

### 3.1 Dataset

| Attribute | Detail |
|---|---|
| **Source File** | `weatherHistory.csv` |
| **Approximate Row Count** | ~96,000 observations |
| **Source Format** | Comma-separated values (CSV) |
| **Encoding** | UTF-8 |

### 3.2 Time Period

The dataset covers historical weather observations spanning multiple years. The exact date range will be confirmed during Phase 1 (Dataset Exploration). Based on the nature of the dataset, coverage is expected to span approximately **2006 to 2016**, providing roughly a decade of hourly weather observations.

### 3.3 Observation Granularity

| Attribute | Detail |
|---|---|
| **Grain** | One row per weather observation at a specific date and time |
| **Observation Frequency** | Hourly (one observation per hour) |
| **Lowest Temporal Unit** | Hour |
| **Highest Temporal Unit** | Year |

### 3.4 Historical Analysis Scope

- Full historical period in the source dataset
- No real-time or near-real-time data feeds
- No rolling window updates — full historical load with incremental extension capability

### 3.5 Reporting Dimensions

The warehouse exposes the following analytical dimensions for slicing and dicing measures:

| Dimension | Key Attributes |
|---|---|
| **Date** | Year, Quarter, Month, Month Name, Week Number, Day, Day Name, Is Weekend, Season |
| **Time** | Hour, Hour Label (AM/PM), Time of Day Band (Morning / Afternoon / Evening / Night) |
| **Weather Condition** | Summary, Precipitation Type, Daily Summary |

### 3.6 Measurable Facts

| Measure | Source Column |
|---|---|
| Temperature (°C) | `Temperature (C)` |
| Apparent Temperature (°C) | `Apparent Temperature (C)` |
| Humidity (ratio 0–1) | `Humidity` |
| Wind Speed (km/h) | `Wind Speed (km/h)` |
| Wind Bearing (degrees) | `Wind Bearing (degrees)` |
| Visibility (km) | `Visibility (km)` |
| Cloud Cover (ratio 0–1) | `Cloud Cover` |
| Pressure (millibars) | `Pressure (millibars)` |

---

## 4. Functional Requirements

### FR-01 — Data Ingestion

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-01 |
| **Name** | Data Ingestion from Flat File |
| **Priority** | Must Have |
| **Description** | The system must ingest the `weatherHistory.csv` flat file into a staging table (`stg_weather_raw`) via an SSIS package. The ingestion process must load all columns from the source file preserving raw data types. |
| **Acceptance Criteria** | Row count in `stg_weather_raw` equals row count in source file. All source columns are present. No data truncation occurs. Load timestamp is captured per batch. |
| **SSIS Package** | `Package1_Staging.dtsx` |
| **Staging Table** | `stg_weather_raw` |

### FR-02 — Data Quality Validation

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-02 |
| **Priority** | Must Have |
| **Description** | The system must validate each row in the staging table against a defined set of data quality rules before it is promoted to the warehouse layer. Rows failing validation must be written to `dq_rejected_rows` with a rejection reason code and excluded from dimension and fact loading. |
| **Validation Rules** | (1) `Formatted Date` must be non-null and parseable as a valid datetime. (2) `Temperature (C)` must be within the range −60°C to +60°C. (3) `Humidity` must be within the range 0.00 to 1.00. (4) `Wind Speed (km/h)` must be >= 0. (5) `Pressure (millibars)` must be within the range 870 to 1085. (6) `Visibility (km)` must be >= 0. (7) `Precip Type` null values are permitted and will be replaced with the surrogate label `'None'`. |
| **Acceptance Criteria** | All rejected rows are logged to `dq_rejected_rows` with `rejection_reason`. Zero invalid rows exist in `fact_weather_observation`. The `etl_validation` table records pass/fail counts per batch. |
| **SSIS Package** | `Package2_Warehouse.dtsx` |
| **DQ Table** | `dq_rejected_rows` |

### FR-03 — Historical Weather Analytics

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-03 |
| **Priority** | Must Have |
| **Description** | The warehouse must support all 37 business questions defined in Section 6. Fact and dimension tables must be structured and indexed to return query results within acceptable performance thresholds on the full ~96,000 row dataset. |
| **Acceptance Criteria** | All 37 business questions can be answered using SQL queries against the warehouse layer without referencing `stg_weather_raw`. |
| **Primary Tables** | `fact_weather_observation`, `dim_date`, `dim_time`, `dim_weather_condition` |

### FR-04 — Trend Reporting

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-04 |
| **Priority** | Must Have |
| **Description** | The warehouse must expose analytical views and pre-written SQL that support temporal trend analysis. Trends must be calculable at year, quarter, month, week, and day-of-week granularity. |
| **Acceptance Criteria** | Reporting views `vw_monthly_weather_summary` and `vw_annual_trends` are created and return correct aggregated values validated against the staging table. |
| **Deliverable** | `analytical_queries.sql` |

### FR-05 — Auditability and ETL Traceability

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-05 |
| **Priority** | Must Have |
| **Description** | Every ETL execution must be traceable. The `etl_validation` table must record: batch identifier, execution start and end timestamps, source row count, rows loaded to staging, rows passed DQ, rows rejected, rows loaded to fact table, and execution status (Success / Failure / Partial). |
| **Acceptance Criteria** | After every SSIS package execution, one row is inserted into `etl_validation`. Row counts in `etl_validation` reconcile with counts in `stg_weather_raw`, `fact_weather_observation`, and `dq_rejected_rows`. |
| **Audit Table** | `etl_validation` |

### FR-06 — Surrogate Key Management

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-06 |
| **Priority** | Must Have |
| **Description** | All dimension tables must use integer surrogate keys as primary keys. Fact table foreign keys must reference dimension surrogate keys. Natural keys (business keys) must be retained in dimension tables for traceability. |
| **Acceptance Criteria** | No fact table row references a natural key directly. All dimension lookups use surrogate key joins. |

### FR-07 — Re-Runnable ETL Design

| Attribute | Detail |
|---|---|
| **Requirement ID** | FR-07 |
| **Priority** | Must Have |
| **Description** | All SSIS packages must be designed for safe re-execution. Staging tables must be truncated at the start of each run. Dimension tables must use upsert (MERGE) logic to avoid duplicate inserts. Fact tables must be truncated and reloaded or use a deduplication key to prevent duplicate observations. |
| **Acceptance Criteria** | Running `Package1_Staging.dtsx` followed by `Package2_Warehouse.dtsx` twice in sequence produces identical row counts in all warehouse tables. |

---

## 5. Non-Functional Requirements

### NFR-01 — Re-Runability

The ETL pipeline must be idempotent. Executing the full pipeline multiple times against the same source file must produce identical results with no duplicate rows in any warehouse table. Staging tables must be truncated before each load. Dimension inserts must use MERGE or EXISTS checks.

### NFR-02 — Performance

All analytical SQL queries against the fully loaded warehouse (~96,000 fact rows) must complete within **5 seconds** on a standard developer workstation (minimum: 8 GB RAM, quad-core CPU, SQL Server Developer Edition). Dimension tables must be indexed on surrogate keys and natural business keys. The fact table must be indexed on all foreign key columns and the `observation_datetime` column.

### NFR-03 — Data Accuracy

The sum of rows in `fact_weather_observation` plus rows in `dq_rejected_rows` must equal the total rows loaded into `stg_weather_raw`, which must equal the total rows in the source CSV file (excluding the header row). This reconciliation must be verifiable via the `etl_validation` table.

### NFR-04 — Maintainability

- All SQL DDL statements must reside in a single, versioned `create_tables.sql` file with clear section headers.
- All analytical SQL must reside in `analytical_queries.sql` with a query ID comment matching the business question ID (e.g., `-- BQ-01`).
- SSIS packages must use parameterized connection managers to allow environment-specific configuration without modifying package internals.
- All table and column names must follow a consistent naming convention: `snake_case`, no spaces, no abbreviations that are not standard.

### NFR-05 — Portability

The warehouse must run on **SQL Server Developer Edition** (free) to ensure the project is reproducible without licensing cost. All SQL must be compatible with SQL Server 2016 or later. SSIS packages must target SQL Server Integration Services 2019 or later via Visual Studio.

### NFR-06 — Documentability

Every deliverable must be accompanied by the relevant documentation phase output. The project must be self-documenting through `eda_findings.md`, `requirements.md`, `ProjectPlan.md`, and `test_results.md` such that a new team member can onboard without verbal explanation.

---

## 6. Business Questions

### 6.1 Descriptive Analytics

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-01 | What was the highest recorded temperature in the dataset, and on which date did it occur? | `fact_weather_observation`, `dim_date`, `dim_time` |
| BQ-02 | What was the lowest recorded temperature in the dataset, and on which date did it occur? | `fact_weather_observation`, `dim_date`, `dim_time` |
| BQ-03 | What was the overall average temperature across all observations? | `fact_weather_observation` |
| BQ-04 | What was the highest recorded humidity value, and under what weather condition did it occur? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-05 | What was the maximum wind speed ever recorded, and what was the associated weather summary? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-06 | What was the minimum recorded visibility, and on which date did it occur? | `fact_weather_observation`, `dim_date` |
| BQ-07 | What is the overall average pressure across all observations? | `fact_weather_observation` |
| BQ-08 | How many total observations are in the dataset, and how many are after applying data quality filters? | `fact_weather_observation`, `dq_rejected_rows` |
| BQ-09 | What is the distribution of precipitation types across all observations? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-10 | What is the average apparent temperature gap (apparent temperature minus actual temperature) across all observations? | `fact_weather_observation` |

---

### 6.2 Trend Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-11 | How did average temperature vary by calendar month across all years? | `fact_weather_observation`, `dim_date` |
| BQ-12 | How did average humidity change month-over-month across the full dataset period? | `fact_weather_observation`, `dim_date` |
| BQ-13 | How did average wind speed vary by year? | `fact_weather_observation`, `dim_date` |
| BQ-14 | How did average visibility change across years — is there a long-term trend? | `fact_weather_observation`, `dim_date` |
| BQ-15 | What is the average temperature by season (Spring, Summer, Autumn, Winter)? | `fact_weather_observation`, `dim_date` |
| BQ-16 | How did average cloud cover change by quarter across the dataset? | `fact_weather_observation`, `dim_date` |
| BQ-17 | What is the average pressure reading by month? Does pressure follow a seasonal pattern? | `fact_weather_observation`, `dim_date` |
| BQ-18 | How does average temperature differ between weekdays and weekends? | `fact_weather_observation`, `dim_date` |

---

### 6.3 Comparative Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-19 | Compare average temperature, humidity, and wind speed on rainy days versus non-rainy days. | `fact_weather_observation`, `dim_weather_condition` |
| BQ-20 | Compare average visibility on snowy days versus rainy days versus days with no precipitation. | `fact_weather_observation`, `dim_weather_condition` |
| BQ-21 | Compare average temperature between Summer (June–August) and Winter (December–February). | `fact_weather_observation`, `dim_date` |
| BQ-22 | Compare average wind speed during daytime hours (6 AM–6 PM) versus nighttime hours (6 PM–6 AM). | `fact_weather_observation`, `dim_time` |
| BQ-23 | Compare average humidity across all four seasons. | `fact_weather_observation`, `dim_date` |
| BQ-24 | Which year had the highest average temperature, and which had the lowest? | `fact_weather_observation`, `dim_date` |
| BQ-25 | Compare average apparent temperature versus actual temperature by season — which season has the greatest wind-chill or heat-index gap? | `fact_weather_observation`, `dim_date` |

---

### 6.4 Weather Condition Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-26 | What are the top 10 most frequently occurring weather condition summaries? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-27 | What is the average wind speed for each distinct weather condition summary? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-28 | What percentage of total observations recorded precipitation (rain or snow)? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-29 | What is the average visibility for each weather condition summary? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-30 | What is the average cloud cover for each precipitation type (rain, snow, none)? | `fact_weather_observation`, `dim_weather_condition` |
| BQ-31 | What is the most common weather condition during morning hours (6 AM–12 PM) versus evening hours (6 PM–12 AM)? | `fact_weather_observation`, `dim_weather_condition`, `dim_time` |

---

### 6.5 Correlation Analysis

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-32 | Is there a relationship between temperature and humidity? As temperature increases, does humidity decrease? | `fact_weather_observation` |
| BQ-33 | Is there a relationship between atmospheric pressure and visibility? Do higher pressures correspond to better visibility? | `fact_weather_observation` |
| BQ-34 | Is there a relationship between wind speed and the apparent temperature gap (actual minus apparent temperature)? | `fact_weather_observation` |
| BQ-35 | Does cloud cover correlate with lower temperatures? Show average temperature grouped by cloud cover bands (0–0.2, 0.2–0.4, 0.4–0.6, 0.6–0.8, 0.8–1.0). | `fact_weather_observation` |
| BQ-36 | Does high wind speed correlate with lower visibility? Show average visibility grouped by wind speed bands. | `fact_weather_observation` |

---

### 6.6 Anomaly Detection

| Question ID | Business Question | Tables Required |
|---|---|---|
| BQ-37 | Identify observations where temperature exceeded the 99th percentile for the dataset — these represent extreme heat events. On which dates did they occur? | `fact_weather_observation`, `dim_date` |
| BQ-38 | Identify observations where temperature fell below the 1st percentile — these represent extreme cold events. On which dates did they occur? | `fact_weather_observation`, `dim_date` |
| BQ-39 | Identify observations where pressure was outside the normal range (below 960 or above 1040 millibars) — potential storm or high-pressure system events. | `fact_weather_observation`, `dim_date`, `dim_time` |
| BQ-40 | Identify observations where visibility was below 1 km — dense fog or severe weather events. How many occurred, and in which months were they most concentrated? | `fact_weather_observation`, `dim_date` |

---

## 7. Report-to-Table Traceability

The following matrix maps each analytical report category to the required warehouse tables, source CSV columns, and the ETL transformations applied during loading.

| Report Category | Business Questions | Fact Table | Dimension Tables | Source CSV Columns | ETL Transformations Applied |
|---|---|---|---|---|---|
| **Descriptive Analytics** | BQ-01 to BQ-10 | `fact_weather_observation` | `dim_date`, `dim_time`, `dim_weather_condition` | All measure columns; `Formatted Date`; `Summary`; `Precip Type` | Date/time parsing; NULL handling for `Precip Type` → `'None'`; type casting for all numeric measures |
| **Trend Analysis** | BQ-11 to BQ-18 | `fact_weather_observation` | `dim_date` | `Formatted Date`; temperature; humidity; wind speed; visibility; cloud cover; pressure | Date hierarchy derivation (Year, Quarter, Month, Season, Is Weekend) in `dim_date` |
| **Comparative Analysis** | BQ-19 to BQ-25 | `fact_weather_observation` | `dim_date`, `dim_time`, `dim_weather_condition` | `Precip Type`; `Formatted Date`; all measure columns | Season band assignment; precipitation type normalization; time-of-day band derivation in `dim_time` |
| **Weather Condition Analysis** | BQ-26 to BQ-31 | `fact_weather_observation` | `dim_weather_condition`, `dim_time` | `Summary`; `Daily Summary`; `Precip Type`; `Formatted Date` | Deduplication of condition labels; surrogate key assignment for `dim_weather_condition` |
| **Correlation Analysis** | BQ-32 to BQ-36 | `fact_weather_observation` | None (self-join on fact) | Temperature; humidity; pressure; visibility; wind speed; apparent temperature; cloud cover | Computed column `apparent_temp_gap = apparent_temperature_c - temperature_c` in fact table |
| **Anomaly Detection** | BQ-37 to BQ-40 | `fact_weather_observation` | `dim_date`, `dim_time` | Temperature; pressure; visibility; `Formatted Date` | No additional transformation; percentile and threshold logic applied at query time via `PERCENTILE_CONT` and `WHERE` filters |

### Detailed Column-Level Mapping

| Source CSV Column | Staging Column (`stg_weather_raw`) | Warehouse Destination | Transformation |
|---|---|---|---|
| `Formatted Date` | `formatted_date` (NVARCHAR) | `dim_date.full_date`, `dim_time.hour_value`, `fact_weather_observation.observation_datetime` | Parse to DATETIME; extract date part for `dim_date`; extract hour for `dim_time` |
| `Summary` | `summary` (NVARCHAR) | `dim_weather_condition.summary` | Trim whitespace; deduplicate; assign surrogate key |
| `Precip Type` | `precip_type` (NVARCHAR) | `dim_weather_condition.precip_type` | NULL → `'None'`; trim; normalize to title case |
| `Temperature (C)` | `temperature_c` (FLOAT) | `fact_weather_observation.temperature_c` | DQ range check: −60 to +60; reject if out of range |
| `Apparent Temperature (C)` | `apparent_temperature_c` (FLOAT) | `fact_weather_observation.apparent_temperature_c` | Cast to DECIMAL(6,2) |
| `Humidity` | `humidity` (FLOAT) | `fact_weather_observation.humidity` | DQ range check: 0.00 to 1.00; reject if out of range |
| `Wind Speed (km/h)` | `wind_speed_kmh` (FLOAT) | `fact_weather_observation.wind_speed_kmh` | DQ check: >= 0; reject if negative |
| `Wind Bearing (degrees)` | `wind_bearing_degrees` (FLOAT) | `fact_weather_observation.wind_bearing_degrees` | Cast to SMALLINT (0–360) |
| `Visibility (km)` | `visibility_km` (FLOAT) | `fact_weather_observation.visibility_km` | DQ check: >= 0; reject if negative |
| `Cloud Cover` | `cloud_cover` (FLOAT) | `fact_weather_observation.cloud_cover` | Cast to DECIMAL(4,2) |
| `Pressure (millibars)` | `pressure_millibars` (FLOAT) | `fact_weather_observation.pressure_millibars` | DQ range check: 870 to 1085; reject if out of range |
| `Daily Summary` | `daily_summary` (NVARCHAR) | `dim_weather_condition.daily_summary` | Trim; associate with matching `Summary` value |

---

## 8. Out of Scope

The following capabilities are explicitly excluded from this project. Their exclusion is intentional and based on the learning objectives of this implementation.

| Item | Rationale |
|---|---|
| **Real-Time or Near-Real-Time Data Streaming** | The project uses a static historical CSV file. No message queues, event streams, or live sensor feeds are in scope. Technologies such as Kafka, Azure Event Hub, or SQL Server Change Data Capture are not required. |
| **Weather Forecasting or Predictive Modeling** | The warehouse is designed for descriptive and analytical reporting against historical data only. No machine learning models, regression analysis, or forecasting algorithms are in scope. |
| **Machine Learning Integration** | No ML frameworks (e.g., Python scikit-learn, Azure ML, SQL Server ML Services) are in scope. |
| **External Weather API Integration** | No live data feeds from providers such as OpenWeatherMap, NOAA, or Dark Sky are in scope. The source is the static `weatherHistory.csv` file only. |
| **Multi-Source Data Integration** | Only the single `weatherHistory.csv` source file is in scope. No geographic, demographic, or economic data sources are integrated. |
| **Row-Level Security or Multi-Tenancy** | All users of the warehouse have equivalent read access. No role-based data masking or tenant isolation is implemented. |
| **Automated Scheduling or Orchestration** | No SQL Server Agent jobs, Windows Task Scheduler automation, or Azure Data Factory pipelines are in scope. SSIS packages are executed manually via Visual Studio or SSMS. |
| **High Availability or Disaster Recovery** | The project runs on a single SQL Server Developer Edition instance. No Always On Availability Groups, database mirroring, or backup/restore automation is in scope. |
| **Cloud Deployment** | The warehouse is deployed on a local SQL Server instance only. No Azure SQL, Azure Synapse Analytics, or cloud-based deployment is in scope. |
| **Power BI Service or Shared Dashboards** | Power BI reporting is listed as an optional Phase 8 deliverable using Power BI Desktop only. No Power BI Service, workspaces, or shared dashboard publishing is in scope. |

---

*End of Document — Business Requirements v1.0*
