# POC 5 --- Logistics & Supply Chain Control Tower

**Difficulty:** Complex\
**Domain:** Logistics / supply chain\
**Primary interview themes:** Multi-source integration, entity
resolution, event-driven state, geospatial/time analytics, master data,
orchestration dependencies, end-to-end SLAs.

## 1. Business problem

A logistics company receives orders from a sales system, warehouse
inventory from MySQL, shipment milestones from carrier feeds, and
delivery exceptions from PostgreSQL. Operations leaders need a
near-current view of order fulfillment, late shipments, inventory risk,
carrier performance, and order-to-delivery cycle time.

## 2. Objectives

-   Integrate relational sources and file-based carrier feeds with
    different schemas.
-   Create a canonical order/shipment event model.
-   Resolve duplicated or conflicting tracking events.
-   Handle out-of-order milestones and late corrections.
-   Publish operational KPIs with lineage and reconciliation.
-   Demonstrate a reliable, replayable pipeline and a clear path from
    batch to more frequent refresh.

## 3. Reference architecture

``` mermaid
flowchart LR
  A[MySQL orders + inventory] --> D[Fabric Data Factory]
  B[PostgreSQL warehouse + exceptions] --> D
  C[Carrier CSV / JSON feeds] --> D
  D --> E[Bronze Lakehouse]
  E --> F[PySpark schema mapping + dedup]
  F --> G[Silver canonical orders and events]
  G --> H[SQL / PySpark state reconstruction]
  H --> I[Gold control-tower marts]
  I --> J[Warehouse / SQL endpoint]
  J --> K[Operational dashboard]
  L[Run audit + DQ + reconciliation] --- D
  L --- F
```

## 4. Sources and schemas

MySQL: -
`orders(order_id, customer_token, order_created_at, promised_delivery_at, destination_region, order_status, updated_at)` -
`order_lines(order_id, sku, ordered_qty, unit_price)` -
`inventory(sku, warehouse_id, available_qty, reserved_qty, snapshot_ts)`

PostgreSQL: -
`shipments(shipment_id, order_id, carrier_code, tracking_id, dispatch_ts, updated_at)` -
`delivery_exceptions(exception_id, shipment_id, event_ts, exception_code, resolution_status)` -
`warehouses(warehouse_id, region, capacity, effective_from, effective_to)`

Carrier feed:
`carrier_event_id, tracking_id, event_code, event_ts, received_ts, location_code, source_sequence`

Use synthetic customer tokens and addresses. Avoid storing real
recipient details.

## 5. End-to-end pipeline

1.  **Ingest:** orchestrate MySQL/PostgreSQL incremental extracts and
    carrier CSV/JSON file arrival.
2.  **Land raw:** retain original feed payloads and metadata, including
    carrier, file name, arrival time, schema version, and run ID.
3.  **Validate contracts:** check required fields, data types, known
    event codes, file completeness, and expected volume.
4.  **Normalize:** map carrier-specific event codes to canonical
    milestones such as picked up, in transit, out for delivery,
    delivered, and exception.
5.  **Resolve identity:** map tracking IDs and shipments to orders using
    governed keys; quarantine ambiguous matches.
6.  **Deduplicate:** use carrier event ID when reliable, otherwise a
    documented composite key and deterministic tie-breaker.
7.  **Reconstruct state:** sort milestones by event time and source
    sequence, retain the event history, and calculate current shipment
    status.
8.  **Handle late data:** recompute impacted shipment states and KPI
    windows when a delayed event or correction arrives.
9.  **Enrich:** join order lines, warehouse inventory, carrier, region,
    and effective-dated warehouse reference data.
10. **Calculate KPIs:** on-time delivery, order cycle time, exception
    rate, inventory availability, fulfillment completeness, and carrier
    performance.
11. **Publish:** expose curated gold marts through a Warehouse/SQL
    endpoint and dashboard.
12. **Reconcile and monitor:** compare order/shipment counts, track
    unmatched events, monitor freshness, and alert on abnormal drops or
    late feeds.

## 6. Technology responsibilities

-   **Microsoft Fabric:** orchestration, Lakehouse/OneLake, notebooks,
    Delta tables, Warehouse/SQL endpoint.
-   **SQL:** state/event queries, SLA metrics, window functions,
    dimensional models, exception analysis.
-   **Python:** feed simulators, schema mapping config, reusable
    validators, test harness.
-   **PySpark:** distributed parsing, canonicalization, deduplication,
    state reconstruction, incremental writes.
-   **MySQL:** order and inventory source.
-   **PostgreSQL:** shipment, warehouse, and exception source.

## 7. KPI definitions

-   **On-time delivery rate:** delivered shipments with actual delivery
    at or before the promised timestamp divided by eligible delivered
    shipments; report undelivered overdue shipments separately.
-   **Order cycle time:** delivery timestamp minus order creation
    timestamp, with explicit rules for split shipments.
-   **Exception rate:** shipments with at least one qualifying exception
    divided by shipments in scope.
-   **Fulfillment completeness:** shipped quantity divided by ordered
    quantity, with partial shipment handling defined.
-   **Available inventory:** define whether this is `available_qty`, or
    on-hand less reservations and safety stock. Do not mix definitions
    across sources.

Document each definition and test it with small hand-calculated
examples.

## 8. Data modeling

Suggested gold model: - `fact_order_line` --- one row per order line. -
`fact_shipment_event` --- one row per canonical shipment event. -
`fact_shipment_current` --- one row per shipment as of the latest
accepted event. - `fact_inventory_snapshot` --- one row per
SKU/warehouse/snapshot. - `dim_carrier`, `dim_warehouse`, `dim_date`,
`dim_region`, `dim_event_type`.

Retain event history separately from current state so a corrected status
does not erase the audit trail.

## 9. Reliability and best practices

-   Use idempotent ingestion and Delta merge logic.
-   Do not advance source watermarks until downstream commits and
    quality gates succeed.
-   Preserve raw carrier payloads to enable replay after mapping fixes.
-   Version canonical event-code mappings and apply effective dates if
    mappings change.
-   Keep source-specific watermarks separate.
-   Quarantine ambiguous tracking-to-shipment joins.
-   Apply schema contracts, freshness SLAs, row-count thresholds, and
    reconciliation by source and date.
-   Use secret management and least-privilege access; never commit
    credentials.
-   Monitor Spark skew, file sizes, partition choices, and SQL query
    plans.
-   Keep pipeline parameters externalized by environment; use separate
    development/test/production configuration.
-   Define retry, timeout, backfill, and incident-response procedures.

## 10. Test plan

-   Carrier sends the same event twice.
-   Delivery event arrives before an earlier in-transit event.
-   A tracking ID maps to multiple candidate shipments.
-   A promised delivery date is updated after dispatch.
-   Shipment is split across multiple deliveries.
-   Carrier changes an event code or adds a field.
-   One source is delayed while other sources succeed.
-   Rerun/backfill does not duplicate events or inflate KPIs.
-   Source-to-target reconciliation and hand-calculated KPI tests.
-   Load test with millions of synthetic events and a skewed major
    carrier.

## 11. Interview SQL exercises

1.  Calculate on-time delivery rate by carrier and month.
2.  Use `LAG` to detect missing or out-of-order shipment milestones.
3.  Find orders that are overdue and have no delivery event.
4.  Rank warehouses by fulfillment delays and inventory shortage.
5.  Calculate the 90th-percentile order cycle time by region.
6.  Identify SKUs with repeated stockout risk based on recent demand and
    available inventory.

## 12. Deliverables

-   MySQL/PostgreSQL DDL and synthetic carrier-feed generator.
-   Fabric pipeline with parameterized ingestion and dependencies.
-   Bronze/Silver/Gold notebooks and schemas.
-   Canonical event-code mapping and KPI dictionary.
-   SQL interview query pack.
-   Automated tests, reconciliation report, runbook, architecture
    diagram, and dashboard.

## 13. Stretch goals

Add CDC, more frequent file-arrival triggers, a simulated event broker,
data contracts for carriers, and a late-event replay service. Compare
the complexity and operational costs of hourly batch, micro-batch, and
streaming approaches.
