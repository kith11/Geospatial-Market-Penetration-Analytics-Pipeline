# SILID --- Software Requirements Specification

> Functional and Non-Functional Requirements\
> Version 1.1 — Step 0 contract reconciliation

# 1. Purpose

This document defines the software requirements for the SILID Weather
Intelligence Platform.

Requirement identifiers remain stable in Git so implementation commits, issues,
tests, and pull requests can reference them directly.

The [data contracts](DATA_CONTRACTS.md) define source semantics, grains,
identities, quality/publication policy, and KPI measurement. The
[traceability matrix](REQUIREMENT_TRACEABILITY.md) maps every requirement to a
delivery step and acceptance check. These are target behaviors, not claims that
the current prototype implements them.

------------------------------------------------------------------------

# 2. Functional Requirements

## FR-001 --- Automated Weather Ingestion

**Requirement:** The system shall perform scheduled pulls of current
weather conditions and hourly forecast data for a configured set of
Philippine cities.

**Acceptance criteria:**

-   Ingestion is initiated by Airflow.
-   Cities are configuration-driven rather than hardcoded into separate
    DAGs.
-   Raw source payloads are persisted to Bronze.
-   Source and ingestion metadata are retained.

**Core DE:** batch ingestion, parameterization, orchestration.

------------------------------------------------------------------------

## FR-002 --- Hourly Forecast Storage

**Requirement:** The system shall ingest and persist at least 24--48
hours of hourly forecast data per configured city per ingestion run.
The MVP contract requests and validates 48 consecutive hourly slots.

Minimum target variables:

-   Temperature
-   Relative humidity
-   Precipitation probability
-   Wind

**Core DE:** time-series ingestion, historical persistence.

------------------------------------------------------------------------

## FR-003 --- Historical Weather Archive

**Requirement:** The system shall maintain an append-only historical
record of captured current conditions and forecast snapshots in Bronze.
For Open-Meteo, current conditions are model-derived, not station observations.
Derived canonical projections may change through versioned replay; source history
and forecast/score versions remain available.

The archive shall support:

-   Trend analysis
-   Percentile analysis
-   Reprocessing
-   Auditing

**Core DE:** immutable history, reproducibility, append-only storage.

------------------------------------------------------------------------

## FR-004 --- Heat Index Computation

**Requirement:** The system shall derive heat index from temperature and
relative humidity using the standard Rothfusz regression / NOAA formula.

The computation belongs in the transformation pipeline rather than being
blindly sourced from the weather API.

------------------------------------------------------------------------

## FR-005 --- Productivity Score Calculation

**Requirement:** The system shall compute a documented and versioned
hourly `0–100` Productivity Suitability Score.

Inputs include:

-   Heat index
-   Humidity
-   Precipitation risk
-   Time of day

The score shall retain a version identifier so future formula changes
remain traceable.

**Core DE:** governed business logic, reproducibility, metric
versioning.

------------------------------------------------------------------------

## FR-006 --- Best Deep-Work Window Recommendation

**Requirement:** The system shall rank the current day's hourly slots by
Productivity Score and expose the highest-ranked windows through Gold
tables.

The dashboard shall consume the precomputed result rather than
independently recomputing the ranking. MVP windows are individual future hourly
slots; score descending and target hour ascending define deterministic order.
Expired recommendations are hidden without recomputing rankings.

------------------------------------------------------------------------

## FR-007 --- Workout Recommendation

**Requirement:** The system shall support a separate workout scoring
profile with weights appropriate for physical activity.

The score profile shall be represented explicitly in the data model.

------------------------------------------------------------------------

## FR-008 --- Rain / Heat Alerts

**Requirement:** The system shall generate alert records when forecasted
heat index or precipitation probability crosses configured thresholds.

Alert records shall contain sufficient context to identify:

-   City
-   Relevant hour/date
-   Alert type
-   Severity
-   Threshold breached

------------------------------------------------------------------------

## FR-009 --- Dashboard Filtering

**Requirement:** The dashboard shall support:

-   City selection
-   Date-range selection
-   Score profile selection: Deep Work / Workout

------------------------------------------------------------------------

## FR-010 --- Historical Trend Analysis

**Requirement:** The dashboard shall display historical heat-index and
productivity-score trends for trailing periods such as 7 and 30 days.

The system shall provide percentile context such as "today vs. typical."
Weather history uses model-derived observations; score history uses published
forecast scores available before their target hour. Reference populations, minimum
sample counts, formula separation, and missing-history behavior follow DATA_CONTRACTS.md.

------------------------------------------------------------------------

## FR-011 --- Bronze Raw Preservation

**Requirement:** Bronze shall preserve source payloads as received
without silently dropping, renaming, or reshaping source fields.

**Core DE:** raw immutability and replayability.

------------------------------------------------------------------------

## FR-012 --- Silver Conformance

**Requirement:** The system shall transform valid Bronze data into
typed, deduplicated, timezone-normalized Silver records.

All analytical timestamps shall be normalized consistently to
`Asia/Manila` where specified by the design.

------------------------------------------------------------------------

## FR-013 --- Gold Analytical Models

**Requirement:** The system shall expose analytics-ready Gold models
containing the project's business metrics and dashboard-serving
structures.

Gold shall use dimensional/star-schema modeling where specified.

------------------------------------------------------------------------

## FR-014 --- Forecast Version Preservation

**Requirement:** Forecast records shall preserve a stable `forecast_snapshot_id`,
`fetched_at`, and provider-supplied `forecast_issued_at` when available. If the
source does not supply issue time, retain null and `issue_time_status = unavailable`;
never substitute retrieval time. Previous snapshots for the same target hour
shall remain available. A snapshot identifies a retrieved response, not a provider
model run. See [ADR-001](adr/001-source-history.md).

**Core DE:** temporal modeling and forecast-history preservation.

------------------------------------------------------------------------

## FR-015 --- Data Quality Gates

**Requirement:** The pipeline shall execute blocking quality checks
before invalid data is promoted downstream. All critical checks must pass; the
aggregate quality-pass KPI cannot waive a critical failure. Quarantine an invalid
city batch; preserve its raw artifacts and validation reasons.

Checks include:

-   Required schema
-   Required fields
-   Valid types
-   Valid ranges
-   Duplicate detection
-   Timestamp sanity

------------------------------------------------------------------------

## FR-016 --- Dead-Letter Handling

**Requirement:** Malformed JSON, non-200 source responses, and other
invalid ingestion artifacts shall be caught and logged rather than
silently discarded.

Where applicable, invalid records/responses shall be routed to a Bronze
dead-letter representation.

------------------------------------------------------------------------

## FR-017 --- Incremental Transformation

**Requirement:** Silver and Gold transformations shall process newly
arrived data incrementally where appropriate rather than fully
rebuilding all history for every hourly run.

**Core DE:** incremental loading and efficient recomputation.

------------------------------------------------------------------------

## FR-018 --- Idempotent Reprocessing

**Requirement:** Re-running an ingestion or transformation for the same
logical input shall not create unintended logical duplicates.

**Core DE:** idempotency, deterministic processing.

------------------------------------------------------------------------

## FR-019 --- Pipeline Metrics

**Requirement:** Each pipeline run shall expose operational metrics
including, where applicable:

-   Rows in
-   Rows out
-   Duration
-   Source
-   Run/task status
-   Freshness
-   Quality status

------------------------------------------------------------------------

## FR-020 --- Last Known-Good Serving

**Requirement:** When a new ingestion/transformation fails quality
validation, the system shall continue serving the most recent valid Gold
dataset.

The dashboard shall expose a visible stale-data/freshness indicator and the latest
attempt outcome. Build isolated Gold candidates, validate every city in the run
manifest, and publish data plus metadata atomically. Before the first successful
publication, show no-data status. Historical replay cannot reset live freshness.
See [ADR-002](adr/002-quality-and-publication.md).

------------------------------------------------------------------------

# 3. Non-Functional Requirements

## NFR-001 --- Scalability / Configurability

Adding a new city or secondary source should primarily require
configuration or connector implementation rather than duplication of
entire DAGs.

------------------------------------------------------------------------

## NFR-002 --- Maintainability

Transformation logic shall live in version-controlled dbt models where
applicable.

Business/scoring configuration shall be centralized rather than
duplicated across scripts.

------------------------------------------------------------------------

## NFR-003 --- Observability

Every DAG run shall provide structured logs and operational metrics
sufficient to diagnose success, failure, volume, duration, and
freshness.

Prometheus/Grafana shall visualize the selected pipeline metrics.

**Core DE:** observability, operational ownership.

------------------------------------------------------------------------

## NFR-004 --- Reliability

Transient API/network failures shall use automatic retries with
exponential backoff.

Target ingestion reliability:

``` text
≥ 99% scheduled runs succeed or retry-succeed
```

------------------------------------------------------------------------

## NFR-005 --- Idempotency

Pipeline retries and manual reruns shall not produce unintended
duplicate logical records.

This is a mandatory reliability property.

------------------------------------------------------------------------

## NFR-006 --- Fault Tolerance

A failed ingestion or quality gate shall not unnecessarily make the
dashboard unavailable.

The system shall degrade to last known-good Gold data where possible.

------------------------------------------------------------------------

## NFR-007 --- Data Freshness

For the hourly pipeline, Bronze data should be no more than
approximately **65 minutes old** under normal operation.

------------------------------------------------------------------------

## NFR-008 --- Pipeline Latency

Target Bronze-to-Gold processing latency:

``` text
< 10 minutes per run
```

------------------------------------------------------------------------

## NFR-009 --- Data Quality

Target validation pass rate:

``` text
≥ 98% of configured Great Expectations checks per batch
```

Report this separately from the requirement that 100% of critical checks pass.
Count executed expectations under a declared suite version; skipped required
checks block, and zero executed checks is unavailable, not a successful batch.

The platform targets **zero silent schema-drift incidents**.

------------------------------------------------------------------------

## NFR-010 --- Historical Completeness

Target:

``` text
≥ 95% hourly coverage over the trailing 30 days
```

------------------------------------------------------------------------

## NFR-011 --- Dashboard Performance

The default dashboard view should load in approximately:

``` text
≤ 2 seconds
```

The dashboard should query serving-ready Gold models rather than raw
layers.

------------------------------------------------------------------------

## NFR-012 --- Security

Secrets shall not be committed to Git.

Credentials/API keys shall use:

-   Environment variables and/or Docker secrets
-   Least-privilege database roles

------------------------------------------------------------------------

## NFR-013 --- Modularity

The repository shall maintain clear separation among:

``` text
ingestion
transformation
quality
serving
monitoring
infrastructure
```

Components should be independently testable where practical.

------------------------------------------------------------------------

## NFR-014 --- Code Quality

Python should be typed where practical.

The project shall use:

-   Linting
-   Formatting
-   Unit tests
-   CI

The source specification proposes tools such as:

``` text
ruff / flake8
black
pytest
```

------------------------------------------------------------------------

## NFR-015 --- Documentation

The repository shall document:

-   Architecture
-   Setup / quickstart
-   Pipelines
-   dbt models
-   Data model
-   Architectural decisions
-   Scoring methodology
-   Operational behavior

Recommended artifacts:

``` text
README.md
ARCHITECTURE.md
docs/adr/
dbt docs
architecture diagrams
```

------------------------------------------------------------------------

## NFR-016 --- Reproducibility

The core local platform shall be reproducible through Docker/Docker
Compose rather than relying on undocumented machine-specific setup.

------------------------------------------------------------------------

# 4. Data Quality Requirements

## DQ-001 --- Schema

Validate expected fields, types, and structure after Bronze extraction.

## DQ-002 --- Humidity Range

``` text
0 <= humidity_pct <= 100
```

## DQ-003 --- Required Fields

Required values include:

``` text
temperature
humidity
timestamp
```

## DQ-004 --- Duplicate Detection

Observation uniqueness should include the logical key:

``` text
(city_id, observation_timestamp, source_id)
```

## DQ-005 --- Timestamp Sanity

Records with unreasonable timestamps or invalid timezone formats shall
be rejected or quarantined.

## DQ-006 --- Optional Nulls

Non-critical fields such as UV index may be nullable when explicitly
modeled as optional.

## DQ-007 --- Anomaly Monitoring

Suspicious changes, such as a heat-index jump greater than the
configured threshold between consecutive hours for a city, should be
surfaced as monitoring alerts rather than necessarily hard-failing the
pipeline.

------------------------------------------------------------------------

# 5. Data Modeling Requirements

## DM-001 --- `dim_city`

Grain: one row per city.

## DM-002 --- `dim_date`

Grain: one row per calendar date.

## DM-003 --- `dim_hour`

Grain: one row per hour-of-day.

## DM-004 --- `dim_score_profile`

Grain: one row per scoring profile.

## DM-005 --- `fact_weather_observation`

Grain: one row per city per observed hour. In this MVP, select the latest valid
model-derived current-condition sample within each Manila hour; retain the exact
source time and lineage. A forecast cannot fill a missing observation hour.

## DM-006 --- `fact_weather_forecast`

Grain: one row per city per forecasted hour per retrieved forecast snapshot.
Preserve all snapshot versions and nullable provider issue time.

## DM-007 --- `fact_productivity_score`

Grain: one row per city per hour per score profile in the published current
projection. Keep separate score history at city × hour × profile × forecast
snapshot × formula version so recomputation does not erase earlier scores.

## DM-008 --- `fact_alert`

Grain: one row per triggered alert evaluation event, keyed by city, target hour,
alert type, forecast snapshot, and threshold version. Retry does not duplicate an
event; a new forecast may create another event. Current alerts use published snapshots.

------------------------------------------------------------------------

# 6. Orchestration Requirements

## ORCH-001 --- Hourly DAG

Primary DAG:

``` text
weather_pipeline_hourly
```

Task flow:

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

Metrics finalization must also run on failure paths without turning a failed DAG
into a successful one. `extract` includes durable Bronze landing; task outputs
carry artifact references. Candidates remain invisible until `publish_gold`.

## ORCH-002 --- Daily Rollup DAG

``` text
weather_pipeline_daily_rollup
```

Purpose:

-   Daily aggregates
-   Historical trend models
-   Percentile calculations

## ORCH-003 --- Schedule

``` text
hourly pipeline → every 60 minutes
daily rollup    → 00:15 Asia/Manila
```

## ORCH-004 --- Extraction Retry Policy

Target policy:

``` text
3 retries after the first attempt
native exponential backoff
base delay: 1 minute
maximum delay: 15 minutes
```

Actual waits follow the selected Airflow version; exact 1/5/15-minute delays are
not required. Retry transient network errors, 429, and 5xx; fail deterministic
configuration/data errors directly. Airflow owns retries; no second HTTP retry loop.
See [ADR-003](adr/003-learning-and-execution-policy.md).

## ORCH-005 --- Transformation Retry Policy

Transformation/dbt tasks use one retry because repeated failure is more
likely to indicate a deterministic data or logic problem.

## ORCH-006 --- Failure Notification

Final failures should invoke a structured failure callback containing
useful run/task context.

## ORCH-007 --- Daily Dependency

The daily rollup shall wait for the required previous hourly processing
to complete successfully and check every expected previous-day city/hour.
Wait at most 60 minutes after the daily task starts; record incomplete-period
failure on timeout and retain prior rollups. Later replay repairs missing periods.
The 95% monthly coverage target does not make an incomplete daily batch complete.

------------------------------------------------------------------------

# 7. MVP Scope Boundary

The following are **not required for the initial DE-first MVP**:

-   Kafka streaming
-   ML productivity prediction
-   Personalized user profiles
-   Push/SMS notifications
-   Mobile application
-   Terraform cloud infrastructure
-   Full AWS deployment
-   PAGASA integration
-   AI-generated advice

They remain future/stretch work after the core batch platform is
reliable.
