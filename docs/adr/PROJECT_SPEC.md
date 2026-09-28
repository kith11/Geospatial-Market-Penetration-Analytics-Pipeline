# SILID --- Project Specification

> Weather Intelligence Platform for Productivity\
> Data Engineering Portfolio Project\
> Version 1.1 — Step 0 contract reconciliation

The [data contracts](DATA_CONTRACTS.md) define precise grains, identities, source
limitations, publication rules, and KPI denominators. Follow the
[delivery sequence](REQUIREMENT_TRACEABILITY.md) for execution order and the
[traceability matrix](REQUIREMENT_TRACEABILITY.md) for acceptance evidence.

## 1. Project Overview

SILID is a production-style Data Engineering platform that ingests,
cleans, models, and serves Philippine weather data to answer:

> **When should I do my most demanding work today?**

The platform treats weather as a time-series signal and converts raw
meteorological data into actionable productivity recommendations through
a governed, testable, and observable data pipeline.

SILID is intentionally **Data Engineering-first**. The main portfolio
value is the end-to-end engineering lifecycle rather than machine
learning.

## 2. Core Problem

Philippine weather conditions --- especially high temperature, humidity,
heat index, and unpredictable rainfall --- can affect study, knowledge
work, exercise, commuting, and outdoor activity.

Existing weather products primarily describe conditions. SILID adds an
analytical layer that transforms weather observations and forecasts into
scheduling signals.

## 3. Primary MVP Users

The MVP prioritizes:

1.  University / senior high school students
2.  Remote software engineers and other knowledge workers
3.  Freelancers / gig workers

The architecture should remain extensible to commuters, fitness users,
and outdoor workers.

## 4. Core User Questions

SILID should answer:

-   When should I study or work today?
-   Will conditions become too hot for comfortable deep work?
-   What are the best workout windows?
-   What is the short-horizon rain risk?
-   Is today's weather/productivity outlook better or worse than usual?

## 5. Value Proposition

The core value is **translation**:

``` text
raw third-party weather data
        ↓
reliable ingestion
        ↓
validated historical data
        ↓
engineered weather metrics
        ↓
productivity/workout scores
        ↓
ranked actionable windows
```

The platform should produce trustworthy, explainable recommendations
while preserving the underlying data as a reusable analytical asset.

------------------------------------------------------------------------

# 6. ⭐ Core Data Engineering Architecture

SILID uses a **Medallion Architecture**.

``` text
Weather APIs
    │
    ▼
┌──────────────────────────────┐
│ BRONZE                       │
│ Raw • Immutable • As-received│
└──────────────────────────────┘
    │
    │ cleaning / typing / dedup
    ▼
┌──────────────────────────────┐
│ SILVER                       │
│ Validated • Typed • Conformed│
└──────────────────────────────┘
    │
    │ metrics / business logic
    ▼
┌──────────────────────────────┐
│ GOLD                         │
│ Analytics-ready • Star schema│
└──────────────────────────────┘
    │
    ├──────────────► Streamlit
    │
    └──────────────► Monitoring
```

## 6.1 Bronze --- Raw Landing Zone

Responsibilities:

-   Preserve API responses exactly as received
-   Maintain immutable historical source data
-   Store ingestion metadata
-   Support replay and reprocessing
-   Preserve malformed responses through dead-letter handling where
    possible

Target storage:

-   MinIO locally
-   S3-compatible object storage in a future cloud deployment
-   Optional parsed PostgreSQL mirror for convenience, never the raw authority

Persist the response body before JSON decoding, with immutable metadata stored
separately. Preserve malformed bodies and retain legacy JSONB rows without claiming
that their original response representation can be recovered. See ADR-001.

### Core DE concept: Raw immutability

Bronze is the source-of-truth history from which downstream datasets can
be regenerated.

## 6.2 Silver --- Cleaned and Conformed

Responsibilities:

-   Schema validation
-   Type conversion
-   Timezone normalization to `Asia/Manila`
-   Deduplication
-   Null handling
-   Standardized relational representation

### Core DE concept: Data contracts

An HTTP 200 response does not guarantee valid analytical data. Data must
satisfy explicit expectations before being trusted downstream.

## 6.3 Gold --- Analytics / Business Layer

Responsibilities:

-   Heat index
-   Productivity Score
-   Workout Score
-   Alerts
-   Recommended work windows
-   Historical aggregates
-   Percentile context
-   Dashboard-ready tables

Gold is modeled using a **star schema** optimized for read-heavy
analytical queries. Build isolated candidates; publish data and metadata in one
transaction only after all critical checks and all configured cities pass.
Historical repair cannot replace the current forecast with an older snapshot.

## 6.4 Presentation Layer

Streamlit reads **Gold only**.

The dashboard must not depend directly on Bronze payload shapes or
perform core transformation logic itself.

## 6.5 Monitoring Layer

Airflow and pipeline metrics feed Prometheus/Grafana for operational
visibility.

------------------------------------------------------------------------

# 7. ⭐ Core Data Engineering Concepts

## 7.1 Batch Ingestion

Scheduled REST API extraction of weather observations and forecasts.

Key concerns:

-   Network/API failures
-   Retries
-   Source contracts
-   Raw preservation
-   Multi-city parameterization
-   Multi-source support

## 7.2 ETL / ELT

SILID primarily follows an ELT-oriented pattern:

``` text
Extract
  ↓
Load raw data
  ↓
Transform into trusted/analytical layers
```

Raw data is preserved before downstream transformation.

## 7.3 Idempotency

Pipeline reruns must not create logical duplicates.

Mechanisms may include:

-   Deterministic keys
-   Unique constraints
-   UPSERT/merge behavior
-   Deduplication
-   dbt incremental unique keys

## 7.4 Incremental Processing

Bronze is append-only. Silver and Gold should process newly arrived data
rather than fully rebuilding history every hour.

## 7.5 Partitioning

Bronze should be partitioned by ingestion date.

Example:

``` text
bronze/
  year=2026/
    month=09/
      day=27/
```

## 7.6 Dimensional Modeling

Gold uses facts and dimensions.

The most important modeling question is:

> **What does one row represent?**

That definition is the table's **grain**.

## 7.7 Data Quality

Data quality is a first-class pipeline stage, not an afterthought.

Checks include:

-   Schema
-   Nulls
-   Ranges
-   Uniqueness
-   Duplicates
-   Timestamps
-   Anomalies

## 7.8 Orchestration

Airflow coordinates:

-   Schedules
-   Tasks
-   Dependencies
-   Retries
-   Failure callbacks
-   Sensors

## 7.9 Fault Tolerance

A failed ingestion must not automatically take the served product
offline.

Desired behavior:

``` text
new ingestion fails
      ↓
do not publish invalid Gold
      ↓
serve last known-good Gold
      ↓
display stale-data indicator
```

## 7.10 Observability

The platform should expose enough information to answer:

-   Did the pipeline run?
-   Did it succeed?
-   How many rows arrived?
-   Is the data fresh?
-   Did quality checks pass?
-   How long did processing take?
-   What failed?

------------------------------------------------------------------------

# 8. Data Sources

## MVP Primary Source

**Open-Meteo**

Its current conditions are model-derived. The retained observation fact name
means captured current-condition samples, not station measurements. The standard
API does not document provider issue time; keep it null with an availability
status, and identify forecast history by retrieval snapshot.

Purpose:

-   Current weather
-   Hourly forecast data
-   Zero-cost primary ingestion source

## Secondary Source Candidates

-   OpenWeatherMap
-   WeatherAPI.com

Purpose:

-   Multi-source ingestion
-   Fallback/failover patterns

## Stretch Source

**PAGASA**

Potential use:

-   Philippine-specific advisories
-   Heat-index advisories
-   Tropical cyclone bulletins

This remains stretch scope because the project specification does not
assume a stable clean public REST API.

------------------------------------------------------------------------

# 9. Data Model

## Dimensions

### `dim_city`

**Grain:** one row per city.

``` text
city_id PK
city_name
region
latitude
longitude
timezone
```

### `dim_date`

**Grain:** one row per calendar date.

``` text
date_id PK
full_date
day_of_week
is_weekend
month
season_ph
```

### `dim_hour`

**Grain:** one row per hour of day.

``` text
hour_id PK
hour_24
is_typical_work_hour
is_typical_sleep_hour
```

### `dim_score_profile`

**Grain:** one row per scoring profile.

``` text
profile_id PK
profile_name
weight_config_version
```

## Facts

### `fact_weather_observation`

**Grain:** one city × one observed hour. Select the latest valid current-condition
sample within the Manila hour, retaining source time and artifact lineage. Missing
observation hours remain missing; forecasts do not fill them.

### `fact_weather_forecast`

**Grain:** one city × one forecasted hour × one retrieved forecast snapshot.

Must retain:

``` text
forecast_snapshot_id
fetched_at
forecast_issued_at (nullable provider-supplied time)
issue_time_status (supplied / unavailable)
```

Previous forecasts are not overwritten.

### `fact_productivity_score`

**Grain:** one city × one hour × one score profile in the current published
projection. Additional immutable score history includes forecast snapshot and
formula version. Historical trends select published scores available by the
target hour, avoiding later-forecast leakage.

### `fact_alert`

**Grain:** one triggered evaluation per city × target hour × alert type ×
forecast snapshot × threshold version. Current alerts use the published snapshot.

`dim_score_profile` references a current version while immutable prior weight
configurations remain available. `season_ph` defaults to `unknown` until a sourced
classification is adopted. Percentile populations and minimum sample sizes are
defined in DATA_CONTRACTS.md.

## Slowly Changing Dimensions

`dim_city` uses **Type 1** behavior for the MVP.

A future Type 2 implementation may add:

``` text
valid_from
valid_to
is_current
```

if historically changing city metadata becomes relevant.

------------------------------------------------------------------------

# 10. ⭐ Technology Stack

  -----------------------------------------------------------------------
  Concern                 Technology              Purpose
  ----------------------- ----------------------- -----------------------
  Programming             **Python**              API ingestion, pipeline
                                                  utilities, tests

  Database                **PostgreSQL**          relational Silver/Gold
                                                  storage

  Orchestration           **Apache Airflow**      scheduling, DAGs,
                                                  retries, dependencies

  Containerization        **Docker + Docker       reproducible local
                          Compose**               infrastructure

  Transformation          **SQL + dbt**           tested/versioned
                                                  analytical
                                                  transformations

  Object Storage          **MinIO /               immutable Bronze
                          S3-compatible**         storage

  Data Validation         **Great Expectations**  declarative
                                                  data-quality gates

  Dashboard               **Streamlit**           data-product
                                                  presentation

  Monitoring              **Prometheus +          metrics and
                          Grafana**               observability

  Testing                 **pytest + dbt tests**  code and data
                                                  correctness

  Version Control         **Git + GitHub**        source control and CI

  Project Tracking        **Jira / GitHub         milestones and backlog
                          Projects**              

  Cloud --- future        **AWS**                 S3, RDS/Aurora,
                                                  MWAA/ECS

  Infrastructure ---      **Terraform**           infrastructure as code
  future                                          

  Streaming --- stretch   **Apache Kafka**        event-driven ingestion
  -----------------------------------------------------------------------

### Scope rule

Technologies should be included because they satisfy an engineering
requirement. Kafka, Terraform, cloud migration, ML, and personalization
are not required for the DE-first MVP.

------------------------------------------------------------------------

# 11. Pipeline KPIs

  Metric                       Target
  ---------------------------- -----------------------------------------------
  Ingestion success            ≥ 99% scheduled runs succeed or retry-succeed
  Bronze freshness             ≤ 65 minutes old for hourly DAG
  Bronze → Gold latency        \< 10 minutes
  Data-quality pass rate       ≥ 98% per batch
  Silent schema drift          0 incidents
  Dashboard default load       ≤ 2 seconds
  Historical hourly coverage   ≥ 95% over trailing 30 days

------------------------------------------------------------------------

KPI measurement follows [DATA_CONTRACTS.md](DATA_CONTRACTS.md): all critical checks
pass even when aggregate pass rate exceeds 98%; expected-run denominators include
missing runs; forecasts do not count toward observation completeness; replay does
not reset freshness. Before 30 days of operation, report provisional evidence.

# 12. Portfolio Goal

The project should demonstrate an end-to-end data lifecycle:

``` text
source
  ↓
ingestion
  ↓
raw storage
  ↓
validation
  ↓
cleaning
  ↓
modeling
  ↓
business logic
  ↓
quality gate
  ↓
serving
  ↓
monitoring
```

The key portfolio claim is not merely familiarity with individual tools.
It is the ability to design, implement, explain, test, operate, and
defend the complete system.
