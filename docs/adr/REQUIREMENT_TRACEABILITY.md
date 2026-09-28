# SILID requirement traceability and delivery gates

## Status and ownership

Step numbers follow the delivery sequence below, mirrored from the local
`DETAILED_IMPLEMENTATION_PLAN.md`, not the older topic-phase numbers. The local
roadmap is currently Git-ignored; this tracked sequence makes the project
understandable in a clean clone without requiring that personal reference. Owners below are engineering responsibilities,
not additional people or agents. Every row is a **planned runtime acceptance gate**;
none is marked passed merely because documentation exists. NFR-015 has Step 0
artifacts delivered but remains open through final operating documentation.

Step 0 deliverables: [data contracts](DATA_CONTRACTS.md),
[ADR-001](adr/001-source-history.md), [ADR-002](adr/002-quality-and-publication.md),
[ADR-003](adr/003-learning-and-execution-policy.md), and reconciled requirements,
specification, and plans. The user learning review follows document validation.
Step 1 and all later runtime work remain unstarted.

## Delivery sequence

Only Step 0 documentation is delivered. Request each later step separately.

| Step | Bounded implementation unit | Depends on |
|---|---|---|
| 0 | Reconcile source, temporal, identity, quality and publication contracts | Repository review |
| 1 | Packaging, validated settings, linting, unit tests and CI | 0 |
| 2 | PostgreSQL roles/migrations, persistent MinIO and deterministic Airflow setup | 1 |
| 3 | Reliable source connector, immutable Bronze, manifests and interrupted-write recovery | 2 |
| 4 | Hourly city-driven ingestion DAG, retries, run records and failure context | 3 |
| 5 | Bronze quality gates, typed dbt Silver, incremental checkpoints and replay | 4 |
| 6 | Gold dimensions and observation/forecast facts with documented grain | 5 |
| 7 | NOAA heat index, versioned score profiles, rankings and alerts | 6 |
| 8 | Isolated Gold candidates, blocking gates and atomic publication | 7 |
| 9 | Daily dependencies, historical trends, percentiles and targeted repair | 8 |
| 10 | Gold-only dashboard with filters, consistency and visible freshness | 8–9 |
| 11 | Prometheus/Grafana, failure drills, recovery evidence and integration CI | 4–10 |
| 12 | Deterministic demonstration, clean-clone walkthrough and portfolio documentation | 11 |

## Requirement map

| Requirement | Delivery step(s) | Responsible concern | Acceptance evidence required |
|---|---|---|---|
| FR-001 | 3–4 | Ingestion / orchestration | Configured cities land current and forecast artifacts from the hourly DAG. |
| FR-002 | 3, 5 | Ingestion / quality | Each accepted city batch has 48 consecutive forecast slots with required variables. |
| FR-003 | 3, 5–7 | Storage / transformation | Earlier source and forecast versions survive new collection and replay. |
| FR-004 | 7 | Metrics | NOAA reference and applicability/unit-boundary examples pass. |
| FR-005 | 7 | Scoring | Scores are 0–100, explainable, and linked to immutable formula versions. |
| FR-006 | 7, 10 | Recommendations / serving | Gold ranks eligible current-day slots deterministically; UI does not rerank. |
| FR-007 | 6–7 | Modeling / scoring | Deep Work and Workout have separate versioned profile behavior. |
| FR-008 | 7 | Alerts | Heat/rain events retain city, hour, severity, threshold and deduplicated identity. |
| FR-009 | 10 | Dashboard | City, date range, and profile filters return the intended published Gold data. |
| FR-010 | 9–10 | Analytics / dashboard | Known fixtures verify 7/30-day trends, populations, insufficient-history state. |
| FR-011 | 3 | Storage | Stored body checksum matches captured body before JSON decoding; metadata is separate. |
| FR-012 | 5 | Silver | Types, timezone boundaries, null handling, and duplicate handling pass. |
| FR-013 | 6–8 | Gold | Documented fact/dimension joins answer analytical queries on validated publications. |
| FR-014 | 3, 5–6 | Temporal modeling | Same target hour across snapshots survives; missing issue time stays null. |
| FR-015 | 5, 8 | Quality | Any critical failure blocks the relevant promotion regardless of aggregate pass rate. |
| FR-016 | 3, 5 | Ingestion / quality | Non-200/malformed bodies persist; transport failures without bodies retain context. |
| FR-017 | 5–9 | Transformation | Pending-artifact processing and affected-window repair match a full rebuild. |
| FR-018 | 3–9 | Pipeline reliability | Repeated same-input processing produces no unintended logical duplicates. |
| FR-019 | 4–11 | Operations | Rows, duration, status, source, freshness and quality survive failure paths. |
| FR-020 | 8, 10–11 | Publication / serving | Rejected candidate leaves prior publication intact with visible failure/staleness. |
| NFR-001 | 1, 3–4 | Configuration | Adding a city requires configuration, not a copied DAG. |
| NFR-002 | 1, 5–7 | Maintainability | Relational/business logic is in dbt and scoring configuration is centralized. |
| NFR-003 | 4, 11 | Observability | Forced failures are diagnosable through logs, run records, and Grafana. |
| NFR-004 | 4, 11 | Reliability | Retry scenarios pass; measured success includes missing scheduled runs. |
| NFR-005 | 3–9 | Reliability | Retries/replays preserve logical counts and accepted-artifact mapping. |
| NFR-006 | 8, 10–11 | Serving | Ingestion/quality failures do not remove the last valid dataset. |
| NFR-007 | 4, 11 | Freshness | Per-city valid Bronze source age meets 65-minute target under stated operation. |
| NFR-008 | 8, 11 | Latency | Publication minus earliest selected successful landing is under 10 minutes. |
| NFR-009 | 5, 8, 11 | Quality | Report >=98% GE pass target separately from all-critical-pass gates; injected drift is visible. |
| NFR-010 | 9, 11 | Completeness | Expected active city-hour denominator yields >=95% over an observed 30-day window. |
| NFR-011 | 10–11 | Performance | Default Today rendering is measured against 2 seconds with hardware/cache context. |
| NFR-012 | 1–2, 10–11 | Security | No committed secrets; dashboard cannot read raw or mutate analytical data. |
| NFR-013 | 1–12 | Architecture | Stage modules are separated and component behavior is independently testable. |
| NFR-014 | 1–12 | Engineering | Lint, formatting, unit tests and growing integration/dbt CI pass per step. |
| NFR-015 | 0–12 | Documentation | Contracts, decisions, setup, models, methodology and runbooks reflect delivered behavior. |
| NFR-016 | 1–2, 12 | Reproducibility | Clean-clone Compose setup and deterministic demonstration succeed. |
| DQ-001 | 3, 5 | Quality | Missing/renamed fields and incompatible types block; additive drift is recorded. |
| DQ-002 | 5 | Quality | Humidity 0 and 100 pass; out-of-range values fail. |
| DQ-003 | 5 | Quality | Missing temperature, humidity or valid timestamp blocks conformance. |
| DQ-004 | 5 | Silver | Repeated city/source/valid-time observation is canonical once; raw responses retained. |
| DQ-005 | 5 | Temporal quality | Invalid timestamps/timezone metadata quarantine the city batch. |
| DQ-006 | 5 | Quality | Explicit optional nulls pass without bypassing required-field checks. |
| DQ-007 | 7, 11 | Monitoring | Configured metric jumps emit advisory alerts without automatically failing valid data. |
| DM-001 | 6 | Modeling | dim_city has one row per city and documented Type 1 behavior. |
| DM-002 | 6 | Modeling | dim_date has unique Manila calendar dates; unknown season is explicit. |
| DM-003 | 6 | Modeling | dim_hour has one row for each local hour 0–23. |
| DM-004 | 6–7 | Modeling | Profile keys are unique and prior configuration versions remain retrievable. |
| DM-005 | 6 | Modeling | One canonical current-condition sample per city/hour; missing hours are not forecast-filled. |
| DM-006 | 6 | Modeling | Forecast grain includes snapshot identity; multiple versions coexist. |
| DM-007 | 7–8 | Modeling | Current score grain remains city/hour/profile; versioned history is separate. |
| DM-008 | 7 | Modeling | Alert key includes forecast and threshold versions; replay does not duplicate events. |
| ORCH-001 | 4–8 | Orchestration | Hourly tasks enforce extract/validate/transform/gate/publish; failure metrics preserve failed state. |
| ORCH-002 | 9 | Orchestration | Daily DAG builds aggregates, trends, percentile context from eligible history. |
| ORCH-003 | 4, 9 | Scheduling | Hourly collection and 00:15 Manila daily schedule are verified at date boundaries. |
| ORCH-004 | 4 | Retries | Three native bounded-backoff retries; deterministic errors fail directly; no nested retry loop. |
| ORCH-005 | 5–9 | Retries | Transformation tasks have one retry and retain failure evidence. |
| ORCH-006 | 4, 11 | Operations | Final-failure callback records run/task/error context without requiring external messaging. |
| ORCH-007 | 9 | Dependencies | Daily task checks all prior-day expected coverage, times out, and repairs via replay. |

## Dependency and learning gates

| Gate | Required delivery | Evidence before advancing |
|---|---|---|
| Contracts | Step 0 | All 58 IDs mapped; walkthrough covers retry, new forecast, formula change, missing city and failed publication. Explain why identities differ. |
| Reproducible baseline | Steps 1–2 | Configuration checks, clean startup, migration/persistence and role checks. Explain configuration versus state. |
| Durable scheduled inputs | Steps 3–4 | Preserved bodies, interrupted-write recovery, retries and failure records. Explain idempotent effects. |
| Trusted history | Steps 5–6 | Critical gates, stable grains, incremental replay and model tests. Explain event time versus processing time. |
| Governed serving | Steps 7–8 | Formula references, versioned history, atomic publication failure drills. Explain why passing tests precedes visibility. |
| Historical product | Steps 9–10 | Complete-period rollups, no future-information leakage, Gold-only UI and timing. Explain population and cache freshness. |
| Operational portfolio | Steps 11–12 | Soak measurements, restore drill, clean-clone demo and honest documentation. Explain measured guarantees and limitations. |

A learning review asks you to predict a failure outcome, inspect the evidence,
and explain the tradeoff in your own words. It is not a request to memorize tool
syntax. Do not advance to the next step without the user's explicit request.

## Recommended Step 0 Git workflow

Branch: `docs/reconcile-data-contracts`. These are recommendations; this step does
not automatically create a branch or commits. Review the complete diff first.

| Commit | Exact scope | Why separate | Validation before committing |
|---|---|---|---|
| `docs(architecture): reconcile source and temporal contracts` | DATA_CONTRACTS.md, ADR-001, ADR-002, REQUIREMENTS.md, PROJECT_SPEC.md | One coherent semantic change covers identity, source meaning, critical gates and publication requirements. | Requirement IDs unchanged; source citations/local links resolve; walkthrough invariants agree with the requirements. |
| `docs(plan): align delivery gates and requirement traceability` | ADR-003, REQUIREMENT_TRACEABILITY.md, IMPLEMENTATION_PLAN.md (local roadmap status updated separately; Git-ignored) | Aligns workflow, dependency order, measurements and delivery tracking around the contract baseline. | All 58 requirement IDs mapped once; all 13 roadmap steps/commit tables remain; no runtime files changed; Markdown diff passes whitespace checks. |

No application tests are claimed for a documentation-only change. Runtime
scenarios above become tests when their components are implemented. After this
step's learning review, request Step 1 to establish packaging, validated settings,
linting, tests, and CI; do not begin those changes during Step 0.
