# Requirement Understanding (Draft)

## Purpose

Build a Data Warehouse project using a weather dataset to learn:

* Dimensional Modeling
* ETL using SSIS
* SQL Server Database Design
* Data Quality Validation
* Analytical Querying

## Source Dataset

File:
weatherHistory.csv

Initial observations:

* Historical weather data
* Contains hourly weather observations
* Includes both descriptive and numerical attributes

## Business Goal

The CSV file contains weather data but is not optimized for reporting or analysis.

The goal is to transform the raw file into a structured warehouse that can answer analytical questions using SQL.

## Assumptions

* Dataset is static (not continuously updated)
* Historical analysis only
* Single source file
* No real-time processing needed

## Business Questions

*Descriptive Analytic
BQ-01: What was the highest recorded temperature, and on which date did it occur?
BQ-02: What was the lowest recorded temperature, and on which date did it occur?
BQ-03: What is the overall average temperature across all observations?
BQ-04: What was the maximum wind speed ever recorded, and what weather condition was associated with it?
BQ-05: What was the minimum recorded visibility, and on which date did it occur?
BQ-06: What is the distribution of precipitation types (rain, snow, none) across all observations?
BQ-07: Is there a relationship between temperature and humidity? Show average humidity grouped by temperature bands (below 0°C, 0–10°C, 10–20°C, above 20°C).
BQ-08: Does high wind speed correspond to lower visibility? Show average visibility grouped by wind speed bands (0–10, 10–20, 20–30, above 30 km/h).

* Trend Analysis
BQ-09: How did average temperature vary by calendar month across all years?
BQ-10: How did average wind speed vary by year?
BQ-11: What is the average temperature by season (Spring, Summer, Autumn, Winter)?
BQ-12: How does average temperature differ between weekdays and weekends?

* Comparative Analysis
BQ-13: Compare average temperature, humidity, and wind speed on rainy days versus non-rainy days.
BQ-14: Compare average temperature between Summer (June–August) and Winter (December–February).
BQ-15: Compare average wind speed during daytime hours (6 AM–6 PM) versus nighttime hours.
BQ-16: Which year had the highest average temperature, and which had the lowest?

* Weather Condition Analysis
BQ-17: What are the top 10 most frequently occurring weather condition summaries?
BQ-18: What percentage of total observations recorded precipitation (rain or snow)?
BQ-19: What is the average visibility for each weather condition summary?
BQ-20: What is the most common weather condition during morning hours (6 AM–12 PM) versus evening hours (6 PM–12 AM)?

* Anomaly Detection
BQ-21: Identify observations where temperature exceeded the 99th percentile (extreme heat events). On which dates did they occur?
BQ-22: Identify observations where temperature fell below the 1st percentile (extreme cold events). On which dates did they occur?
BQ-23: Identify observations where visibility was below 1 km (dense fog or severe weather). How many occurred, and in which months were they most concentrated?
