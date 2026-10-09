# POC 1 --- Retail & E-commerce Analytics Data Pipeline

**Difficulty:** Medium\
**Domain:** Retail / e-commerce\
**Primary interview themes:** Incremental ingestion, dimensional
modeling, SQL analytics, data quality, slowly changing dimensions,
orchestration, reconciliation.

## 1. Business problem

A retailer receives order, order-line, customer, product, payment, and
inventory data from an operational MySQL database. Analysts need trusted
daily sales, margin, returns, inventory, and customer metrics. The
pipeline must cope with late updates, duplicate extracts, cancelled
orders, and changing customer/product attributes.

## 2. Goals and acceptance criteria

-   Ingest source data incrementally and preserve a traceable raw
    history.
-   Build a cleansed, conformed model with documented keys and business
    rules.
-   Publish daily sales, returns, gross margin, and inventory metrics.
-   Reprocess a date range safely without duplicating facts.
-   Detect missing data, invalid amounts, orphan keys, and
    source-to-target count/value mismatches.
-   Demonstrate monitoring, restartability, and a documented recovery
    procedure.

Suggested measurable targets for a synthetic dataset: process 1 million
order lines in under 15 minutes on a modest development environment;
achieve zero duplicate business keys in curated facts; reconcile source
totals to target totals within an explicitly defined tolerance.

## 3. Suggested architecture

``` mermaid
flowchart LR
  A[MySQL OLTP] --> B[Fabric Data Factory pipeline]
  B --> C[Raw / Bronze Lakehouse]
  C --> D[PySpark cleansing and deduplication]
  D --> E[Silver Delta tables]
  E --> F[Gold dimensional model]
  F --> G[Fabric Warehouse / SQL endpoint]
  G --> H[Power BI or SQL analysis]
  I[Pipeline audit + quality results] --> J[Monitoring and alerts]
  B --> I
  D --> I
```

Use Microsoft Fabric Lakehouse, OneLake, Data Factory pipelines,
notebooks, Delta tables, and a Warehouse or SQL analytics endpoint. If a
MySQL connector or gateway is unavailable in the chosen environment,
export repeatable CSV/Parquet snapshots and document that as a
source-ingestion substitute.

## 4. Source data and schema

Create synthetic MySQL tables or CSV extracts:

-   `customers(customer_id, name, email, city, state, created_at, updated_at)`
-   `products(product_id, sku, category, unit_cost, list_price, updated_at)`
-   `orders(order_id, customer_id, order_ts, status, currency, updated_at)`
-   `order_items(order_id, line_id, product_id, quantity, unit_price, discount_amount)`
-   `payments(payment_id, order_id, payment_ts, amount, payment_status, updated_at)`
-   `returns(return_id, order_id, line_id, return_ts, quantity, refund_amount)`
-   `inventory_snapshots(snapshot_date, product_id, warehouse_id, on_hand_qty)`

Use generated data rather than personal or production customer
information.

## 5. End-to-end pipeline

1.  **Extract:** Capture a consistent source watermark (`updated_at` or
    an increasing ID). Persist the last successful watermark in a
    control table.
2.  **Land raw:** Write source name, ingestion timestamp, batch/run ID,
    source file or extract ID, and schema version alongside unmodified
    records.
3.  **Validate arrival:** Check required files/tables, expected columns,
    non-empty batches, and reasonable volume thresholds.
4.  **Cleanse:** Normalize timestamps and currencies; trim strings;
    standardize status values; quarantine malformed records rather than
    silently dropping them.
5.  **Deduplicate:** Select the latest record by business key and source
    update time, with deterministic tie-breaking.
6.  **Apply incremental changes:** Use Delta `MERGE`/upsert logic for
    current-state entities. Preserve raw history for audit and replay.
7.  **Model:** Create dimensions (`dim_customer`, `dim_product`,
    `dim_date`) and facts (`fact_sales`, `fact_returns`,
    `fact_inventory_snapshot`). Use surrogate keys where appropriate.
8.  **Build aggregates:** Publish daily sales by
    product/category/region, return rate, gross margin, average order
    value, and inventory coverage.
9.  **Reconcile:** Compare source and target row counts, distinct order
    counts, and financial sums by batch/date.
10. **Publish:** Expose curated tables through a Warehouse or SQL
    endpoint and create example SQL queries / a small dashboard.
11. **Operate:** Record run status, duration, watermarks, row counts,
    rejected rows, and error summaries.

## 6. Technology responsibilities

-   **Microsoft Fabric:** Data Factory orchestration, Lakehouse/OneLake
    storage, notebook execution, Delta tables, Warehouse/SQL endpoint,
    monitoring.
-   **SQL:** Joins, aggregations, CTEs, window functions, deduplication
    logic, dimensional modeling, query optimization.
-   **Python:** Config handling, reusable validation functions,
    date/watermark utilities, test-data generation, logging.
-   **PySpark:** Distributed cleansing, type casting, deduplication,
    joins, Delta writes/merges, quality checks.
-   **MySQL:** OLTP source design, indexing on primary keys and
    incremental watermark columns.
-   **PostgreSQL (optional extension):** Mirror the source or host
    pipeline metadata/control tables; compare SQL dialect and query
    plans.

## 7. Data model and business rules

-   Define revenue consistently: for example,
    `quantity * unit_price - discount_amount`, excluding cancelled
    orders and separately accounting for refunds.
-   Decide whether tax, shipping, and currency conversion are included;
    document the decision.
-   Treat returns as a separate fact, not as overwritten sales.
-   Choose SCD Type 1 for correcting customer contact attributes and SCD
    Type 2 for historical product category/cost changes if historical
    reporting requires them.
-   Define the fact grain explicitly: one row per order line for sales,
    one row per return line for returns, one row per
    product/warehouse/date for inventory snapshots.

## 8. Reliability and engineering standards

-   Idempotency: rerunning a batch must not create duplicate facts.
-   Restartability: persist run IDs and watermarks; advance the
    watermark only after all required stages succeed.
-   Late-arriving changes: use a configurable lookback window or CDC
    where available.
-   Schema evolution: detect unexpected columns/type changes and require
    an explicit compatibility decision.
-   Quarantine: keep rejected rows with reason codes and run IDs.
-   Security: least-privilege credentials, secrets in approved secret
    storage, no credentials in notebooks or Git.
-   Observability: structured logs, pipeline-level status, row counts,
    duration, freshness, and alert thresholds.
-   Performance: partition large facts by a useful date key when
    justified; avoid excessive small files; inspect Spark plans and SQL
    query plans.

## 9. Test plan

-   Unit tests for revenue calculations, status normalization, and
    watermark logic.
-   Data tests for unique keys, non-null required fields, valid
    quantities, non-negative amounts where appropriate, and referential
    integrity.
-   Integration test with a small source snapshot.
-   Idempotency test: run the same batch twice and compare
    counts/totals.
-   Late-update test: update an older order and verify it is reflected.
-   Failure-recovery test: simulate a failed publish stage and verify
    the watermark does not advance prematurely.
-   Reconciliation test: source/target totals match under documented
    exclusions.

## 10. Example interview SQL questions

1.  Find the top 10 categories by net revenue for the last 30 days.
2.  Calculate month-over-month revenue growth using `LAG`.
3.  Identify customers whose first purchase was in the current month.
4.  Calculate return rate by category and flag categories above a
    threshold.
5.  Find products with low stock and high 30-day sales velocity.

## 11. Deliverables

-   Architecture diagram and data dictionary.
-   Source DDL and synthetic-data generator.
-   Fabric pipeline and notebooks.
-   Bronze/Silver/Gold table definitions.
-   SQL scripts for metrics and reconciliation.
-   Automated tests and sample test report.
-   Runbook, configuration example, and sample dashboard/screenshots.

## 12. Stretch goals

Add CDC, SCD Type 2, multi-currency conversion, incremental
semantic-model refresh, and a second source in PostgreSQL. Explain the
trade-offs and costs of each addition.
