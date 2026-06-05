# Weather Analytics Data Warehouse — Architecture

**Document Version:** 1.0  
**Date:** June 5, 2026  
**Status:** Approved  

---

## Table of Contents

1. [Modeling Approach](#1-modeling-approach)
2. [Data Flow Overview](#2-data-flow-overview)
3. [Star Schema](#3-star-schema)
4. [Table Justifications](#4-table-justifications)
5. [Why Staging Exists](#5-why-staging-exists)
6. [Table Definitions](#6-table-definitions)

---

## 1. Modeling Approach

The warehouse uses **Kimball Dimensional Modeling**:

- A central **fact table** stores all numeric measures at the grain of one hourly observation.
- Surrounding **dimension tables** store descriptive context for filtering and grouping.
- All dimensions use **integer surrogate keys** as primary keys.
- The fact table references dimensions exclusively via surrogate keys — never by natural business keys.
- Slowly Changing Dimension handling is **Type 1** (overwrite). The source is a static historical file; no history-tracking is required.

**Grain:**

> One row in `fact_weather_observation` represents one weather observation recorded at a specific date and hour. This is the lowest level of detail available in the source dataset.

---

## 2. Data Flow Overview

```
weatherHistory.csv
        │
        │  Package1_Staging.dtsx
        │  (Flat File Source → OLE DB Destination)
        ▼
stg_weather_raw
        │
        │  Package2_Warehouse.dtsx
        │  (Conditional Split → DQ routing)
        │
        ├──── FAIL ──────────────────────► dq_rejected_rows
        │                                  (rows failing DQ rules)
        │
        └──── PASS ──► dim_weather_condition  (MERGE on summary + precip_type)
                   │
                   ├── Lookup: date_key    ◄── dim_date
                   ├── Lookup: time_key    ◄── dim_time
                   └── Lookup: condition_key ◄── dim_weather_condition
                   │
                   ▼
          fact_weather_observation
                   │
                   ▼
           etl_validation
           (row counts + execution status)
```

**Key design decisions visible in this flow:**

- Staging is a deliberate checkpoint between the file read and the warehouse write. See Section 5.
- DQ validation runs against staging rows, not the source file. Rejected rows are preserved in `dq_rejected_rows` for inspection.
- `dim_date` and `dim_time` are pre-populated via T-SQL before any SSIS run. They are not loaded by SSIS.
- All three dimension lookups occur inside Package 2 before the fact row is written, ensuring no orphan foreign keys.

---

## 3. Star Schema

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
│   dim_time   │        │     │  dim_weather_cond   │
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

---

## 4. Table Justifications

| Table | Why It Exists |
|---|---|
| `stg_weather_raw` | Landing zone between the CSV and the warehouse — see Section 5 for full justification |
| `dim_date` | Enables grouping by year, month, quarter, season, and weekday without parsing raw timestamps at query time; the `season` and `is_weekend` columns would otherwise require computation on every query |
| `dim_time` | Enables time-of-day banding (Morning / Afternoon / Evening / Night) required by business questions BQ-15 and BQ-20; contains exactly 24 static rows, populated once via T-SQL |
| `dim_weather_condition` | Normalises the free-text `Summary` and `Precip Type` strings from the CSV into a single deduplicated reference table; eliminates ~96,000 repeated string values in the fact table |
| `fact_weather_observation` | Central table storing all 8 numeric measures at the hourly grain; all 23 business questions query this table |
| `dq_rejected_rows` | Captures rows failing DQ validation with reason codes; allows inspection of bad data without losing it; enables the reconciliation rule: fact count + rejected count = staging count |
| `etl_validation` | Audit log recording row counts and execution status per ETL run; makes every pipeline execution traceable and verifiable via a single SQL query |

---

## 5. Why Staging Exists

The staging table `stg_weather_raw` is not simply a copy of the CSV loaded into a database table. It is a deliberate architectural layer that separates three concerns:

**1. Ingestion isolation**  
SSIS Package 1 reads the CSV file once and writes all rows to staging. If anything breaks in Package 2 (DQ validation, dimension lookups, fact loading), the data is already in the database. Package 1 does not need to re-read the file to re-run the warehouse load.

**2. Data quality validation**  
All DQ checks run against rows in `stg_weather_raw`, not against the source file. Rows that fail validation are written to `dq_rejected_rows` with a rejection reason code. The warehouse tables are never touched by invalid data.

**3. Re-runability**  
`stg_weather_raw` is truncated at the start of every Package 1 run. This means the pipeline is safe to re-execute at any time — Package 2 always reads from the same clean, complete staging snapshot regardless of how many times the pipeline has been run.

**Without staging**, SSIS would read the CSV and write directly to the fact table in a single data flow pass. This eliminates the DQ checkpoint, removes the ability to inspect rejected rows, and makes it impossible to re-run the warehouse load without re-reading the source file.

---

## 6. Table Definitions

### `stg_weather_raw`

All numeric columns are loaded as `NVARCHAR` to prevent SSIS type conversion failures on dirty data. Type casting is applied in Package 2 after DQ validation passes.

| Column | Type | Notes |
|---|---|---|
| `stg_id` | INT IDENTITY | Staging row identifier |
| `formatted_date` | NVARCHAR(50) | Raw timestamp string from CSV |
| `summary` | NVARCHAR(100) | Hourly weather description |
| `precip_type` | NVARCHAR(20) | May be NULL in source |
| `temperature_c` | NVARCHAR(20) | Cast to DECIMAL(6,2) during DQ |
| `apparent_temperature_c` | NVARCHAR(20) | Cast to DECIMAL(6,2) during DQ |
| `humidity` | NVARCHAR(20) | Cast to DECIMAL(5,4) during DQ |
| `wind_speed_kmh` | NVARCHAR(20) | Cast to DECIMAL(7,2) during DQ |
| `wind_bearing_degrees` | NVARCHAR(20) | Cast to SMALLINT during DQ |
| `visibility_km` | NVARCHAR(20) | Cast to DECIMAL(7,2) during DQ |
| `cloud_cover` | NVARCHAR(20) | Cast to DECIMAL(5,4) during DQ |
| `pressure_millibars` | NVARCHAR(20) | Cast to DECIMAL(8,2) during DQ |
| `daily_summary` | NVARCHAR(500) | Daily narrative description |
| `stg_load_datetime` | DATETIME | Set by SSIS on load |

---

### `dim_date`

Populated once via a T-SQL tally-table script covering the full dataset date range. Not loaded by SSIS.

| Column | Type | Description |
|---|---|---|
| `date_key` | INT | YYYYMMDD integer (e.g., 20160101) — standard Kimball pattern; human-readable and sorts naturally |
| `full_date` | DATE | Calendar date |
| `year` | SMALLINT | Calendar year |
| `quarter` | TINYINT | 1–4 |
| `month_number` | TINYINT | 1–12 |
| `month_name` | NVARCHAR(10) | e.g., January |
| `day_name` | NVARCHAR(10) | e.g., Monday |
| `is_weekend` | BIT | 1 if Saturday or Sunday |
| `season` | NVARCHAR(10) | Spring / Summer / Autumn / Winter |

Season is derived at population time using:

```sql
CASE
    WHEN month_number IN (3, 4, 5)   THEN 'Spring'
    WHEN month_number IN (6, 7, 8)   THEN 'Summer'
    WHEN month_number IN (9, 10, 11) THEN 'Autumn'
    ELSE 'Winter'
END
```

---

### `dim_time`

Contains exactly 24 rows (one per hour of the day). Populated once via T-SQL `INSERT`. Not loaded by SSIS.

| Column | Type | Description |
|---|---|---|
| `time_key` | TINYINT | Hour value 0–23; serves as both PK and natural key |
| `hour_value` | TINYINT | Hour 0–23 |
| `hour_label` | CHAR(8) | 12-hour label, e.g., 02:00 AM |
| `time_of_day_band` | NVARCHAR(15) | Night (0–5) / Morning (6–11) / Afternoon (12–17) / Evening (18–23) |

---

### `dim_weather_condition`

Deduplicated on the natural key `(summary, precip_type)`. Loaded via `MERGE` in Package 2 — each unique combination gets one surrogate key regardless of how many fact rows reference it.

| Column | Type | Description |
|---|---|---|
| `weather_condition_key` | INT IDENTITY | Surrogate PK |
| `summary` | NVARCHAR(100) | Hourly condition label, e.g., "Partly Cloudy" |
| `precip_type` | NVARCHAR(20) | rain / snow / None |

---

### `fact_weather_observation`

| Column | Type | Description |
|---|---|---|
| `observation_key` | BIGINT IDENTITY | Surrogate PK |
| `date_key` | INT | FK → `dim_date.date_key` |
| `time_key` | TINYINT | FK → `dim_time.time_key` |
| `weather_condition_key` | INT | FK → `dim_weather_condition.weather_condition_key` |
| `observation_datetime` | DATETIME | Full parsed observation timestamp |
| `temperature_c` | DECIMAL(6,2) | Actual temperature in degrees Celsius |
| `apparent_temperature_c` | DECIMAL(6,2) | Feels-like temperature in degrees Celsius |
| `apparent_temp_gap` | DECIMAL(6,2) | Computed: `apparent_temperature_c − temperature_c` |
| `humidity` | DECIMAL(5,4) | Ratio 0.0000–1.0000 |
| `wind_speed_kmh` | DECIMAL(7,2) | Wind speed in km/h |
| `wind_bearing_degrees` | SMALLINT | Wind direction 0–360 degrees |
| `visibility_km` | DECIMAL(7,2) | Visibility in km |
| `cloud_cover` | DECIMAL(5,4) | Ratio 0.0000–1.0000 |
| `pressure_millibars` | DECIMAL(8,2) | Atmospheric pressure in mb |
| `etl_load_datetime` | DATETIME | Row insert timestamp |

---

### `dq_rejected_rows`

| Column | Type | Description |
|---|---|---|
| `rejection_key` | INT IDENTITY | Surrogate PK |
| `stg_id` | INT | Reference to `stg_weather_raw.stg_id` |
| `formatted_date` | NVARCHAR(50) | Raw date value from the rejected staging row |
| `rejection_reason_code` | NVARCHAR(20) | Coded reason, e.g., DQ_NULL_DATE, DQ_TEMP_RANGE, DQ_HUMIDITY_RANGE |
| `rejection_reason_desc` | NVARCHAR(200) | Human-readable description of the failure |
| `rejected_datetime` | DATETIME | Timestamp when the row was rejected |

---

### `etl_validation`

| Column | Type | Description |
|---|---|---|
| `validation_key` | INT IDENTITY | Surrogate PK |
| `batch_id` | INT | ETL run identifier |
| `execution_start` | DATETIME | Package execution start time |
| `execution_end` | DATETIME | Package execution end time |
| `staging_row_count` | INT | Rows loaded to `stg_weather_raw` |
| `dq_pass_count` | INT | Rows passing all DQ rules |
| `dq_reject_count` | INT | Rows written to `dq_rejected_rows` |
| `fact_row_count` | INT | Rows loaded to `fact_weather_observation` |
| `execution_status` | NVARCHAR(10) | Success / Failure |

---

*End of Document — Warehouse Architecture v1.0*
