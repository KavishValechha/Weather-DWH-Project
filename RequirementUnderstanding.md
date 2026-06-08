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

## Candidate Questions

* What was the highest recorded temperature?
* What was the lowest recorded temperature?
* What is the average temperature by season?
* How does humidity vary throughout the year?
* What weather conditions occur most frequently?
* How does wind speed affect visibility?

