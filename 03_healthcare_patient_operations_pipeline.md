# POC 3 --- Healthcare Operations & Appointment Analytics Pipeline

**Difficulty:** Advanced\
**Domain:** Healthcare operations\
**Primary interview themes:** Data privacy, interoperability, data
quality, slowly changing reference data, temporal joins, governed
access.

## 1. Business problem

A fictional hospital network has appointment, encounter, provider,
department, and billing datasets stored across PostgreSQL and MySQL.
Operations teams need reliable reports on appointment access, waiting
times, cancellations, utilization, and billing completeness. The POC
uses synthetic data only.

## 2. Goals

-   Combine datasets with different schemas and update patterns.
-   Standardize codes, timestamps, and operational statuses.
-   Preserve historical provider/department mappings.
-   Publish governed aggregate metrics without exposing unnecessary
    patient-level data.
-   Demonstrate data lineage, quality controls, and controlled access.

## 3. Architecture

``` mermaid
flowchart LR
  A[MySQL scheduling] --> D[Fabric Data Factory]
  B[PostgreSQL encounters and billing] --> D
  D --> E[Bronze Lakehouse]
  E --> F[PySpark standardization]
  F --> G[Silver conformed healthcare operations]
  G --> H[Gold operational marts]
  H --> I[Warehouse / SQL endpoint]
  I --> J[Aggregate dashboards]
  K[Quality checks + audit log] --- D
  K --- F
```

## 4. Synthetic source tables

MySQL: -
`appointments(appointment_id, patient_token, provider_id, department_id, scheduled_start, scheduled_end, status, updated_at)` -
`provider_roster(provider_id, specialty_code, department_id, effective_from, effective_to, updated_at)`

PostgreSQL: -
`encounters(encounter_id, appointment_id, checkin_ts, consultation_start_ts, checkout_ts, encounter_status)` -
`billing_claims(claim_id, encounter_id, charge_amount, claim_status, submitted_at, updated_at)` -
`departments(department_id, department_name, region, effective_from, effective_to)`

Use synthetic patient tokens that cannot be mapped to real people. Avoid
real medical notes, diagnoses, or other health data.

## 5. Pipeline design

1.  Extract each source with a source-specific watermark and schema
    contract.
2.  Land immutable Bronze tables with batch ID, extraction time, source
    system, and source key.
3.  Validate key uniqueness, timestamp order, allowed statuses, valid
    provider/department references, and amount constraints.
4.  Standardize time zones, status values, code formats, and null
    conventions.
5.  Deduplicate updates by primary key and source update timestamp.
6.  Join appointments to encounters and claims while preserving
    unmatched records for investigation.
7.  Build effective-dated dimensions for provider specialty and
    department history.
8.  Calculate operational metrics: no-show rate, cancellation rate,
    median waiting time, provider utilization, claim submission lag, and
    appointment-to-encounter match rate.
9.  Publish aggregated marts. Use row-level patient detail only for a
    restricted debugging table if genuinely required.
10. Reconcile appointment counts to source totals and report exclusions
    explicitly.
11. Monitor freshness, late changes, data quality, and pipeline
    failures.

## 6. Technology roles

-   **Fabric:** orchestration, Lakehouse, notebook transformations,
    curated Warehouse/SQL endpoint.
-   **SQL:** temporal joins, window functions, aggregate marts, anomaly
    queries.
-   **Python:** schema/quality helpers, synthetic-data generation, unit
    tests.
-   **PySpark:** cross-source cleansing, deduplication, large joins,
    Delta writes.
-   **MySQL:** scheduling system source.
-   **PostgreSQL:** encounters and billing source.

## 7. Metric definitions to agree before coding

-   **Waiting time:** consultation start minus check-in, excluding
    impossible negative intervals and defining how missing timestamps
    are handled.
-   **No-show rate:** no-shows divided by eligible scheduled
    appointments; exclude cancellations according to the written rule.
-   **Provider utilization:** occupied appointment minutes divided by
    available scheduled minutes, with breaks and blocked slots treated
    explicitly.
-   **Claim lag:** claim submission timestamp minus encounter checkout
    timestamp.
-   **Match rate:** appointments linked to a valid encounter divided by
    the defined eligible appointment population.

Every metric should have a business definition, SQL implementation,
owner, and test.

## 8. Privacy and governance

-   Use synthetic data; do not upload identifiable patient data to a
    personal development workspace.
-   Minimize fields in curated layers and remove unnecessary direct
    identifiers.
-   Restrict raw access more tightly than aggregate access.
-   Keep secrets out of notebooks, source control, logs, and output
    files.
-   Document retention and deletion expectations, access roles, lineage,
    and permitted use.
-   Treat the design as a learning exercise, not a claim of compliance
    with any healthcare regulation.

## 9. Reliability, performance, and testing

-   Idempotent upserts and replayable raw data.
-   Independent watermarks for each source.
-   Quarantine invalid rows with reason codes.
-   Effective-dated join tests at boundary timestamps.
-   Tests for daylight-saving/time-zone conversion if relevant to the
    selected region.
-   Test missing encounter, duplicate appointment, changed provider
    specialty, and late claim.
-   Reconcile totals at source, Bronze, Silver, and Gold levels.
-   Inspect Spark plans, join skew, partition sizing, and small-file
    accumulation.
-   Add freshness SLAs and alert on unexpected drops in volume.

## 10. Example SQL interview questions

1.  Calculate no-show rates by department and month.
2.  Find the median and 90th-percentile waiting time by specialty.
3.  Identify appointments without a matching encounter.
4.  Join each appointment to the provider specialty valid on its
    scheduled date.
5.  Calculate claim submission lag and the percentage submitted within a
    target interval.

## 11. Deliverables

-   Source DDL and synthetic-data generator.
-   Fabric pipeline, notebooks, and curated table definitions.
-   Metric dictionary and privacy/governance notes.
-   SQL exercises and data-quality dashboard.
-   Automated tests, reconciliation report, lineage/architecture
    diagrams, and runbook.

## 12. Stretch goals

Add FHIR-shaped synthetic records, a slowly changing provider dimension,
data product ownership, and separate restricted versus aggregate
workspaces. Explain why standards-shaped data is not automatically
semantically correct.
