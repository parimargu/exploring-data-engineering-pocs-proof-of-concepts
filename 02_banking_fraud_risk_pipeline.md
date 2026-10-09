# POC 2 --- Banking Transactions & Fraud-Risk Analytics Pipeline

**Difficulty:** Medium--Advanced\
**Domain:** Banking / payments\
**Primary interview themes:** Event-time processing, risk features, data
governance, auditability, imbalanced data, late events, secure handling.

## 1. Business problem

A simulated payments provider needs a dependable analytics pipeline to
analyze card or account transactions, identify suspicious patterns, and
produce daily risk summaries. This is an analytics and learning
POC---not a production fraud decision system and not a substitute for
regulated controls.

## 2. Outcomes

-   Integrate transaction, account, merchant, and case-review data.
-   Handle duplicates, reversals, delayed events, and corrections.
-   Build reusable transaction-level risk features and daily aggregates.
-   Create an auditable suspicious-activity review dataset.
-   Measure data quality and model/heuristic performance without leaking
    future information.

## 3. Architecture

``` mermaid
flowchart TD
  A[PostgreSQL payment simulator] --> B[Fabric ingestion pipeline]
  B --> C[Bronze raw transaction events]
  C --> D[PySpark validation and event dedup]
  D --> E[Silver canonical transactions]
  E --> F[SQL + PySpark feature engineering]
  F --> G[Gold risk features and daily aggregates]
  G --> H[Warehouse / Power BI]
  I[Case outcomes / labels] --> C
  J[Audit + quality + run metadata] --- B
  J --- D
  J --- F
```

## 4. Sample source tables

-   `transactions(transaction_id, account_id, merchant_id, event_ts, ingestion_ts, amount, currency, channel, status, updated_at)`
-   `accounts(account_id, opened_at, account_type, risk_segment, country_code)`
-   `merchants(merchant_id, category, country_code, onboarding_date)`
-   `transaction_events(event_id, transaction_id, event_type, event_ts, source_sequence)`
-   `fraud_cases(case_id, transaction_id, opened_at, reviewed_at, outcome, reason_code)`

Use synthetic accounts and transactions only. Never use real card
numbers, credentials, or personally identifiable banking records.

## 5. Pipeline stages

1.  Generate realistic synthetic events, including legitimate activity,
    suspicious patterns, reversals, and late arrivals.
2.  Ingest source changes using an increasing event ID, CDC where
    supported, or a repeatable watermark extract.
3.  Store immutable raw events with ingestion metadata.
4.  Validate schema, event IDs, timestamps, currency codes, and amount
    bounds.
5.  Deduplicate using `event_id` and source sequence; keep the raw
    record for audit.
6.  Resolve transaction lifecycle events into a canonical transaction
    state. Do not treat a reversal as an ordinary negative sale without
    an explicit rule.
7.  Build time-aware features such as transaction count and sum per
    account over prior 1-hour, 24-hour, and 7-day windows.
8.  Join merchant/account reference data using event-time-valid records
    when historical correctness matters.
9.  Create gold tables for daily transaction volume/value,
    country/category breakdowns, rule-triggered review queues, and case
    outcomes.
10. Evaluate heuristics or a simple baseline model against synthetic
    labels. Split train/test data chronologically to avoid future
    leakage.
11. Publish metrics and provide a clear disclaimer that automated flags
    are review signals, not proof of fraud.

## 6. Technology mapping

-   **Fabric Data Factory:** schedule and dependency management.
-   **Lakehouse/Delta:** raw event retention, curated tables, replay.
-   **PySpark:** high-volume event cleansing and rolling feature
    calculations.
-   **SQL:** analytical windows, cohort summaries, joins, explainable
    rules.
-   **Python:** synthetic data, reusable validators, baseline metrics.
-   **PostgreSQL:** transaction/event source and case-management
    simulation.
-   **MySQL extension:** load a separate merchant or customer reference
    dataset and compare source integration approaches.

## 7. Important design decisions

-   **Event time vs ingestion time:** retain both. A late event can
    change a historical window.
-   **Deduplication:** define the stable event key and deterministic
    conflict resolution.
-   **Reversals and chargebacks:** model lifecycle events explicitly.
-   **Point-in-time correctness:** feature values used for a historical
    prediction must only include data available at that time.
-   **Labels:** case outcomes may be delayed and biased; document
    unknown/unreviewed outcomes.
-   **Privacy:** use synthetic identifiers, restrict access, and avoid
    sensitive fields in logs.
-   **Explainability:** store which rules/features caused a flag and the
    rule version.

## 8. Example explainable rules

These are illustrative thresholds to test, not universal fraud
indicators: - Many transactions in a short interval compared with the
account's baseline. - Unusual geographic or merchant-category pattern
relative to synthetic historical activity. - Repeated authorization
failures followed by a successful payment. - A burst of small
transactions followed by a high-value transaction.

Avoid hard-coding a rule as a final decision. Track false positives and
false negatives and discuss the cost of each.

## 9. Reliability and best practices

-   Store immutable raw events and versioned rule definitions.
-   Make feature jobs idempotent and support bounded recomputation for
    late events.
-   Track event freshness, late-arrival rate, duplicate rate, null
    rates, and feature-table completeness.
-   Use explicit schema contracts and quarantine invalid records.
-   Apply least privilege, encryption supported by the platform, and
    secret management.
-   Ensure retry policies do not double-count transactions.
-   Monitor skewed account keys and avoid expensive unbounded window
    operations.
-   Maintain a runbook for replaying a time interval and reconciling
    totals.

## 10. Tests

-   Duplicate event delivery.
-   Late event arriving after daily aggregate creation.
-   Reversal and status correction.
-   Invalid currency, timestamp, or amount.
-   Point-in-time feature test to detect leakage.
-   Re-run the same interval and verify identical outputs.
-   Chronological model evaluation and baseline comparison.
-   Reconciliation of transaction counts and sums by processing date.

## 11. Interview SQL exercises

1.  Rank accounts by transaction value in a rolling time window.
2.  Use `LAG` to compare each transaction with the previous transaction
    for the same account.
3.  Calculate merchant-level approval and reversal rates.
4.  Find accounts with an unusual count relative to their own historical
    average.
5.  Report precision, recall, and false-positive rate from synthetic
    case outcomes.

## 12. Deliverables

-   Synthetic event generator and PostgreSQL DDL.
-   Fabric orchestration pipeline and notebooks.
-   Data dictionary, feature definitions, rule catalog, and lineage
    diagram.
-   Quality/reconciliation tables and test results.
-   SQL portfolio queries and a dashboard.
-   Security notes, model limitations, and replay runbook.

## 13. Stretch goals

Add CDC, a streaming/event-hub source, model registry/version tracking,
drift monitoring, and a human-review feedback loop. Keep the baseline
reproducible and explain why a batch POC does not prove production
low-latency capability.
