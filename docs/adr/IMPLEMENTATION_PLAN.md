# SILID --- Technical Implementation Plan

> Data Engineering Implementation Roadmap\
> Version 1.1 — Step 0 execution reconciliation

The [delivery sequence](REQUIREMENT_TRACEABILITY.md) Steps 0–12 defines the
execution order and mirrors the local detailed learning roadmap. The original phase numbers below remain reference topic numbers,
not an instruction to postpone testing, run tracking, or idempotency.
[Data contracts](DATA_CONTRACTS.md) govern behavior;
[requirement traceability](REQUIREMENT_TRACEABILITY.md) defines delivery gates.
Step 0 is documentation only; runtime guarantees remain unimplemented.

# 1. Implementation Objective

Build SILID as an end-to-end, locally reproducible Data Engineering
platform with a reliable path:

``` text
Weather API
    ↓
Bronze
    ↓
Silver
    ↓
Gold
    ↓
Streamlit
```

The implementation should prioritize **correct architecture and
operational reliability before stretch features**.

The guiding engineering principle is:

> **Fail loud, fail early, degrade gracefully.**

------------------------------------------------------------------------

# 2. ⭐ Core Stack

``` text
Python
PostgreSQL
Apache Airflow
Docker + Docker Compose
SQL + dbt
MinIO
Great Expectations
Streamlit
Prometheus
Grafana
pytest
Git + GitHub
```

Stretch/future:

``` text
AWS
Terraform
Kafka
secondary weather sources
PAGASA
ML
```

------------------------------------------------------------------------

# 3. Target Repository Structure

``` text
silid-weather-intelligence/
├── README.md
├── ARCHITECTURE.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── .github/
│   └── workflows/
├── airflow/
│   ├── dags/
│   ├── plugins/
│   └── config/
├── ingestion/
│   ├── sources/
│   │   └── open_meteo.py
│   ├── bronze_writer.py
│   └── tests/
├── transform/
│   └── dbt_project/
│       ├── models/
│       │   ├── silver/
│       │   └── gold/
│       ├── tests/
│       └── dbt_project.yml
├── quality/
│   └── great_expectations/
├── dashboard/
│   └── streamlit_app/
├── monitoring/
│   ├── prometheus/
│   └── grafana/
├── infra/
├── docs/
│   ├── PROJECT_SPEC.md
│   ├── REQUIREMENTS.md
│   ├── IMPLEMENTATION_PLAN.md
│   ├── DATA_CONTRACTS.md
│   ├── REQUIREMENT_TRACEABILITY.md
│   ├── architecture-diagram.png
│   ├── adr/
│   └── screenshots/
└── tests/
```

Organize by **pipeline stage / engineering concern**, not merely by file
type.

------------------------------------------------------------------------

# 4. Implementation Principles

## 4.1 Configuration over duplication

Cities, source URLs, thresholds, and score weights should be centrally
configurable.

Avoid separate hardcoded DAGs for each city.

## 4.2 Preserve raw data

Bronze is immutable and replayable.

Do not transform away source information before it is durably stored.

## 4.3 Define grain before tables

Every fact/dimension model should explicitly document what one row
represents.

## 4.4 Make reruns safe

Every stage should be designed with idempotency in mind.

## 4.5 Separate orchestration from business logic

Airflow should coordinate code, not contain large amounts of
transformation logic.

## 4.6 Keep analytical logic in dbt/SQL where appropriate

Silver/Gold transformations, business metrics, tests, and documentation
should be expressed through dbt when practical.

## 4.7 Test both code and data

``` text
pytest   → code correctness
dbt/GE   → data correctness
```

------------------------------------------------------------------------

# 5. Phase 0 --- Repository and Engineering Baseline

## Goals

Establish a version-controlled development foundation before
implementing pipeline logic.

## Tasks

-   [ ] Initialize Git repository
-   [ ] Create repository directory structure
-   [ ] Add `PROJECT_SPEC.md`
-   [ ] Add `REQUIREMENTS.md`
-   [ ] Add `IMPLEMENTATION_PLAN.md`
-   [ ] Add `.gitignore`
-   [ ] Add `.env.example`
-   [ ] Establish Python project/package configuration
-   [ ] Configure formatting/linting
-   [ ] Configure pytest
-   [ ] Add initial GitHub CI workflow
-   [ ] Create `docs/adr/`

## Definition of Done

``` text
git clone
→ environment setup documented
→ basic tests/lint can run
```

## Learning focus

-   Repository architecture
-   Python packaging
-   CI
-   Environment configuration
-   ADRs
-   Git workflow

------------------------------------------------------------------------

# 6. Phase 1 --- Local Infrastructure

## Goals

Stand up the minimum platform infrastructure with Docker Compose.

## Services

-   PostgreSQL
-   MinIO
-   Airflow

Monitoring services can be introduced later to reduce initial
complexity.

## Tasks

-   [ ] Create `docker-compose.yml`
-   [ ] Configure persistent volumes
-   [ ] Configure service networking
-   [ ] Configure environment variables
-   [ ] Create PostgreSQL databases/schemas
-   [ ] Create MinIO bucket(s)
-   [ ] Start Airflow metadata database/services
-   [ ] Verify service health
-   [ ] Document local startup/shutdown

## Definition of Done

``` bash
docker compose up
```

starts the required core services and they can communicate through the
Compose network.

## Core DE learning

-   Containerized data infrastructure
-   Service dependencies
-   Persistent storage
-   Networking
-   Environment/secrets management

------------------------------------------------------------------------

# 7. Phase 2 --- Open-Meteo Ingestion Connector

## Goals

Create a bounded Python ingestion component independent of Airflow.

## Suggested interface

``` python
def fetch_weather(city_config, request_time):
    ...
```

The exact implementation may differ, but the connector should have clear
inputs/outputs.

## Tasks

-   [ ] Define city configuration structure
-   [ ] Define requested Open-Meteo variables
-   [ ] Implement HTTP client
-   [ ] Configure timeout behavior
-   [ ] Handle non-200 responses
-   [ ] Validate that a response is parseable
-   [ ] Attach ingestion metadata
-   [ ] Add structured logging
-   [ ] Unit-test success/failure behavior

## Failure cases to test deliberately

-   [ ] Timeout
-   [ ] HTTP 500
-   [ ] Malformed JSON
-   [ ] Missing expected top-level field
-   [ ] Empty response
-   [ ] Invalid city configuration

## Definition of Done

The connector can fetch weather data for an initial set of approximately
3--5 configured Philippine cities without Airflow.

## Core DE learning

-   REST ingestion
-   Defensive source integration
-   Contracts
-   Error handling
-   Unit testing

------------------------------------------------------------------------

# 8. Phase 3 --- Bronze Storage

## Goals

Persist raw payloads before analytical transformation.

## Target

``` text
Open-Meteo
    ↓
Python connector
    ↓
MinIO Bronze
```

Optional convenience mirror:

``` text
Bronze raw PostgreSQL table
```

## Object key strategy

Use deterministic, partition-friendly paths.

Example:

``` text
bronze/open_meteo/
year=2026/
month=09/
day=27/
city=miagao/
<run-or-ingestion-id>.json
```

## Metadata to retain

At minimum consider:

``` text
source
city
ingestion_timestamp
run_id
request metadata
raw payload
```

## Tasks

-   [ ] Implement `bronze_writer.py`
-   [ ] Create bucket/path conventions
-   [ ] Preserve raw JSON
-   [ ] Add ingestion metadata
-   [ ] Design duplicate/rerun behavior
-   [ ] Implement dead-letter location/table
-   [ ] Test repeat execution
-   [ ] Test malformed-response persistence

## Definition of Done

A manual ingestion run creates traceable raw Bronze objects without
modifying prior valid history.

## Core DE learning

-   Raw immutability
-   Object storage
-   Partitioning
-   Replayability
-   Audit trails
-   Idempotency

------------------------------------------------------------------------

# 9. Phase 4 --- First Airflow DAG

## Goals

Move from manual execution to scheduled orchestration.

## DAG

``` text
weather_pipeline_hourly
```

Initial version:

``` text
extract_and_land_bronze → artifact references
```

Capture and land the response before parsing. Do not send raw payloads through
Airflow task metadata. Reuse durable accepted artifacts on retries.

Then evolve toward:

``` text
extract
  ↓
validate_bronze
  ↓
transform_to_silver
  ↓
dbt_run_gold
  ↓
quality_gate_gold
  ↓
publish_gold
  ↓
publish_metrics
```

## Tasks

-   [ ] Create DAG
-   [ ] Parameterize configured cities
-   [ ] Set hourly schedule
-   [ ] Configure retries
-   [ ] Configure exponential backoff
-   [ ] Add task logging
-   [ ] Verify reruns
-   [ ] Test forced API failure
-   [ ] Confirm failure state is visible

## Definition of Done

Airflow automatically lands new raw weather payloads in Bronze every
hour.

## Core DE learning

-   DAGs
-   Task dependencies
-   Scheduling
-   Retries
-   Orchestration vs processing

------------------------------------------------------------------------

# 10. Phase 5 --- Bronze Validation and Dead-Letter Flow

## Goals

Prevent malformed source data from silently moving downstream.

## Validation targets

-   Expected structure
-   Required fields
-   Types
-   Value ranges
-   Timestamp sanity

## Tasks

-   [ ] Initialize Great Expectations
-   [ ] Define Bronze expectations
-   [ ] Integrate validation into DAG
-   [ ] Block promotion on critical failure
-   [ ] Route malformed artifacts to dead-letter representation
-   [ ] Record validation result/metrics
-   [ ] Test intentional bad payloads

## Definition of Done

Bad source data is visible and isolated rather than silently becoming
trusted analytical data.

## Core DE learning

-   Data contracts
-   Data quality gates
-   Quarantine/dead-letter patterns
-   Defensive pipelines

------------------------------------------------------------------------

# 11. Phase 6 --- Silver Layer

## Goals

Convert valid raw weather data into typed, conformed relational data.

## Transformations

-   Parse raw payload
-   Normalize field names where defined
-   Cast data types
-   Normalize timestamps/timezone
-   Deduplicate
-   Handle allowed nulls
-   Persist relational Silver records

## Important timezone

``` text
Asia/Manila
```

## Idempotency design

Define logical uniqueness explicitly.

Observation example:

``` text
(city_id, observation_timestamp, source_id)
```

Forecasts preserve `forecast_snapshot_id` and nullable provider-supplied
`forecast_issued_at`; issue time alone is not a key. See DATA_CONTRACTS.md.

## Tasks

-   [ ] Define Silver schema
-   [ ] Implement Bronze → Silver transformation
-   [ ] Add uniqueness constraints/logic
-   [ ] Add null/range checks
-   [ ] Add timestamp validation
-   [ ] Test duplicate rerun
-   [ ] Test partially invalid batch

## Definition of Done

Silver contains clean, typed, deduplicated records safe for analytical
modeling.

## Core DE learning

-   Conformed data
-   Relational modeling
-   Data types
-   Deduplication
-   Idempotency
-   Temporal data

------------------------------------------------------------------------

# 12. Phase 7 --- dbt Project

## Goals

Move analytical transformation logic into a version-controlled dbt
project.

## Tasks

-   [ ] Initialize dbt project
-   [ ] Configure PostgreSQL target
-   [ ] Register Silver sources
-   [ ] Add source freshness where useful
-   [ ] Build staging/intermediate models if needed
-   [ ] Add schema tests
-   [ ] Add documentation
-   [ ] Generate dbt docs

## Core dbt concepts to learn

``` text
sources
ref()
models
materializations
tests
documentation
incremental models
is_incremental()
unique_key
```

## Definition of Done

dbt can build and test the analytical model from trusted Silver inputs.

------------------------------------------------------------------------

# 13. Phase 8 --- Gold Star Schema

## Goals

Implement the analytics-serving model.

## Dimensions

-   [ ] `dim_city`
-   [ ] `dim_date`
-   [ ] `dim_hour`
-   [ ] `dim_score_profile`

## Facts

-   [ ] `fact_weather_observation`
-   [ ] `fact_weather_forecast`
-   [ ] `fact_productivity_score`
-   [ ] `fact_alert`

## Required modeling discipline

For every table document:

``` text
grain
primary/logical key
foreign keys
source
refresh strategy
tests
```

## Type 1 SCD

Use Type 1 for `dim_city` in MVP.

Document Type 2 as a future option rather than implementing unnecessary
complexity.

## Definition of Done

Dashboard-oriented analytical queries can be answered through simple
predictable Gold joins.

## Core DE learning

-   Dimensional modeling
-   Star schema
-   Grain
-   Facts/dimensions
-   SCD
-   Analytical access patterns

------------------------------------------------------------------------

# 14. Phase 9 --- Business Metrics

## Goals

Implement the actual SILID value proposition in governed
transformations.

## Heat Index

-   [ ] Implement Rothfusz/NOAA calculation
-   [ ] Unit-test known inputs/outputs
-   [ ] Document assumptions

## Productivity Score

Inputs:

-   Heat index
-   Humidity
-   Precipitation risk
-   Time of day

Requirements:

-   [ ] 0--100 output
-   [ ] Centralized weights/configuration
-   [ ] Version identifier
-   [ ] Documented formula

## Workout Score

-   [ ] Separate score profile
-   [ ] Different weighting
-   [ ] Same versioning discipline

## Recommended Windows

-   [ ] Rank current-day hours
-   [ ] Materialize dashboard-ready result
-   [ ] Preserve underlying hourly score data

## Alerts

-   [ ] Heat threshold logic
-   [ ] Rain threshold logic
-   [ ] Severity
-   [ ] Threshold breached

## Definition of Done

Gold directly answers the project's user questions without requiring
Streamlit to contain business logic.

------------------------------------------------------------------------

# 15. Phase 10 --- Incremental Processing

## Goals

Avoid full historical rebuilds for every hourly run.

## Bronze

``` text
append-only
```

## Silver / Gold

Use incremental strategies where appropriate.

Select pending artifacts by artifact ID, stage, and transformation version.
Commit outputs before checkpoint completion and make repeated writes idempotent.
A maximum event/ingestion timestamp alone can skip late or replayed input; it is
not the processing contract. Introduce this with Silver in detailed Step 5, then
verify full-rebuild equivalence in Step 9.

## Tasks

-   [ ] Identify incremental models
-   [ ] Define unique keys
-   [ ] Define watermark strategy
-   [ ] Test reruns
-   [ ] Test late-arriving data
-   [ ] Test targeted reprocessing
-   [ ] Document full-refresh procedure

## Definition of Done

A normal hourly run processes only the required new/changed data while
remaining safe to rerun.

## Core DE learning

-   Incremental loading
-   Watermarks
-   Merge semantics
-   Late-arriving data
-   Backfills

------------------------------------------------------------------------

# 16. Phase 11 --- Final Quality Gate

## Goals

Prevent invalid Gold data from becoming the current served snapshot. Candidates
are isolated from readers. Every critical check and every configured city must
pass before a transaction promotes affected rows and publication metadata.
Metrics finalization runs on failure paths too, without masking failure.

## Checks

-   [ ] Required Gold models exist
-   [ ] Keys are unique where required
-   [ ] Relationships are valid
-   [ ] Required metrics are non-null
-   [ ] Scores are within 0--100
-   [ ] Freshness is acceptable
-   [ ] Row counts are plausible

## Failure behavior

``` text
quality gate fails
       ↓
stop publish
       ↓
retain last known-good Gold
       ↓
dashboard marks data stale
```

## Definition of Done

The serving layer never knowingly promotes a failed Gold validation run.

------------------------------------------------------------------------

# 17. Phase 12 --- Daily Rollup

## DAG

``` text
weather_pipeline_daily_rollup
```

## Schedule

``` text
00:15 Asia/Manila
```

## Responsibilities

-   Daily aggregates
-   7/30-day trends
-   Percentile context

## Dependency

Gate rollup on all expected prior-day city/hour coverage and required hourly
processing/publication. Wait at most 60 minutes, then record incomplete-period
failure and retain the previous rollups. Repair later through explicit replay.

## Definition of Done

Historical trend models are computed from complete/validated source
periods.

------------------------------------------------------------------------

# 18. Phase 13 --- Streamlit Serving Layer

## Pages

### Today

-   Current conditions
-   Heat index
-   Hourly Productivity Score
-   Best deep-work windows
-   Workout windows
-   Freshness/pipeline-health indicator

### Trends

-   7-day trends
-   30-day trends
-   Heat index
-   Humidity
-   Productivity Score
-   Percentile context

### Alerts

-   Heat alerts
-   Rain alerts
-   Severity

### About / How It Works

-   Scoring methodology
-   Architecture
-   Data pipeline
-   Repository link

## Rules

-   [ ] Query Gold only
-   [ ] Do not duplicate transformation logic in Streamlit
-   [ ] Surface data freshness
-   [ ] Target ≤ 2 second default load

## Definition of Done

A user can understand the recommended work window within approximately
five seconds of opening the primary view.

------------------------------------------------------------------------

# 19. Phase 14 --- Observability

## Metrics

Track at minimum:

``` text
ingestion_success
rows_in
rows_out
task_duration
pipeline_latency
data_freshness
quality_pass_rate
failed_runs
historical_completeness
```

## Tasks

-   [ ] Add structured JSON logs
-   [ ] Expose custom metrics
-   [ ] Configure Prometheus
-   [ ] Configure Grafana
-   [ ] Create pipeline-health dashboard
-   [ ] Add failure callback
-   [ ] Add freshness indicator
-   [ ] Test monitoring during forced failure

## Definition of Done

You can diagnose pipeline health without manually inspecting database
tables.

## Core DE learning

-   Logs
-   Metrics
-   Alerts
-   Freshness
-   Operational ownership

------------------------------------------------------------------------

# 20. Phase 15 --- Testing and CI

## Python tests

Use pytest for:

-   API connector behavior
-   Error handling
-   Bronze writer
-   Heat-index helper logic if implemented in Python
-   Configuration parsing
-   Utility functions

## dbt tests

Use for:

-   `unique`
-   `not_null`
-   `relationships`
-   `accepted_values`
-   custom business assertions

## Data-quality tests

Use Great Expectations for ingestion/batch-level validation.

## CI

On pull requests/pushes, run appropriate:

``` text
lint
format check
pytest
dbt parse/compile
dbt tests where test DB is available
```

## Definition of Done

Core pipeline behavior can be changed with automated regression
protection.

------------------------------------------------------------------------

# 21. Phase 16 --- Documentation and Portfolio Polish

## Required repository documentation

-   [ ] `README.md`
-   [ ] `ARCHITECTURE.md`
-   [ ] `PROJECT_SPEC.md`
-   [ ] `REQUIREMENTS.md`
-   [ ] `IMPLEMENTATION_PLAN.md`
-   [ ] ADRs
-   [ ] dbt docs
-   [ ] Architecture diagram
-   [ ] Star-schema diagram
-   [ ] Airflow DAG screenshot
-   [ ] Dashboard screenshots
-   [ ] 15--30 second demo GIF

## README structure

1.  Elevator pitch
2.  Architecture diagram
3.  Key features
4.  Tech stack with justifications
5.  Quickstart
6.  Dashboard/demo
7.  Data model
8.  Project story
9.  Lessons learned
10. Roadmap

## ADR candidates

``` text
ADR-001: PostgreSQL instead of a full lakehouse
ADR-002: MinIO for local S3-compatible Bronze
ADR-003: Medallion architecture
ADR-004: Type 1 SCD for dim_city
ADR-005: Batch-first instead of Kafka
ADR-006: Gold-only dashboard access
```

------------------------------------------------------------------------

# 22. Learning-Gated Milestones

These retain the original milestone themes, not four promised calendar weeks.
Advance only after validation and a learning review. Detailed Steps 0–12 are the
authoritative dependency order; CI, tests, run records, and rerun safety start
with the relevant components.

## Milestone A --- Foundations & Bronze

Deliver:

-   Docker Compose
-   PostgreSQL
-   MinIO
-   Airflow
-   Open-Meteo connector
-   Raw Bronze writer
-   First scheduled DAG
-   Repository baseline

**Core DE learned:** ingestion, orchestration basics, raw storage,
Docker.

## Milestone B --- Silver & Quality

Deliver:

-   Bronze validation
-   Silver schema
-   Cleaning/typing
-   Deduplication
-   Timezone normalization
-   Great Expectations
-   Dead-letter handling
-   Blocking quality gates

**Core DE learned:** data contracts, quality, idempotency, defensive
pipelines.

## Milestone C --- Gold & dbt

Deliver:

-   dbt project
-   Star schema
-   Heat index
-   Productivity Score
-   Workout Score
-   Alerts
-   Incremental models
-   dbt tests/docs

**Core DE learned:** SQL transformation, dimensional modeling,
incremental processing.

## Milestone D --- Serving & Operations

Deliver:

-   Streamlit
-   Daily rollups/trends
-   Prometheus
-   Grafana
-   CI hardening
-   README
-   Architecture diagrams
-   Demo assets

**Core DE learned:** serving, observability, freshness, operational
maturity.

------------------------------------------------------------------------

# 23. Codex-Assisted Development Protocol

Use Codex to accelerate implementation, but keep each task bounded
enough to understand fully.

For each implementation:

``` text
1. State the requirement ID.
2. State the DE concept being practiced.
3. Define inputs.
4. Define outputs.
5. Define invariants/guarantees.
6. Define expected failure cases.
7. Ask Codex to implement one bounded change.
8. Review the diff.
9. Run tests.
10. Intentionally break/test failure behavior.
11. Explain the implementation yourself.
12. Commit.
```

Before accepting Codex-generated code, answer:

``` text
Why does this component exist?
What data enters it?
What data leaves it?
What can fail?
What happens when it fails?
Can it be safely rerun?
How is correctness tested?
How is it observed?
Why is this design preferable here?
```

## Suggested commit style

``` text
feat(ingestion): add Open-Meteo weather connector
feat(bronze): persist immutable raw weather payloads
feat(airflow): schedule hourly Bronze ingestion
test(quality): validate humidity and required fields
feat(silver): normalize hourly weather records
feat(dbt): add Gold weather fact models
feat(metrics): add versioned productivity score
feat(observability): export pipeline freshness metrics
docs(adr): document batch-first ingestion decision
```

------------------------------------------------------------------------

# 24. Stretch Roadmap

Only after the batch MVP is stable:

1.  Second source + failover
2.  PAGASA enrichment
3.  Multi-city comparison
4.  Notifications
5.  AWS deployment
6.  Terraform
7.  Kafka streaming
8.  User profiles
9.  Personalized scoring
10. ML productivity prediction
11. AI-generated daily summaries

------------------------------------------------------------------------

# 25. Final Technical Definition of Done

The MVP is complete when this lifecycle works reliably:

``` text
external weather source
        ↓
scheduled ingestion
        ↓
immutable Bronze
        ↓
schema/data validation
        ↓
typed + deduplicated Silver
        ↓
incremental dbt transformations
        ↓
Gold star schema
        ↓
versioned productivity metrics
        ↓
blocking quality gate
        ↓
Streamlit serving
        ↓
Prometheus/Grafana monitoring
```

And when the engineer can explain **why every major architectural
decision exists**, not merely demonstrate that the containers run.
