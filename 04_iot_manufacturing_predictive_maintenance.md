# POC 4 --- Manufacturing IoT & Predictive Maintenance Pipeline

**Difficulty:** Advanced--Complex\
**Domain:** Manufacturing / industrial IoT\
**Primary interview themes:** Time-series data, high-volume ingestion,
late/out-of-order events, incremental aggregates, feature engineering,
operational reliability.

## 1. Business problem

A fictional factory collects machine telemetry (temperature, vibration,
pressure, runtime) and maintenance work orders. Engineering teams want
to understand equipment health, detect abnormal operating patterns, and
estimate maintenance risk. Start with batch files and optionally add
streaming as a second phase.

## 2. Outcomes

-   Ingest high-volume time-series readings and reference/work-order
    data.
-   Process out-of-order and duplicated telemetry safely.
-   Produce hourly/daily equipment metrics and explainable anomaly
    flags.
-   Build a labeled maintenance dataset without target leakage.
-   Demonstrate incremental recomputation, performance tuning, and
    operational monitoring.

## 3. Architecture

``` mermaid
flowchart TD
  A[Telemetry simulator / Parquet files] --> B[Fabric Data Factory]
  C[PostgreSQL asset + work orders] --> B
  B --> D[Bronze Delta raw events]
  D --> E[PySpark validation and event-time handling]
  E --> F[Silver canonical telemetry]
  F --> G[Hourly features and anomaly rules]
  C --> H[Silver maintenance records]
  G --> I[Gold equipment health mart]
  H --> I
  I --> J[Warehouse / SQL analytics]
  K[Audit, quality, freshness] --- B
  K --- E
```

## 4. Data model

Telemetry event:
`event_id, asset_id, event_ts, ingest_ts, temperature, vibration_rms, pressure, runtime_hours, firmware_version`

Reference tables: -
`assets(asset_id, line_id, machine_type, commissioned_at, retired_at)` -
`sensor_registry(sensor_id, asset_id, metric_name, unit, valid_from, valid_to)` -
`maintenance_orders(order_id, asset_id, opened_at, completed_at, maintenance_type, failure_code)` -
`downtime_events(event_id, asset_id, start_ts, end_ts, reason_code)`

Generate synthetic events at a configurable frequency and volume. Store
timestamps in UTC and retain the source time zone if one exists.

## 5. Pipeline stages

1.  Generate time-series data with normal cycles, sensor drift, missing
    intervals, duplicates, outliers, and simulated failure events.
2.  Ingest files or database increments with run ID, event date, schema
    version, and ingestion timestamp.
3.  Validate event keys, asset/sensor references, units, timestamp
    ranges, and numeric bounds.
4.  Preserve raw records, then deduplicate by event ID or a documented
    natural key.
5.  Normalize units and align sensor records with the sensor registry
    effective at event time.
6.  Detect late/out-of-order events and recompute affected hourly
    windows.
7.  Create hourly features: mean, min, max, standard deviation,
    percentiles, rate of change, missing-reading ratio, and runtime
    deltas.
8.  Add rolling historical features while ensuring each feature uses
    only information available before the prediction timestamp.
9.  Join maintenance outcomes to construct training labels using a
    defined prediction horizon.
10. Publish gold equipment/day health metrics, downtime, anomaly events,
    and maintenance-risk features.
11. Track processing lag, event volume, late-event ratio, sensor
    coverage, and failed-batch count.

## 6. Technology responsibilities

-   **Fabric Data Factory:** schedule file/database ingestion and
    orchestrate dependencies.
-   **OneLake/Lakehouse/Delta:** scalable event storage and incremental
    curated tables.
-   **PySpark:** partitioned time-series transformations, window
    features, joins, and incremental processing.
-   **SQL:** downtime analysis, equipment rankings, feature inspection,
    and operational reports.
-   **Python:** data simulator, statistical baselines, test utilities,
    and evaluation.
-   **PostgreSQL:** asset registry and maintenance system.
-   **MySQL extension:** alternate sensor registry or production
    reference source.

## 7. Reliability and performance choices

-   Partition by event date only if query patterns and data volume
    justify it; avoid partitioning by high-cardinality asset ID.
-   Compact small files and monitor table growth.
-   Use checkpoint/watermark logic for incremental event processing;
    document the limits of batch approximations.
-   Reprocess a bounded lookback window for late events and define how
    older corrections are handled.
-   Make aggregates deterministic and idempotent.
-   Track units and sensor calibration versions; an apparently valid
    number can still be semantically wrong.
-   Detect clock skew, impossible timestamp order, and implausible
    sensor jumps.
-   Test skewed assets and avoid unbounded per-asset windows.
-   Use versioned feature definitions and preserve source lineage.

## 8. Anomaly and prediction approach

Begin with transparent statistical rules, such as a rolling z-score or
robust median/MAD threshold. Evaluate false alarms per asset/day and
compare against synthetic maintenance events. A later extension can
train a simple classifier, but use chronological validation, prevent
leakage, and document class imbalance. Do not present a simulated model
as proven industrial safety equipment.

## 9. Test cases

-   Duplicate telemetry event.
-   Late event changes an already published hourly aggregate.
-   Sensor switches units or calibration version.
-   Asset is retired or sensor registry changes.
-   Missing readings and time gaps.
-   Extreme but valid readings versus malformed values.
-   Reprocessing a day produces the same output.
-   Labeling test verifies no post-failure measurements enter
    pre-failure features.
-   Load test with increasing event volumes and skewed asset
    distributions.

## 10. Interview SQL exercises

1.  Rank assets by downtime in a rolling 30-day period.
2.  Calculate hourly average and peak vibration by asset.
3.  Find gaps in sensor readings using `LAG`.
4.  Compare temperature with the prior hour and compute percentage
    change.
5.  Measure maintenance frequency and mean time between failures.

## 11. Deliverables

-   Configurable telemetry generator.
-   Fabric pipeline and PySpark notebooks.
-   Delta schemas and SQL views.
-   Feature dictionary, anomaly rule catalog, and evaluation notebook.
-   Load-test results, quality/reconciliation reports, runbook, and
    architecture diagram.

## 12. Stretch goals

Implement a streaming ingestion variant, compare batch and streaming
semantics, add alert delivery simulation, and document late-event
handling, checkpointing, replay, and cost trade-offs.
