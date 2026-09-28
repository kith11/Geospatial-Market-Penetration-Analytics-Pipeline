# SILID data contracts — v1

Status: Step 0 design baseline. These contracts describe required future behavior;
they do not claim that the prototype implements it. Read with
[requirements](REQUIREMENTS.md), [decisions](adr/001-source-history.md), and the
[delivery matrix](REQUIREMENT_TRACEABILITY.md).

## 1. Meaning before storage

A grain answers “what does one row represent?” A key identifies that row.
Event time describes the weather; processing time describes our handling of it.
Keeping these separate prevents duplicate facts and misleading historical joins.

MVP source: Open-Meteo; configured cities remain Manila, Iloilo, Cebu, Baguio,
and Davao. Adding a city changes configuration rather than duplicating a DAG.
Current conditions are model-derived, not station measurements. The retained
`observation` table name means a captured current-condition sample in this MVP.
Product labels must say model-derived current conditions.

Open-Meteo documents model-derived current conditions, a current-condition valid
time, and a processing-duration field named `generationtime_ms`. Its standard
forecast response does not document a provider issuance timestamp. Preserve
`forecast_issued_at = null` with `issue_time_status = unavailable` for this source;
never substitute retrieval time or processing duration. A later source may use
`issue_time_status = supplied` with its actual issue time.
[Source: Open-Meteo API documentation](https://open-meteo.com/en/docs).

Request current temperature, relative humidity, and wind; request 48 consecutive
hourly forecast slots including temperature, humidity, precipitation probability,
and wind. Use explicit Celsius, percent, km/h, and Asia/Manila settings. The
forecast starts at the source/request current-hour boundary, not calendar midnight.
Record the request and validate the returned horizon rather than relying on API
defaults. Extra returned fields stay in Bronze. Required forecast values may not
be null; precipitation probability in current conditions is not a required input.
Apparent temperature must not substitute for the derived NOAA heat index.

## 2. Identities and timestamps

| Name | Meaning and stability |
|---|---|
| `logical_run_id` | One scheduled hourly interval or explicit manual collection request. A task retry/clear reuses it; a deliberate new collection gets a new ID. |
| `attempt_id` | One actual HTTP attempt for a run/source/city. Allocate and record before requesting; distinct network attempts get distinct IDs. |
| `artifact_id` | One received body plus its immutable metadata, associated with an attempt. Rewriting/replaying this artifact retains its ID. |
| `forecast_snapshot_id` | The accepted response artifact's identity. It is a retrieval snapshot, not a claim about a provider model run. |
| `replay_id` | One explicit processing request over existing artifact IDs; does not create new source snapshots. |
| `publication_id` | One successful, coherent serving revision. Allocated for a candidate and made visible only by the promotion transaction. |
| `source_valid_at` / `target_hour` | When current conditions apply / which future hour a forecast describes. |
| `fetched_at` | UTC time the response was received. Never re-stamped during replay. |
| `landed_at` | UTC time body and metadata became durably available. |
| `forecast_issued_at` | Nullable provider-supplied issue time, never inferred from the other clocks. |
| `published_at` | UTC time a validated candidate became visible. |

Store instants as timezone-aware values (`TIMESTAMPTZ` in PostgreSQL). Derive
analytical dates and hours explicitly in Asia/Manila. Local source timestamps
without an offset must be interpreted using the verified response timezone;
reject contradictory timezone metadata. Do not shift UTC values and then label
them UTC again. Daily windows are half-open Manila intervals `[00:00, next 00:00)`.

Snapshot the city list, request/configuration version, and logical interval with
each run. Replays use that manifest, not today's mutable configuration. Business
formula and transformation versions are separate from source artifact identity.

## 3. Bronze durability, replay, and failure handling

MinIO is the authoritative raw archive. Store the HTTP response body exposed by
the client before JSON decoding, reformatting, or field extraction. Transport
framing/compression is not an analytical requirement; record content encoding and
whether the client decoded it. A sidecar holds status, selected response headers,
non-secret request parameters, identities, timestamps, and body SHA-256. Do not
embed metadata by rewriting the source JSON.

Use UTC landing-date/source/city prefixes with an immutable attempt/artifact
suffix. Dates organize retrieval; prefixes are not database partitions and the
checksum is an integrity check, not the logical key. Identical bodies collected
in two different hours remain distinct snapshots.

A run/source/city selects at most one successful artifact. A retry first checks
for that durable artifact and reuses it. If no successful artifact exists, it may
make another HTTP request under a new attempt. A forced refetch is a new logical
collection, not a hidden overwrite. Concurrent successful attempts are retained,
but a unique accepted-artifact mapping selects one for normal processing.

Write body and sidecar before marking the manifest landed. An interrupted write
is not eligible for processing. Reconcile an orphaned complete object/sidecar by
its original IDs and checksum rather than fetching again. If database registration
succeeds but the acknowledgement is lost, a unique key prevents a second logical
registration. This provides idempotent effects, not a cross-system exactly-once
transaction.

Non-200 or malformed bodies remain raw dead-letter artifacts with error context;
never force them into JSONB. Timeouts without a body create attempt/error records.
Invalid artifacts are retained with validation reasons and are not promoted.
Existing `bronze.weather_raw` JSONB data remains intact; any future import labels
it `legacy_parsed`, because original bytes cannot be recovered. A parsed staging
copy is disposable and never replaces the raw archive.

## 4. Silver and Gold grains

| Model | Logical grain / key | Selection and history rule |
|---|---|---|
| Silver current conditions | `(city_id, source_valid_at, source_id)` | First accepted valid artifact wins for repeated source timestamps; preserve all source responses in Bronze. Explicit correction requires versioned replay, not incidental overwrite. |
| Silver forecast | `(city_id, target_hour, source_id, forecast_snapshot_id)` | Preserve every accepted collection snapshot, even when two snapshots forecast the same hour. |
| `dim_city` | `city_id` | Type 1 descriptive updates; source-request coordinates remain in artifact metadata. |
| `dim_date` | Manila calendar date | `season_ph = unknown` until an explicitly sourced classification is adopted. |
| `dim_hour` | Integer local hour 0–23 | Business-hour labels are configuration, not physiological claims. |
| `dim_score_profile` | `profile_id` | Deep Work and Workout; points to a current configuration version. Immutable versioned configurations remain available separately. |
| `fact_weather_observation` | `(city_id, observation_hour)` | In each local hour, choose the latest valid Silver sample from the MVP source; retain exact sample time and artifact lineage. Never substitute a forecast into a missing observation hour. |
| `fact_weather_forecast` | `(city_id, target_hour, forecast_snapshot_id)` | Full retrieval history; nullable provider issue time plus availability status. |
| Score history | `(city_id, target_hour, profile_id, forecast_snapshot_id, formula_version)` | Forecast-based suitability scores, input/component lineage, immutable formula versions. |
| `fact_productivity_score` | `(city_id, target_hour, profile_id)` | Published current projection of score history; use one snapshot per city from the selected collection, never mix hourly rows from arbitrary snapshots. |
| `fact_alert` | `(city_id, target_hour, alert_type, forecast_snapshot_id, threshold_version)` | One triggered evaluation event; retries do not duplicate it. A new forecast may produce a new event. Current alerts select the published snapshot only. |

The MVP has one source; adding another requires explicit source-selection rules
before it can feed single-source Gold grains. Every fact retains source/artifact
lineage. Descriptive dimension changes do not mutate the raw request history.

Scores are forecast-based heuristics, not validated measures of human performance.
Never replace a missing probability with zero or renormalize weights silently.
Invalid required forecast inputs block the city batch. Numerical score curves,
weights, heat-index applicability details, and alert thresholds belong to Step 7
and must be documented and tested before the first score is produced.

Recommendations rank eligible hourly slots, not multi-hour windows. Eligible
hours start at or after publication time on the current Manila date (exclude a
partly elapsed hour). Sort by descending score then ascending target hour. Store
rank and eligibility in Gold; the dashboard may hide expired rows using their
validity times but must not recompute scores or ranking. A stale publication whose
hours have expired displays no upcoming recommendations rather than yesterday's
windows as today's advice.

## 5. Incremental processing and quality

Track processing by `(artifact_id, transformation_version, stage)`. Select pending
artifacts explicitly, including late arrivals. Do not use maximum weather time
as the sole checkpoint. Commit outputs before marking work complete; a crash in
between must be recoverable through deterministic keys. Full rebuild and targeted
replay must converge for the same inputs, code, and configuration versions.

Critical checks: parseable success response; required structures and units;
aligned hourly arrays; required types/values; humidity and precipitation
probability in 0–100; unique keys; timezone/timestamp consistency; 48 consecutive
forecast hours; valid relationships; score bounds; required-city completeness.
A city batch with a critical failure is quarantined as a unit. Other valid cities
may reach Silver, but cannot publish a partial Gold batch.

Unknown added fields are preserved and produce a visible schema-drift event.
Missing/renamed required fields or incompatible types block promotion. Nullable
optional fields such as UV index are allowed explicitly. Statistical jumps are
advisory alerts; their versioned thresholds are defined with business metrics.
All critical checks must pass. The 98% aggregate quality KPI is a separate
measurement and cannot grant permission to publish invalid data.

## 6. Publication and daily history

Gold candidates are invisible to the dashboard. An hourly candidate must include
every city in its run manifest with valid required forecast coverage. Build and
validate first, then promote affected data, serving views/tables, lineage, and
publication metadata in one PostgreSQL transaction. A failure rolls the transaction
back and leaves the prior publication visible.

Serialize publication. Live candidates older than the current logical collection
cannot replace it. Replays repair declared historical ranges, never advance live
freshness or replace the active forecast with an older snapshot. Revalidate a
candidate if its base publication changed. Daily candidates modify historical
serving outputs only and do not reset the hourly weather freshness clock. Use a
single serving revision to make multi-query dashboard reads consistent.

Before the first publication, show no-data status. Thereafter show the source
fetch time, publication time, latest-attempt outcome, and stale status. The latest
attempt failing is visible immediately; age greater than 65 minutes makes the
weather stale. The dashboard role reads only published Gold, including its status
projection. Bad batches never become visible merely because metrics succeeded.

At 00:15 Manila, daily processing waits for all expected prior-day city/hour
coverage and successful required hourly processing/publication. Wait at most
60 minutes; on timeout record an incomplete-period failure, retain prior rollups,
and repair later through replay. One successful latest hourly run is insufficient.
A 95% monthly coverage SLO is not permission to call an incomplete day complete.

Historical heat-index trends use canonical model-derived observations. Historical
score trends use the latest successfully published forecast score available at or
before each target hour; later forecasts must not rewrite what was knowable then.
Preserve publication-to-score lineage to support that selection. Exclude target
hours with no eligible publication and expose their coverage. Do not synthesize
retrospective productivity scores by joining observed heat to arbitrary rain risk.

Trailing windows contain the previous 7 or 30 complete Manila days. Percentile
context compares each target-hour value with the previous 30 complete days for
the same city, local hour, metric, profile (for scores), and formula version.
Weather context uses observed heat-index history and is labeled accordingly;
score context uses the historical published forecast-score population. Exclude
the target day. Require at least 7 valid daily samples; otherwise show insufficient
history. Empirical percentile is `100 * count(reference <= value) / count(reference)`.
Show sample size and period; a high heat percentile means hotter, not better.
Do not pool formula versions. Repairs recompute affected days and their following
30-day dependent windows. These are product defaults, not statistical claims.

## 7. Scheduling and measurement

Airflow owns extraction retries: 3 retries after the first attempt, native
exponential backoff with a 1-minute base and a 15-minute delay cap. Exact delays
are implementation-dependent; the former 1/5/15 sequence is not contractual.
Transient network errors, 429, and 5xx responses are retryable; malformed payloads,
invalid configuration, and other 4xx errors fail without blind retry. Persist error
bodies first. Do not stack a second HTTP retry loop. Transformation/dbt tasks keep
one retry. Validate the selected Airflow version's policy in Step 4.

Collection runs hourly with historical live-source catchup disabled. Backfills
replay saved artifacts; they do not pretend a present API request was made in the
past. Metrics finalization runs on success and failure without masking task/DAG
failure. Failure callbacks record structured context locally; external message
integrations remain outside MVP.

| Measure | Definition / target |
|---|---|
| Ingestion success | Over trailing 30 days, scheduled logical runs landing valid success responses for all expected cities within their retry budget / expected runs. Count missing runs as failures. Report ingestion validation and Gold publication success separately. Manual/replay runs excluded. |
| Operating window | Starts at declared collection activation. Expected slots use configuration effective dates; outages and maintenance after activation remain in denominators. Before 30 days, report actual duration as provisional. |
| Bronze freshness | Per city, now minus latest valid response `fetched_at` durably landed; <=65 minutes under normal operation. Report age of any raw landing separately so a fresh error body cannot look healthy. |
| Gold freshness | Per city, now minus selected published source `fetched_at`; stale when >65 minutes. Replay/publication time alone cannot reset it. |
| Pipeline latency | Hourly `published_at` minus earliest `landed_at` among the selected successful city artifacts; <10 minutes. Report end-to-end scheduled-run latency separately, including retries. |
| Quality pass rate | Per city batch and declared suite version, passed executed GE expectations / total executed expectations; >=98%. Skipped required checks block; zero executed checks is unavailable, never 100%. Emit critical and advisory results separately. |
| Silent schema drift | Every detected contract change emits a durable event. Demonstrate injected additive and breaking changes; do not infer zero drift merely from absent logs. |
| Historical completeness | Valid canonical observation city/hour slots / expected active city/hour slots over trailing 30 days; >=95%. Forecast rows never fill missing observations. Report per-city and overall results. |
| Dashboard latency | Default Today view on declared hardware and data size, time until data rendered; target <=2 seconds. Report cold and warm-cache results separately. |

## 8. Learning walkthrough and acceptance examples

Predict each result before reading the expected outcome. These are design examples,
not executed runtime tests; implementation steps must turn them into regression checks.

| Scenario | Expected outcome / invariant |
|---|---|
| Response lands, worker crashes before manifest registration | Reconcile original artifact and checksum. Retry does not create another selected logical input. |
| Two hourly collections forecast Manila at 15:00 | Both snapshot IDs survive; one target timestamp is not a sufficient forecast key. |
| Formula v2 recomputes the same snapshot | New score-history keys retain v1. Current serving switches only through a validated publication. |
| Cebu fails while four other cities validate | Four cities may reach Silver; previous complete Gold remains active and failed-attempt status is visible. |
| Gold promotion fails halfway through | Transaction rolls back data and metadata together; readers retain the previous revision. |
| New optional field appears | Preserve field, record drift; allow promotion if critical contracts still pass. |
| Humidity is 140 despite 99% aggregate checks passing | Critical failure blocks publication. |
| A late old artifact is replayed | Process by artifact/version state, repair declared history, keep live forecast/freshness unchanged. |
| No prior publication exists | Display no-data status; do not fabricate a last known-good dataset. |
| Observation valid at 23:45 Manila arrives at 00:05 | Group by source-valid date/hour, not landing date; recompute the affected prior day. |

Before completing the learning review, explain: why target hour alone cannot key a
forecast; why retrieval is not issuance; why a database JSON copy is not the raw
archive; why 98% quality can still fail; and why build/test/publish must be separate.
