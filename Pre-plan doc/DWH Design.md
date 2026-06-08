# Data Warehouse Design Draft

## Proposed Grain

One row in the fact table represents:

One weather observation recorded at a specific date and hour.

## Proposed Fact Table

fact_weather_observation

Candidate Measures:

* temperature_c
* apparent_temperature_c
* humidity
* wind_speed_kmh
* wind_bearing_degrees
* visibility_km
* cloud_cover
* pressure_millibars

## Proposed Dimensions

### dim_date

Possible Attributes:

* full_date
* year
* quarter
* month
* day_name
* season
* is_weekend

### dim_time (Needs Review)

Possible Attributes:

* hour
* time_of_day_band

Question:
Can hour be stored directly in fact instead?

### dim_weather_condition

Possible Attributes:

* summary
* precip_type

Purpose:
Reduce repetition of weather descriptions.

## Proposed Supporting Tables

### stg_weather_raw

Purpose:
Landing area for CSV data.

### dq_rejected_rows

Purpose:
Capture records failing validation rules.

### etl_validation

Purpose:
Track ETL execution metrics.

## Draft Star Schema

```
                dim_date
                    |
                    |
```

dim_weather_condition --+-- fact_weather_observation
|
|
dim_time (TBD)

## Open Design Questions

* Is dim_time justified?
* Is dim_weather_condition sufficient?
* Should daily_summary become a dimension?
* Are additional dimensions needed?
