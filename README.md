# Weather Analytics Data Warehouse

A learning project demonstrating end-to-end Data Warehouse implementation using Kimball Dimensional Modeling, SSIS ETL, staging layer design, data quality enforcement, and analytical SQL on a real historical weather dataset.

---

## Dataset

| Attribute | Detail |
|---|---|
| **File** | `weatherHistory.csv` |
| **Size** | ~96,000 hourly weather observations |
| **Period** | Approximately 2006–2016 |
| **Grain** | One row per weather observation at a specific date and hour |

---

## Technology Stack

| Tool | Purpose |
|---|---|
| **SQL Server Developer Edition** | Warehouse storage and query execution (free for development) |
| **SSMS** | DDL execution, query development, table inspection |
| **Visual Studio + SSDT** | SSIS package authoring and debugging |
| **SSIS** | CSV ingestion, data transformation, dimension and fact loading |
| **Git** | Version control for all project files |

---

## Project Outcomes

When complete, this project delivers:

- A validated **Star Schema** in SQL Server with a staging layer, three dimension tables, one fact table, a DQ rejection log, and an ETL audit table
- A **two-package SSIS ETL pipeline** that ingests raw CSV data, applies data quality rules, loads dimensions and facts, and is safe to re-execute
- An **analytical query library** of 23 SQL queries answering business questions across descriptive, trend, comparative, condition, and anomaly detection categories
- A **data quality framework** that isolates rejected rows with reason codes and reconciles counts end-to-end

---

## Deliverables

### Actual Project Deliverables

These are the tangible outputs that prove the project is complete. Project planning tracks these.

| Deliverable | Description |
|---|---|
| **WeatherDWH database** | SQL Server database containing all warehouse tables, indexes, and static dimension data |
| **`create_tables.sql`** | Full DDL for all 7 tables with indexes and static data population scripts |
| **`Package1_Staging.dtsx`** | SSIS package that loads `weatherHistory.csv` into the staging table |
| **`Package2_Warehouse.dtsx`** | SSIS package that applies DQ validation, loads dimensions, and loads the fact table |
| **`analytical_queries.sql`** | 23 analytical SQL queries (BQ-01 to BQ-23) and two reporting views |

### Working Artifacts

These are documents created during the process. They are reference materials, not project outcomes.

| Artifact | Description |
|---|---|
| `eda_findings.md` | Dataset profiling results from Phase 1 |
| `requirements.md` | Business requirements, analytical questions, and scope |
| `architecture.md` | Warehouse design, star schema, and table definitions |
| `PLAN.md` | Phased execution plan with tasks and acceptance criteria |
| `test_results.md` | Validation test results from Phase 6 |

---

## Repository Structure

```
Population-DWH-Project/
│
├── README.md                    ← This file — project overview
├── requirements.md              ← What we are building and why
├── architecture.md              ← How the warehouse is designed
├── PLAN.md                      ← Phased execution plan
│
├── weatherHistory.csv           ← Source data file
│
├── create_tables.sql            ← DDL for all database tables and indexes
├── analytical_queries.sql       ← SQL queries answering all 23 business questions
│
├── eda_findings.md              ← Dataset exploration findings
├── test_results.md              ← Validation test results
│
└── WeatherDWH_SSIS/             ← Visual Studio SSIS project folder
    ├── Package1_Staging.dtsx    ← SSIS: CSV → staging table
    └── Package2_Warehouse.dtsx  ← SSIS: staging → dimensions + fact table
```

---

## Documentation Guide

| Document | Answers |
|---|---|
| `README.md` | What is this project? What does it produce? |
| `requirements.md` | What business problem does it solve? What questions must it answer? |
| `architecture.md` | How is the warehouse structured? Why does each table exist? |
| `PLAN.md` | What phases of work are required? What are the acceptance criteria? |
