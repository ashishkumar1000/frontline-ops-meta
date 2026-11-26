# Inventory, Procurement & Supplier Management — High-Level Design

Reference: `functional-spec.md` (this module), `_context/new-modules-addendum.md` (module map row
19), `_context/conventions.md` (Target Tech Stack, Multi-tenancy Rule, State Machines),
`02-work-orchestration/functional-spec.md` §6 and `02-work-orchestration/low-level-design.md` §1.1
(Approval Routing — reused, not reimplemented).

## 1. Module Responsibilities

The **Inventory/Procurement Service** is the authoritative owner of:

- The `SKU` catalog and category hierarchy, unit-of-measure/conversion rules, and per-location
  reorder thresholds.
- The `StockCount` / `StockCountLine` business behavior layered on top of the `STOCK_COUNT`
  `WorkItem` type: variance computation, threshold flagging, shift-handoff continuity, post-count
  stock adjustment.
- The `PurchaseOrder` / `PurchaseOrderLine` aggregate and its own domain-specific lifecycle
  (`DRAFT -> ... -> CLOSED`, §5 of the architecture doc) — a lifecycle richer than the generic
  WorkItem state machine because a PO has post-approval states (`SENT_TO_SUPPLIER`,
  `PARTIALLY_RECEIVED`) with no WorkItem equivalent.
- Cost-threshold-to-`ApprovalRoute` resolution at PO submission time.
- `Supplier` master data and the `SupplierScorecard` rollup.
- `GoodsReceipt` / `GoodsReceiptLine` reconciliation against a PO.

It explicitly does **not** implement its own approval engine. Every multi-level approval in this
module — a purchase order crossing a cost threshold — is delegated in full to Work Orchestration's
existing `approval_routes` / `work_item_approval_steps` mechanism via a `PURCHASE_APPROVAL`
`WorkItem`. This service never stores a parallel "next approver" rule, a parallel step-status enum,
or a parallel escalation ladder. It stores only a foreign-key reference
(`purchase_orders.approval_work_item_id`) into Work Orchestration's WorkItem, and reacts to that
WorkItem's events.

It is a peer bounded context to Work Service, Issue/CAPA Service, and Form Service in the
platform's modular-domain architecture (source-spec §16.2): it justifies its own service because it
owns a distinct, long-lived transactional aggregate (SKU/PO/Supplier data) with its own query
patterns (catalog lookup, PO listing, scorecard aggregation) that do not belong inside the generic
Work Service.

## 2. Component Diagram

```mermaid
flowchart TD
    subgraph Clients
        MOB[Mobile App - counts, receiving]
        WEB[Web Admin / Control Tower - catalog, PO, suppliers]
    end

    GW[API Gateway / BFF]

    subgraph InvProcService [Inventory/Procurement Service]
        API[Inventory API Layer]
        CAT[SKU Catalog Manager]
        CNT[Stock Count Manager]
        VAR[Variance Calculator]
        PO[Purchase Order Manager]
        ROUTE[Cost-Threshold Route Resolver]
        RCV[Goods Receipt Manager]
        SUP[Supplier Manager]
        SCORE[Scorecard Rollup Engine]
        ALERT[Low-Stock Alert Sweep]
        OUT[Transactional Outbox]
    end

    PG[(Postgres: skus, stock_counts,\nstock_count_lines, purchase_orders,\npurchase_order_lines, suppliers,\nsupplier_scorecards, goods_receipts)]
    REDIS[(Redis: catalog cache,\nidempotency keys)]
    KAFKA[(Kafka Event Backbone)]

    WORK[Work Orchestration Service\n- create_work_item(STOCK_COUNT)\n- create_work_item(PURCHASE_APPROVAL)\n- approval_routes / work_item_approval_steps]
    FORM[Forms & Workflow Builder\n- count TemplateVersion]
    EVID[Evidence Service]
    ISSUE[Issues & CAPA Service]
    ORG[Identity/Org Service]
    INTEG[Integration Hub\n- ERP/POS connectors]
    ANALYTICS[Analytics Platform]

    MOB --> GW
    WEB --> GW
    GW --> API
    API --> CAT
    API --> CNT
    API --> PO
    API --> RCV
    API --> SUP

    CNT --> VAR
    VAR --> PG
    CNT -->|create_work_item type=STOCK_COUNT| WORK
    CNT -->|template ref| FORM
    CNT -->|line evidence ref| EVID

    PO --> ROUTE
    ROUTE -->|resolve ApprovalRoute by total_cost| WORK
    PO -->|create_work_item type=PURCHASE_APPROVAL,\n approval_route_id=resolved| WORK
    PO --> PG

    RCV --> PG
    RCV -->|supplier quality issue| ISSUE
    SUP --> PG
    SCORE --> PG
    SCORE -->|consumes issue.created/resolved| ISSUE
    ALERT --> PG
    ALERT -->|low-stock -> draft PO suggestion| PO

    CAT --> PG
    CAT --> REDIS
    CAT <-->|SKU/supplier sync| INTEG
    PO -->|PO transmission| INTEG
    RCV -->|invoice/receipt data| INTEG

    OUT --> KAFKA
    KAFKA --> ANALYTICS
    KAFKA -.->|approval.step_decided,\ntask.completed/rejected| PO
    ORG -.->|UserScope, OrganizationNode| API
```

The dotted edge from Kafka back into `PO` represents this service **consuming** Work
Orchestration's approval-outcome events, not producing its own competing approval events — the PO
manager is a state-machine follower of Work Orchestration's authoritative approval-step record for
the duration of `PENDING_APPROVAL`.

## 3. Data Ownership

| Entity | Owned here | Referenced from |
|---|---|---|
| `SKU`, `SKUCategory`, `SKUSupplierLink`, `SKULocationThreshold`, `SKULocationStock` | Yes | Analytics (CDC), Integrations (ERP sync target) |
| `StockCount`, `StockCountLine` | Yes (business behavior); WorkItem envelope (schedule, assignment, offline sync, SLA) owned by Work Orchestration | Work Orchestration (`work_items.id` FK) |
| `PurchaseOrder`, `PurchaseOrderLine` | Yes, fully, including lifecycle | Work Orchestration (referenced via `approval_work_item_id` only during `PENDING_APPROVAL`) |
| `Supplier`, `SupplierScorecard` | Yes | Issues & CAPA (Issue carries `supplier_id` reference, not owned here) |
| `GoodsReceipt`, `GoodsReceiptLine` | Yes | Integrations (invoice data feed) |
| `ApprovalRoute`, `WorkItemApprovalStep` | **No — owned by Work Orchestration.** This module only stores an FK to a `WorkItem` whose approval chain lives entirely in Work Orchestration's schema. | N/A — reused by reference |
| `WorkItem` (STOCK_COUNT, PURCHASE_APPROVAL rows) | No — owned by Work Orchestration | Referenced by ID from `stock_counts.work_item_id` and `purchase_orders.approval_work_item_id` |
| `Evidence` | No — owned by Evidence Service | Referenced by ID from count lines / receipt lines |
| `Issue` | No — owned by Issues & CAPA | Referenced by `supplier_id`/`po_id` in `source_ref` |

## 4. External Interfaces

### 4.1 Calls this service makes (outbound)

- `POST /api/v1/work-items` (Work Orchestration) with `type=STOCK_COUNT` — creates the count
  WorkItem for scheduling/assignment/offline sync.
- `POST /api/v1/work-items` (Work Orchestration) with `type=PURCHASE_APPROVAL`,
  `subject_ref={entity_type: "purchase_order", entity_id}`, and `approval_route_id` resolved by
  the Cost-Threshold Route Resolver — this is the sole integration point with the approval engine;
  this service never writes to `approval_routes` or `work_item_approval_steps` directly.
- `GET /api/v1/work-items/{id}/approval-steps` (Work Orchestration) — read-through when rendering
  the PO detail's `approval_chain` block, so the shape shown to the user is always the live
  Work Orchestration record, never a cached/duplicated copy that could drift.
- `POST /internal/issues` or the public Issues & CAPA create endpoint — for supplier-quality Issue
  creation from a receipt/count.
- Integration Hub connector calls for ERP/POS SKU sync, PO transmission, invoice ingestion.

### 4.2 Events consumed (inbound)

- `approval.step_decided` (Work Orchestration, scoped to `PURCHASE_APPROVAL`-type WorkItems) —
  advances/rejects the owning PO.
- `task.completed` / `task.rejected` (Work Orchestration, terminal outcome of the
  `PURCHASE_APPROVAL` WorkItem) — sets `PurchaseOrder.status = APPROVED` or back to `DRAFT`.
- `task.completed` (Work Orchestration, `STOCK_COUNT` WorkItem reaching `VERIFIED`) — triggers final
  stock adjustment if not already applied synchronously.
- `issue.created` / `issue.resolved` (Issues & CAPA, filtered to Issues with a `supplier_id`
  reference) — feeds `SupplierScorecard` quality metrics.
- `organization.node_changed` (Identity/Org) — used by the Route Resolver only insofar as
  `ROLE_AT_LOCATION`/`ROLE_AT_PARENT_NODE` approver-type resolution is itself Work Orchestration's
  concern; this service does not re-resolve org changes itself.

### 4.3 Events produced (outbound)

See functional-spec.md §12 for the full list (`stock_count.*`, `purchase_order.*`,
`goods_receipt.*`, `supplier.scorecard_updated`).

## 5. Key Design Decisions and Trade-offs

- **PurchaseOrder is not a WorkItem; PURCHASE_APPROVAL is.** A PO's full lifecycle
  (`DRAFT/PENDING_APPROVAL/APPROVED/SENT_TO_SUPPLIER/PARTIALLY_RECEIVED/RECEIVED/CLOSED`) does not
  fit the generic WorkItem state machine, which has no post-approval fulfillment states. Rather than
  distort the WorkItem model to accommodate procurement-specific states (which would leak
  domain-specific concerns into a shared platform primitive every other module relies on), this
  module keeps its own PO aggregate and uses a WorkItem **only for the approval sub-flow**, exactly
  the pattern the addendum specifies ("PURCHASE_APPROVAL WorkItem type ... reuses the multi-level
  Approval Routing"). This is the same design shape Asset Maintenance (Module 16) uses for
  `MaintenanceWorkOrder` (a full WorkItem there, because a maintenance job's lifecycle *does* map
  onto the canonical states) — the two modules make different, individually justified choices about
  how much of their domain rides on the WorkItem chassis versus a bespoke aggregate.
- **Stock count is a WorkItem, fully.** Unlike a PO, a stock count's lifecycle (scheduled, assigned,
  in progress, submitted, optionally reviewed, verified) maps cleanly onto the canonical WorkItem
  state machine with no gaps, so it is a full `STOCK_COUNT` WorkItem, template-backed by Forms &
  Workflow Builder, per the addendum's explicit guidance.
- **Approval reuse over a parallel engine — consistency and a single audit trail.** Building a
  second approval engine for POs would (a) duplicate a materially complex, already-hardened
  mechanism (sequential steps, self-approval prevention, delegation, per-step SLA/escalation,
  re-resolution on reorg), (b) fragment the audit trail an auditor needs to answer "who approved
  this, when, and was anyone escalated" across two different schemas depending on approval type,
  and (c) fragment the approval UX into two different components a manager has to learn. Reuse
  keeps exactly one place in the platform where "what does multi-level approval mean" is defined,
  matching P10 (Design for change) — a future change to approval semantics (e.g. adding quorum
  approval) benefits every approval type at once, including PO approval, with zero change to this
  module.
- **Cost-threshold table, not a hardcoded tier.** The threshold-to-route mapping is data
  (`po_approval_thresholds`), resolved at submission time, so a tenant can restructure its spend
  authority matrix without an engineering change — consistent with P10 and the functional spec's
  explicit resolved decision (functional-spec.md §13).
- **`sku_location_stock` as a materialized on-hand table, not a derived read.** On-hand quantity is
  mutated by exactly two code paths (stock-count verification, goods-receipt line save) through one
  shared idempotent adjustment function (LLD §8), rather than computed on read from a full ledger
  scan — necessary for low-stock alert sweep performance at tens of thousands of SKUs per tenant.

## 6. Scalability

- **Catalog reads are the hottest path** (PO line entry, count line lookup, barcode scan) — served
  from Redis-cached SKU/supplier-link lookups keyed `tenant_id:sku_id`/`tenant_id:barcode`,
  invalidated on SKU write, falling back to Postgres on cache miss.
- **Stock-count line writes are high-fan-out** during a scheduled full-count wave across thousands
  of locations simultaneously (e.g. all stores start a Monday morning cycle count) — line upserts
  are partitioned by `tenant_id` and batched per WorkItem to avoid hot-row contention on a shared
  aggregate row; no single `stock_counts` row is written by more than the count's own assignee(s) at
  a time (concurrency detail in LLD §9).
- **PO and supplier data are comparatively low-volume, high-value writes** — no special sharding
  needed beyond standard `tenant_id` partitioning; read replicas serve the supplier scorecard
  dashboard and PO listing to keep transactional write paths uncontended.
- **Scorecard rollup is a background aggregation job** (LLD §10), not a live query over raw receipt/
  issue history on every dashboard load, so dashboard read latency stays flat regardless of a
  tenant's historical PO volume.

## 7. Multi-Tenancy

Every table (`skus`, `sku_categories`, `sku_supplier_links`, `sku_location_stock`, `stock_counts`,
`stock_count_lines`, `purchase_orders`, `purchase_order_lines`, `suppliers`,
`supplier_scorecards`, `goods_receipts`, `goods_receipt_lines`, `po_approval_thresholds`) carries
`tenant_id NOT NULL` with Postgres RLS enforcing `tenant_id = current_setting('app.tenant_id')`
behind application-layer scoping, per `conventions.md`. Barcode-based SKU lookup is scoped
`(tenant_id, barcode)`, never a bare global barcode index, so two tenants selling the same
manufacturer product never collide. `location_id` scoping is a second enforcement axis via
`UserScope` exactly as in every other module — a Store Manager's PO-create scope is bounded to
their assigned `OrganizationNode` subtree.

## 8. Failure Modes and Degradation

| Failure | Degradation strategy |
|---|---|
| Work Orchestration unavailable when creating a `PURCHASE_APPROVAL` WorkItem | PO remains `DRAFT`; submission is retried with backoff (Idempotency-Key on the create call ensures a retry never creates two approval WorkItems for the same PO); user sees "submitting for approval" pending state, not a hard failure, up to a bounded retry window, after which the submission action is surfaced as failed and re-triggerable. |
| Work Orchestration event stream lagging (approval.step_decided delayed) | PO stays visibly `PENDING_APPROVAL` with its last-known step; a manual "refresh approval status" action performs a synchronous read-through to `GET /work-items/{id}/approval-steps` so the UI is never stuck stale for longer than the operator is willing to wait, independent of eventual event delivery. |
| Integration Hub (ERP/POS) unavailable | SKU catalog and supplier records continue to be manageable manually; PO transmission to supplier degrades to a manual "mark as sent" confirmation instead of automated EDI/email dispatch; goods-receipt entry is unaffected (manual entry always available). |
| Evidence Service unavailable during count-line photo capture | Count line still saves with a pending-evidence-upload marker (mirrors platform-wide offline evidence upload semantics, source-spec §9.2); submission is not blocked on evidence upload completion, only on evidence *reference* being recorded. |
| Redis catalog cache unavailable | Falls back to direct Postgres reads for SKU/barcode lookup; degraded latency, not degraded correctness. |
| Scorecard rollup job failure | Dashboard serves the last successfully computed scorecard with a visible `as_of` staleness indicator rather than blocking or showing a zeroed/broken card. |
| Duplicate-PO check service degraded | Falls open (skips the non-blocking warning) rather than blocking PO creation — the check is an advisory, not a correctness guarantee, so it never becomes a hard dependency for the create path. |

## 9. Technology Choices (module-specific, justified)

- **Postgres for all transactional inventory/procurement state** — per `conventions.md`, no
  alternative datastore is justified here; PO/SKU/supplier volumes do not approach the scale that
  would justify a specialized store (contrast Module 17 IoT sensor readings, which do justify
  time-series partitioning).
- **Redis for catalog lookup caching** — the SKU/barcode lookup path is read-heavy and
  latency-sensitive (mobile scan-to-line-jump during a count), a textbook cache-aside use case
  already standard in the platform's Redis usage (permission cache, idempotency keys).
- **No new workflow engine.** Approval progression is entirely Work Orchestration's Temporal-backed
  mechanism; this module does not stand up its own Temporal workflows for PO approval. It may
  optionally use Temporal for the **low-stock alert sweep** and **scorecard rollup** background jobs
  (LLD §10) since those are periodic, retryable, at-least-once jobs that benefit from the platform's
  existing durable-workflow infrastructure rather than a bespoke cron implementation.
- **Kafka event backbone, transactional outbox** — identical pattern to every other module (e.g.
  Issues & CAPA), for exactly the reasons already established: never lose or duplicate an event
  relative to its owning transactional write.
