# Inventory, Procurement & Supplier Management — Functional Specification

Module corresponds to **new-modules-addendum.md row 19** (`19-inventory-procurement`, Tier 2
competitive gap closure). Source: `_context/source-spec.md` (overall platform PRD/FSD baseline —
this module is not in the original v2.0 baseline), `_context/conventions.md` (tech stack, domain
model, API/event conventions, state machines — reused verbatim), `_context/new-modules-addendum.md`
(module map row 19: primary new entities `SKU`, `StockCount`, `StockCountLine`, `PurchaseOrder`,
`PurchaseOrderLine`, `Supplier`, `SupplierScorecard`; key integrations Forms & Workflow Builder,
Work Orchestration, Integrations, Analytics). Shortcode: **INV**. Requirement IDs are
`FR-INV-001`...

This module reuses, and does **not** reimplement, the multi-level Approval Routing mechanism built
for Work Orchestration (`02-work-orchestration/functional-spec.md` §6 "Approval Routing" and
`02-work-orchestration/low-level-design.md` §1.1 `approval_routes` / `work_item_approval_steps`).
Every purchase-order approval chain in this spec — e.g. **Store Manager → District Manager →
Finance** for a purchase order above a cost threshold — is the *same* `ApprovalRoute` /
`WorkItemApprovalStep` schema and state semantics used by Audit/Inspection approvals elsewhere in
the platform, instantiated against a new `PURCHASE_APPROVAL` `WorkItem` type. This spec does not
define a second approval engine, a second escalation ladder, or a second "who approves next" rule
set.

## 1. Purpose and Scope

### 1.1 Purpose

Inventory, Procurement & Supplier Management closes a Tier-2 competitive gap (parity with
Wooqer/Zenput/Operandio-class inventory add-ons) by giving frontline operations a governed system
of record for what stock exists, what was actually counted against what was expected, what needs to
be reordered, who approved the spend, which supplier fulfilled it, and whether that supplier is
performing. It extends the core product loop (`PLAN -> EXECUTE -> EVIDENCE -> VERIFY -> EXCEPTION
-> CORRECT` per source-spec.md §1) into the inventory/procurement domain: a stock count is executed
and verified like any other frontline work; a variance can become a managed exception; a purchase
requisition is governed by the same approval and audit discipline as any other high-stakes
WorkItem.

### 1.2 In Scope

- **SKU / product catalog** — per-tenant SKU master data, category hierarchy, unit of measure,
  supplier linkage, reorder thresholds, discontinuation lifecycle.
- **Stock count workflow** — cycle counts and full counts as template-backed `WorkItem`s
  (`STOCK_COUNT` type), count-line capture (expected vs. counted quantity), variance calculation and
  flagging, shift-handoff continuation.
- **Purchase requisition / purchase order (PO) creation** — line items, cost-threshold-driven
  approval-route resolution, PO lifecycle from `DRAFT` through `CLOSED`, duplicate-PO prevention.
- **Supplier record management and scorecarding** — supplier master data, on-time delivery rate,
  fulfillment accuracy, quality-issue linkage from Issues & CAPA, computed scorecards.
- **Receiving / goods-receipt workflow** — reconciling a delivery against its PO, partial-delivery
  handling, price/quantity variance flagging.
- UX states for stock-count-in-progress, PO-pending-approval, low-stock alerts, and the supplier
  scorecard dashboard.
- `stock_count.*` / `purchase_order.*` / `goods_receipt.*` / `supplier.*` domain events (see §12).

### 1.3 Out of Scope (owned by other modules, referenced here)

- **Approval routing / multi-level approval chains.** Entirely owned by **Work Orchestration
  (EPIC-03)**. This module never stores its own "who approves next" logic, its own escalation
  ladder, or its own step state machine. A `PurchaseOrder` above a configured cost threshold gets a
  `PURCHASE_APPROVAL` `WorkItem` with an `approval_route_id` resolved from Work Orchestration's
  `approval_routes` table, and this module tracks that WorkItem's `id`/`status` by reference. See
  §7 and dependencies (§13).
- **Stock-count template design** (which fields, matrix layout, conditional logic, calculated
  variance field, photo-evidence requirement per line) — owned by **Forms & Workflow Builder
  (EPIC-04)**. This module is the functional owner of the `STOCK_COUNT` `WorkItem` type's business
  behavior (what a count *means*, how variance is computed and stored) but the count's on-screen
  form is a `TemplateVersion` like any other.
- **WorkItem scheduling, assignment, offline sync, SLA/escalation mechanics** for `STOCK_COUNT` and
  `PURCHASE_APPROVAL` WorkItems — **Work Orchestration (EPIC-03)**. This module creates these
  WorkItems via Work Orchestration's `create_work_item` API and reacts to their events; it does not
  own their state machine.
- **Evidence capture/storage** (photos of damaged goods, count-sheet scans, delivery signatures) —
  **Evidence module (EPIC-06)**. This module references `Evidence` by ID.
- **Quality issues arising from a supplier defect** (e.g. contaminated ingredient, defective part) —
  **Issues & CAPA (EPIC-07)** owns the `Issue` lifecycle; this module creates the Issue with a
  `supplier_id` reference and consumes `issue.created`/`issue.resolved` to feed the
  `SupplierScorecard`, but does not own root-cause/CAPA mechanics.
- **ERP/POS synchronization mechanics** (connector config, retries, field mapping) — **Integrations
  & Extensibility (EPIC-13)**. This module is a data owner that the Integration Hub reads from and
  writes to; it does not implement connector infrastructure.
- **Shrinkage/stock-out KPI rollups, executive dashboards** — **Analytics & Control Tower
  (EPIC-12/15)**, consuming this module's events and a CDC feed.
- **Vendor/technician dispatch for physical asset repair** — distinct from this module's `Supplier`
  (a goods vendor); see Module 16 (Asset Maintenance) `Vendor`/`Technician` and Module 25 (Field
  Service) for service vendors. A `Supplier` in this module is a goods/inventory supplier, not a
  service provider, though a tenant may map the same legal entity to both records.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Frontline Associate | Performs stock counts (cycle/full) on mobile, scans barcodes, records counted quantities, flags visible damage/shrinkage, hands off an in-progress count at shift change. |
| Store/Site Manager | Creates purchase requisitions, is the first-level approver for mid-tier spend, reviews low-stock alerts, receives deliveries, reconciles goods receipt against PO, resolves count variances at their location. |
| District/Regional Manager | Second-level approver for higher-cost purchase orders per the reused Work Orchestration approval route; monitors stock-out/shrinkage trends across territory. |
| Finance | Final-level approver for high-cost purchase orders (per tenant-configured route); reviews price-variance exceptions between PO and invoice. |
| Operations Admin | Manages the SKU catalog (create/edit/discontinue SKUs, category hierarchy, unit of measure), manages supplier records, configures cost-threshold-to-approval-route mapping, configures stock-count templates (with Forms & Workflow Builder), reviews supplier scorecards. |
| AI Agent (Resolution Agent, future) | May propose a reorder purchase requisition when a low-stock alert and historical velocity indicate stock-out risk (L1 Recommend by default per source-spec §20.3); never auto-approves a PO — approval always flows through the reused Work Orchestration mechanism and Human-in-the-Loop policy (source-spec §13.4). |

## 3. SKU / Product Catalog Management

- **FR-INV-001 — SKU identity.** A `SKU` is scoped to `tenant_id` and identified by a tenant-unique
  `sku_code` (free text, typically the tenant's own item code) plus an optional `barcode`/`upc` for
  scan lookup. Required fields at creation: `name`, `category_id`, `unit_of_measure`, `status`
  (`ACTIVE`, `DISCONTINUED`, `DRAFT`). A SKU may optionally carry `case_pack_size` (units per case,
  for count/order conversion) and `default_supplier_id`.
- **FR-INV-002 — Category hierarchy.** Categories form an arbitrary-depth tree per tenant
  (`sku_categories`, parent-referencing, mirroring the `OrganizationNode` arbitrary-depth pattern
  from Identity/Org, but scoped to product taxonomy, not location). A SKU belongs to exactly one
  leaf category; reporting rolls up through ancestor categories.
- **FR-INV-003 — Unit of measure.** Each SKU has a canonical `unit_of_measure` (`EACH`, `CASE`, `KG`,
  `LB`, `L`, `GAL`, or a tenant-defined custom unit) used for stock counts and PO line quantities. A
  SKU may declare one or more **unit conversions** (e.g. `1 CASE = 24 EACH`) so a count can be taken
  in cases while inventory value is tracked in eaches, or vice versa. Conversion is applied at write
  time and the base-unit quantity is always the value stored for variance math (FR-INV-024).
- **FR-INV-004 — Supplier linkage.** A SKU may reference one or more suppliers via
  `sku_supplier_links` (many-to-many, with `supplier_sku_code`, `unit_cost`, `is_preferred`,
  `lead_time_days`). A SKU with no linked supplier cannot be added to a PO line (FR-INV-032) but can
  still be counted.
- **FR-INV-005 — Reorder thresholds.** A SKU may carry a per-location `reorder_point` and
  `reorder_quantity` (`sku_location_thresholds`, one row per `(sku_id, location_id)` since reorder
  behavior varies by site velocity). These feed the low-stock alert sweep (§9, LLD background job).
- **FR-INV-006 — Location-scoped availability.** A SKU is catalog-wide by default but may be
  restricted to a subset of `OrganizationNode` subtrees (`sku_location_scope`) — e.g. a regional
  promotional item not stocked enterprise-wide. Stock counts and POs at an out-of-scope location
  cannot reference that SKU.
- **FR-INV-007 — Discontinuation.** Setting `status = DISCONTINUED` blocks the SKU from new PO
  lines and new stock-count templates going forward but does **not** retroactively alter historical
  `StockCountLine`/`PurchaseOrderLine` rows that already reference it (historical-truth principle,
  source-spec P7). See §10 edge case "SKU discontinued mid-cycle."
- **FR-INV-008 — Bulk import/edit.** Operations Admin can bulk-create/update SKUs via CSV upload
  (validated row-by-row, partial success reported per row with error codes) or via the ERP sync path
  (Integrations, §13).
- **FR-INV-009 — Audit trail.** Every SKU field change is recorded to `AuditLog` (source-spec §17.1)
  with before/after values; SKU cost changes specifically are also retained in `sku_cost_history` for
  price-variance comparison at receiving time (§8).

## 4. Stock Count Workflow

- **FR-INV-020 — Stock count as a WorkItem.** A stock count is a `STOCK_COUNT`-type `WorkItem`
  (per `new-modules-addendum.md`), created either from a Work Orchestration `schedule` (recurring
  cycle counts, e.g. weekly per category) or ad hoc by a Store Manager. It is **template-backed**:
  `WorkItem.template_version_id` references a Forms & Workflow Builder `TemplateVersion` whose
  schema defines which SKUs/categories are in scope for that count and what evidence (photo of
  shelf, damage note) is required per line.
- **FR-INV-021 — Count types.** `count_type ∈ {CYCLE, FULL, SPOT}`. `CYCLE` counts a configured
  subset of SKUs/categories on a rotation; `FULL` counts the entire in-scope catalog for a location;
  `SPOT` is an ad hoc, unscheduled count of one or a few SKUs (e.g. triggered by a suspected
  shrinkage event or a low-stock alert investigation).
- **FR-INV-022 — Count lines and expected quantity.** On `STOCK_COUNT` WorkItem creation, one
  `StockCountLine` row is materialized per in-scope SKU, pre-populated with `expected_quantity`
  (the system's current on-hand estimate for that SKU/location, derived from the last closed count
  plus net PO receipts and any POS/ERP-sync depletion feed since then — see Resolved Decisions §13). The
  associate enters `counted_quantity`; `variance_quantity = counted_quantity - expected_quantity`
  and `variance_value = variance_quantity * unit_cost` are computed at line-submit time (LLD
  pseudocode §8).
- **FR-INV-023 — Barcode-driven entry.** The mobile count UI supports scanning a SKU's barcode to
  jump directly to its count line (reusing the platform's barcode/QR form component per
  source-spec §8.1) rather than requiring manual list scroll — critical for counts with hundreds of
  SKUs.
- **FR-INV-024 — Variance threshold flagging.** A tenant-configured `variance_flag_threshold_pct`
  (default 5%) and/or `variance_flag_threshold_value` (default $50, tenant currency) determines
  whether a line is flagged `NEEDS_REVIEW`. A flagged line requires either a `variance_reason` free
  text or a photo evidence reference before the count can be submitted (mirrors Forms &
  Workflow Builder's conditional-mandatory-field pattern, source-spec §8.2).
- **FR-INV-025 — Running variance total.** While a count is `IN_PROGRESS`, the mobile UX shows a
  running total: lines counted / lines remaining, and a running net variance value, so the associate
  (or the manager watching remotely) can see how the count is trending before submission (§6 UX
  states).
- **FR-INV-026 — Shift handoff (started by one user, completed by another).** A `STOCK_COUNT`
  WorkItem's `assignee` can be reassigned mid-count without losing entered line data — count lines
  are persisted incrementally (not only on final submit) keyed to the WorkItem, not the assignee.
  `stock_count_lines.entered_by`/`entered_at` is recorded per line, independent of the WorkItem's
  current assignee, so an audit of a submitted count shows exactly which associate entered which
  line even when the count spans a shift change. See §10 edge case.
- **FR-INV-027 — Submission and WorkItem state.** Submitting a count transitions the `STOCK_COUNT`
  WorkItem `IN_PROGRESS -> SUBMITTED` (canonical WorkItem state machine, unchanged). If the tenant
  requires manager verification of counts with `NEEDS_REVIEW` lines, the WorkItem additionally
  enters `PENDING_REVIEW` gated on a **single-step** `ApprovalRoute` (Store Manager) — reusing the
  same mechanism as §7, not a separate review flow. Counts with no flagged lines auto-transition
  `SUBMITTED -> VERIFIED` (no manager step required, tenant-configurable).
- **FR-INV-028 — Post-count inventory adjustment.** On `VERIFIED`, each line's `counted_quantity`
  becomes the new authoritative on-hand quantity for that SKU/location (`sku_location_stock`
  upsert), and a `stock_count.completed` event carries the full line-level variance payload for
  Analytics (shrinkage KPI) and for the low-stock alert sweep to re-evaluate thresholds
  immediately rather than waiting for its next scheduled run.

## 5. Purchase Requisition / Purchase Order Creation

- **FR-INV-030 — PO as a first-class module entity.** Unlike a stock count, a `PurchaseOrder` is
  **not itself** a `WorkItem` — it is this module's own aggregate (owning its own lifecycle state
  machine, §7/§11) because a PO has domain-specific states (`SENT_TO_SUPPLIER`,
  `PARTIALLY_RECEIVED`) that do not map onto the generic WorkItem state machine. What *is* a
  WorkItem is the **approval step** of that lifecycle: when a PO needs approval, this module creates
  a `PURCHASE_APPROVAL`-type `WorkItem` referencing the PO (`subject_ref = {entity_type:
  "purchase_order", entity_id: <po_id>}`, the same `subject_ref` shape Work Orchestration already
  uses for generic `Approval`-type WorkItems). The PO's own status mirrors that WorkItem's outcome
  (§7) but the PO record, its line items, and its post-approval lifecycle remain entirely owned
  here.
- **FR-INV-031 — Creation and line items.** A Store/Site Manager (or Operations Admin, or the
  low-stock alert sweep proposing a draft) creates a `PurchaseOrder` with `location_id`,
  `supplier_id`, and one or more `PurchaseOrderLine` rows (`sku_id`, `quantity`, `unit_of_measure`,
  `unit_cost` defaulted from `sku_supplier_links.unit_cost` but editable, `line_total` computed).
  `PurchaseOrder.total_cost` is the sum of line totals; currency is the tenant's configured currency
  (multi-currency per-supplier is out of scope for this version — see Resolved Decisions §13).
- **FR-INV-032 — Supplier-SKU validation.** Every line's `sku_id` must have an active
  `sku_supplier_links` row for the PO's `supplier_id`, and the SKU must not be `DISCONTINUED`
  (unless the PO is being created to fulfill an already-approved backorder against a SKU that was
  discontinued after ordering — see §10 edge case). Violation returns `422
  SKU_NOT_LINKED_TO_SUPPLIER` or `422 SKU_DISCONTINUED`.
- **FR-INV-033 — Cost-threshold-driven approval routing.** On submission from `DRAFT`, the PO's
  `total_cost` is evaluated against a tenant-configured **cost-threshold-to-route table**
  (`po_approval_thresholds`, LLD §1) to resolve which `ApprovalRoute` (if any) applies:
  - Below the lowest threshold (tenant default: **under $500**): **auto-approved**, no
    `PURCHASE_APPROVAL` WorkItem is created; PO goes `DRAFT -> APPROVED` immediately, fully
    audited (`approved_by = SYSTEM_AUTO`, `approval_reason = "under threshold"`).
  - Mid-range (tenant default: **$500–$5,000**): resolves to a **single-step** ApprovalRoute
    (e.g. Store Manager or District Manager depending on tenant configuration).
  - High (tenant default: **over $5,000**): resolves to a **multi-level** ApprovalRoute (e.g.
    **Store Manager → District Manager → Finance**), using the exact sequential
    `WorkItemApprovalStep` mechanism from Work Orchestration §6 — same step statuses
    (`PENDING/ACTIVE/APPROVED/REJECTED/SKIPPED/ESCALATED`), same self-approval prevention, same
    delegation, same per-step SLA/escalation, same "request more evidence" semantics
    (Work Orchestration FR-WORK-050 through FR-WORK-059, all reused verbatim, not reimplemented).
  See LLD §7 pseudocode for the resolution algorithm.
- **FR-INV-034 — PO detail exposes the same approval-chain shape.** `GET
  /api/v1/purchase-orders/{id}` returns an `approval_chain` block whose shape matches Work
  Orchestration's `GET /api/v1/work-items/{id}/approval-steps` response one-for-one (same field
  names: `step_index`, `approver_type`, `resolved_approver_user_id`, `status`, `decision`,
  `decision_by`, `decision_at`, `comment`) so the web/mobile approval UI can render it with the
  same component used for every other multi-level approval in the platform, not a bespoke PO
  approval widget. See LLD §4 for the exact response example.
- **FR-INV-035 — Approval decisions delegate to Work Orchestration.** An approver acts via Work
  Orchestration's own `POST /api/v1/work-items/{id}/approval-steps/{step_index}/decision` endpoint
  (not a parallel `/purchase-orders/.../decision` endpoint) against the `PURCHASE_APPROVAL`
  WorkItem's `work_item_id`. This module subscribes to Work Orchestration's `approval.step_decided`
  and `task.completed`/`task.rejected`-class events on that WorkItem to progress the PO's own status
  (`PENDING_APPROVAL -> APPROVED` on final-step approval; `PENDING_APPROVAL -> REJECTED` -> back to
  `DRAFT` on any-step rejection), per Work Orchestration FR-WORK-053/054. This module never polls;
  it reacts to the event.
- **FR-INV-036 — Duplicate PO prevention.** Creating a PO for the same `(tenant_id, location_id,
  supplier_id)` with a line-item set that is a superset/exact-match of an existing **non-terminal**
  PO (`DRAFT`, `PENDING_APPROVAL`, `APPROVED`, `SENT_TO_SUPPLIER`) created within a configurable
  lookback window (default 24 hours) surfaces a **non-blocking warning**
  (`POSSIBLE_DUPLICATE_PO`, referencing the candidate PO id) rather than a hard reject — a manager
  may legitimately need two same-day orders from the same supplier. Idempotency-Key on the create
  endpoint separately prevents true double-submission from a flaky client retry (platform
  convention, distinct concern from business-level duplicate detection).
- **FR-INV-037 — Sending to supplier.** On `APPROVED`, the PO can be transitioned to
  `SENT_TO_SUPPLIER` either automatically (tenant-configured, via the Integration Hub's supplier
  EDI/email/portal channel) or manually by the creating manager confirming out-of-band
  communication. A `purchase_order.sent` event is emitted either way.
- **FR-INV-038 — Line-item edits after submission.** Once a PO leaves `DRAFT`, line items are
  immutable except via a formal **PO revision**: editing a `PENDING_APPROVAL` or `APPROVED` PO's
  lines creates a new PO `version` (optimistic-concurrency counter, platform convention) and, if the
  revised `total_cost` crosses into a different threshold tier, **restarts approval routing from
  scratch** against the new tier's route (a fresh `PURCHASE_APPROVAL` WorkItem; the prior one, if
  any, is cancelled with reason `SUPERSEDED_BY_REVISION`).

## 6. Supplier Management and Scorecarding

- **FR-INV-040 — Supplier record.** A `Supplier` is scoped to `tenant_id` with `name`, `contact`
  fields, `status` (`ACTIVE`, `SUSPENDED`, `RETIRED`), `payment_terms`, and an optional
  `integration_ref` (external ERP vendor ID for sync, §13).
  Operations Admin owns creation/edit; suspension blocks new POs against that supplier but does not
  affect in-flight ones.
- **FR-INV-041 — Scorecard metrics.** `SupplierScorecard` is a rolling, periodically recomputed
  aggregate (default monthly, plus trailing 90-day) per `(tenant_id, supplier_id[, location_id])`
  with at minimum:
  - `on_time_delivery_rate` — goods receipts where `actual_delivery_date <= po_promised_date`
    divided by total receipts in period.
  - `fulfillment_accuracy_rate` — receipt lines with `received_quantity == ordered_quantity` AND
    no price variance, divided by total receipt lines.
  - `quality_issue_count` / `quality_issue_rate` — count of Issues (Issues & CAPA) with
    `source_ref.supplier_id` matching, divided by total POs/receipts in period.
  - `average_lead_time_days` — actual vs. `sku_supplier_links.lead_time_days` commitment.
  - `open_issue_count` — currently open quality Issues linked to this supplier.
- **FR-INV-042 — Quality issue linkage.** When a goods receipt (§8) or a stock count reveals a
  supplier-attributable quality problem (damaged/expired/wrong-item delivery), the receiving
  manager can create an Issue (Issues & CAPA, `category = SUPPLIER_QUALITY`) directly from the
  receipt/count screen, pre-populated with `supplier_id`, `po_id`, `sku_id`. This module listens for
  `issue.created`/`issue.resolved` on Issues carrying a `supplier_id` reference to feed
  `quality_issue_count`/`open_issue_count` — it does not duplicate Issue lifecycle state.
- **FR-INV-043 — Scorecard dashboard.** Operations Admin and District/Regional Manager personas see
  a ranked supplier scorecard dashboard (§6 UX states) filterable by location/region and time
  window, with drill-down to the underlying POs/receipts/issues behind any metric (source-spec P8
  Explainability — every score exposes its drivers).
- **FR-INV-044 — Preferred supplier and cost impact on routing.** `sku_supplier_links.is_preferred`
  and the scorecard's `on_time_delivery_rate` are surfaced (not enforced) when a manager picks a
  supplier for a new PO line — a below-threshold supplier does not block PO creation but the UI
  surfaces a soft warning, since blocking a legitimate emergency reorder would violate the
  mobile-first "fast action" principle (source-spec P1).

## 7. Receiving / Goods-Receipt Workflow

- **FR-INV-050 — Goods receipt creation.** A `GoodsReceipt` is created against exactly one
  `PurchaseOrder` (`SENT_TO_SUPPLIER`, `PARTIALLY_RECEIVED`, or `APPROVED` in the case of a
  will-call pickup that skips explicit "sent" confirmation) when a delivery physically arrives. Each
  `GoodsReceiptLine` records `received_quantity`, `received_unit_cost` (from the supplier invoice or
  packing slip), and optional `condition_note`/evidence reference.
- **FR-INV-051 — Partial delivery.** `received_quantity` per line may be less than
  `PurchaseOrderLine.quantity`. When any line is under-received, the PO transitions
  `SENT_TO_SUPPLIER -> PARTIALLY_RECEIVED` rather than `RECEIVED`; the outstanding quantity remains
  visible as a `backorder_quantity` per line, and a subsequent `GoodsReceipt` against the same PO is
  permitted to complete it. A PO reaches `RECEIVED` only when the cumulative received quantity
  across all receipts meets or exceeds every line's ordered quantity (over-receipt handling below).
- **FR-INV-052 — Price variance flagging.** If `received_unit_cost` differs from the PO line's
  `unit_cost` by more than a tenant-configured `price_variance_flag_pct` (default 3%), the receipt
  line is flagged `PRICE_VARIANCE` and routed to a manager review queue (not a second approval
  chain — a simple acknowledgment/dispute action on the receipt, since the spend was already
  approved at PO creation; a materially different total cost that would have changed the approval
  tier at PO time is instead handled as a PO revision per FR-INV-038 before the receipt is
  finalized).
- **FR-INV-053 — Over-receipt.** `received_quantity` greater than ordered quantity is permitted but
  flagged `OVER_RECEIPT`; it does not block receipt completion but requires a `condition_note`
  (e.g. "supplier included promotional extra units") for audit clarity.
- **FR-INV-054 — Stock update.** On receipt-line save, `sku_location_stock.on_hand_quantity` for
  that `(sku_id, location_id)` is incremented by `received_quantity` (base-unit-converted per
  FR-INV-003) — this is the second of the two paths (alongside stock-count verification,
  FR-INV-028) that mutate on-hand stock; both go through the same idempotent stock-adjustment
  function (LLD §8) to avoid divergent update logic.
- **FR-INV-055 — Closing a PO.** Once `RECEIVED` (fully reconciled) and any `PRICE_VARIANCE`/
  `OVER_RECEIPT` flags are acknowledged, a manager or scheduled job transitions the PO to `CLOSED`
  (terminal). A `CLOSED` PO is immutable except for supplemental notes; it remains queryable for
  audit and scorecard computation indefinitely per tenant retention policy.

## 8. UX States

- **Stock-count in-progress.** Header shows count name, location, `count_type`; body is a scrollable
  SKU list (or barcode-scan-first entry) with a sticky footer showing `X of Y lines counted` and a
  running net variance value (color-coded neutral/amber/red by magnitude vs. threshold, FR-INV-025).
  A line already flagged `NEEDS_REVIEW` shows an inline badge and blocks submit until its
  `variance_reason`/evidence is attached (FR-INV-024).
- **PO pending approval.** The PO detail screen embeds the same step-progress stepper component
  used by Work Orchestration's generic approval UX (source-spec §6 "Approval Routing", FR-INV-034)
  — e.g. "Step 2 of 3 — awaiting Finance" — rather than a bespoke PO-only progress bar, so approvers
  get one consistent mental model across audit approvals, requisition approvals, and any other
  approval type in the platform.
- **Low-stock alert.** Surfaced in the manager's exceptions feed (mirroring Work Orchestration's
  manager "Exceptions | Approvals | Team | Location health" workspace, source-spec §7.3) as
  `SKU below reorder point at <location>` with one-tap "Create purchase requisition" pre-filled
  with `reorder_quantity` from the SKU's threshold config.
- **Supplier scorecard dashboard.** A ranked table/card view (District/Regional Manager, Operations
  Admin) with on-time %, fulfillment accuracy %, open quality issues, and a trend sparkline per
  supplier; clicking a metric drills into the underlying POs/receipts/issues (FR-INV-043).
- **Empty states.** No SKUs in catalog yet (Operations Admin CTA to import); no counts scheduled for
  this location; no suppliers configured (blocks PO creation with a clear CTA, not a silent dead
  end).
- **Offline.** A stock count started offline (mobile) queues line entries in the local offline store
  exactly like any other WorkItem submission (source-spec §9.2/§19) — count-line writes are
  idempotent by `(work_item_id, sku_id, client_uuid)` so a retried sync never double-counts a line.
  PO creation and approval decisions require connectivity (they are manager/back-office actions, not
  a frontline offline-critical path) but a PO drafted offline (e.g. tablet at a poor-signal
  warehouse) queues as a single idempotent create.
- **Permission-denied.** A user without `inventory:po:create` scope at the target location sees the
  PO-creation entry point disabled with an explanation, consistent with the platform's
  "Effective Access" preview pattern (source-spec §6.2).

## 9. Validation and Edge Cases

| Case | Behavior |
|---|---|
| Partial delivery against a PO | PO -> `PARTIALLY_RECEIVED`; outstanding backorder quantity tracked per line; subsequent receipt permitted (FR-INV-051). |
| Price variance between PO and invoice | Receipt line flagged `PRICE_VARIANCE`, routed to manager acknowledgment queue; does not reopen the original PO approval chain (FR-INV-052). |
| Stock count started by one user, completed by another (shift handoff) | Reassign the `STOCK_COUNT` WorkItem; already-entered lines persist with per-line `entered_by` attribution; submission remains a single WorkItem event (FR-INV-026). |
| Duplicate PO prevention | Non-blocking `POSSIBLE_DUPLICATE_PO` warning on near-identical PO within lookback window; hard idempotency only for literal retry via `Idempotency-Key` (FR-INV-036). |
| SKU discontinued mid-cycle | Historical count/PO lines referencing it are untouched; new PO lines/new count templates cannot add it; an already-`PENDING_APPROVAL` PO referencing a SKU discontinued after submission is allowed to complete approval and receiving (a backorder against a committed order should still be honored) but cannot be revised to add more of that SKU (FR-INV-007/FR-INV-032). |
| PO line item race (two managers editing the same DRAFT PO concurrently) | Optimistic concurrency (`version`/ETag) — second writer gets `409 Conflict` with the current line-item state; no silent overwrite (platform convention). |
| Cost-threshold boundary edit | Editing a `PENDING_APPROVAL` PO's lines such that `total_cost` crosses a threshold tier boundary restarts approval routing against the new tier (FR-INV-038), never silently continues the old chain against a now-mismatched cost. |
| Zero-line PO submission | Blocked at `422 PO_HAS_NO_LINES`; a PO must have at least one line to leave `DRAFT`. |
| Supplier suspended after PO sent | In-flight POs against a suspended supplier are unaffected (already committed); new POs against that supplier are blocked at creation with `422 SUPPLIER_SUSPENDED`. |
| Unit-of-measure mismatch on count vs. catalog | Count line quantity is always captured in the SKU's declared count unit (which may differ from its order unit) and converted to base unit before variance math (FR-INV-003); a template misconfigured with an unconvertible unit is a template-publish-time validation error (Forms & Workflow Builder), not a runtime one. |
| Rejected PO re-submission | On any-step rejection, PO returns to `DRAFT` (not `PENDING_APPROVAL`) with the rejection reason visible, mirroring Work Orchestration FR-WORK-051/054; edits are required before re-submission, which restarts approval routing at step 0. |

## 10. Non-Functional Requirements

- **Count throughput.** A full count of a 2,000-SKU location must remain fluid on mid-tier mobile
  hardware — line list virtualization, barcode-scan jump, and incremental (not batch-only) local
  persistence are required, not optional optimizations.
- **Approval latency parity.** Because PO approval reuses Work Orchestration's mechanism, this
  module inherits its NFRs for approval-step operations (step decision write, notification fan-out)
  rather than defining separate ones — no per-hop slowdown versus a single-approver flow
  (Work Orchestration functional-spec.md §11 NFR reference).
  Note: consult Work Orchestration's own NFR section for the numeric target; this module does not
  restate or diverge from it.
- **Scorecard freshness.** Scorecards recompute at least daily (incremental) and fully monthly;
  dashboard staleness must be visible (`as_of` timestamp on the scorecard view), never presented as
  real-time when it is a batch rollup.
- **Multi-tenancy.** Every table carries `tenant_id`; SKU/supplier/PO data never leaks across
  tenants even via barcode collision (barcodes are only unique within a tenant's catalog, not
  globally).
- **Auditability.** Every stock-count line entry, PO state transition, approval decision (by
  reference into Work Orchestration's own audit trail), and goods-receipt line is attributable to an
  actor and timestamp, consistent with source-spec §21 audit requirements.
- **Scale.** Designed for the platform's stated scale envelope (source-spec §23): a large tenant may
  have tens of thousands of SKUs, hundreds of thousands of stock-count lines/day across locations,
  and a steady stream of POs; schema and indexing (LLD §1) are built for that volume, not a
  single-location pilot.

## 11. Dependencies

- **Work Orchestration (EPIC-03).** Provides the `STOCK_COUNT` and `PURCHASE_APPROVAL` `WorkItem`
  types, their scheduling/assignment/offline-sync/SLA-escalation mechanics, and — critically — the
  **entire multi-level Approval Routing mechanism** (`approval_routes` template table,
  `work_item_approval_steps` per-instance audit table, sequential step progression, delegation,
  self-approval prevention, per-step SLA/escalation, rejection semantics). This module is
  explicit that it consumes this mechanism by reference (`approval_route_id` on the PO points at
  the *same* `approval_routes` row type Work Orchestration owns; there is no
  `inventory_approval_routes` table). See functional §5 (FR-INV-030 through FR-INV-038).
- **Forms & Workflow Builder (EPIC-04).** Owns the `TemplateVersion` schema behind every
  `STOCK_COUNT` WorkItem — field layout, conditional variance-reason requirement, calculated
  variance field definition.
- **Issues & CAPA (EPIC-07).** Owns supplier-quality Issue lifecycle; this module creates Issues
  with a `supplier_id` reference and consumes their lifecycle events to feed `SupplierScorecard`.
- **Evidence (EPIC-06).** Stores count-line photos, damage evidence, delivery signatures/photos;
  referenced by ID from `stock_count_lines` and `goods_receipt_lines`.
- **Integrations & Extensibility (EPIC-13).** ERP/POS sync — inbound SKU master sync, outbound PO
  transmission (EDI/email/portal), inbound invoice data for price-variance comparison, outbound
  goods-receipt confirmation.
- **Analytics & Control Tower (EPIC-12/15).** Consumes this module's events (§12) and CDC feed for
  shrinkage rate, stock-out rate, PO cycle time, and supplier performance KPIs.
- **Identity & Organization (EPIC-01/02).** `location_id` scoping, `UserScope` for
  `inventory:*` permission scopes, org-hierarchy-based approval-route resolution (`ROLE_AT_LOCATION`/
  `ROLE_AT_PARENT_NODE` approver types resolve through the same OrganizationNode graph).

## 12. Canonical Events

Following the platform event envelope (source-spec §18.4, conventions.md API conventions):

- `stock_count.created`
- `stock_count.line_updated`
- `stock_count.variance_flagged`
- `stock_count.completed`
- `purchase_order.created`
- `purchase_order.submitted_for_approval` (triggers the `PURCHASE_APPROVAL` WorkItem creation)
- `purchase_order.approved`
- `purchase_order.rejected`
- `purchase_order.sent`
- `purchase_order.receipt_recorded`
- `purchase_order.closed`
- `goods_receipt.price_variance_flagged`
- `supplier.scorecard_updated`

This module also **consumes** Work Orchestration's `approval.step_decided` and
`task.completed`/`task.rejected`-class events (scoped to `PURCHASE_APPROVAL`-type WorkItems) and
Issues & CAPA's `issue.created`/`issue.resolved` (scoped to Issues carrying a `supplier_id`
reference).

## 13. Resolved Decisions

- **Cost-threshold-to-approval-route mapping (FR-INV-033).** Firm decision: a **tenant-configurable
  table** (`po_approval_thresholds`: ordered `(min_cost, max_cost, approval_route_id)` rows per
  tenant, optionally per location-subtree for large multi-brand tenants), not a hardcoded 3-tier
  constant. The $500/$5,000 figures in this spec are seeded as illustrative defaults per tenant at
  onboarding and are editable by Operations Admin thereafter. Rationale: a hardcoded tier would force
  an engineering change every time a tenant restructures its spend-authority matrix; a data-driven
  table costs nothing extra to build (LLD §1.10) and keeps this consistent with every other
  tenant-configurable threshold in the platform.
- **Expected-quantity source for a stock count (FR-INV-022).** Firm decision: `expected_quantity` is
  derived from `sku_location_stock.on_hand_quantity` (the last known authoritative value, updated
  only by the two paths in FR-INV-028/054) for MVP and for the foreseeable roadmap — **not** a live
  ERP/POS perpetual-inventory feed. Rationale: this keeps the count feature fully functional without
  a hard dependency on a tenant's ERP/POS integration being live, matches the platform's
  offline-first posture, and any tenant wanting perpetual-inventory-fed counts can layer that in later
  as an Integration Hub enhancement (module 12) that simply writes into `sku_location_stock` more
  frequently — no schema or workflow change required on this module's side.
- **Multi-currency PO handling (FR-INV-031).** Firm decision: **one currency per tenant** for PO
  cost-threshold evaluation and `purchase_orders.currency_code` (LLD §1.11), for the full scope of
  this spec. A supplier billed in a different currency is out of scope; a tenant with genuine
  multi-currency sourcing must normalize to its tenant currency before PO creation (e.g. via a manual
  exchange-rate entry at PO line level) until a dedicated currency-normalization capability is built.
  Rationale: multi-currency threshold evaluation requires a live FX-rate source and rounding/timing
  rules that are a distinct, non-trivial feature; scoping it out of this version avoids under-building
  it, and the vast majority of this platform's frontline-operations tenants source domestically.
- **Count-vs-PO variance is informational, not automatically corrective (FR-INV-024, FR-INV-028).**
  Firm decision: a large count variance never auto-creates a PO. It surfaces a low-stock alert (if
  below reorder point) or a supplier-quality Issue prompt (if damage/shrinkage suspected), and a
  human always initiates the resulting requisition. Rationale: consistent with source-spec P5
  (Policy before AI) and the platform-wide L1-Recommend default autonomy for any agent involvement
  (engineering-decisions.md §12 reinforces that any AI-adjacent recommendation affecting spend stays
  human-initiated).
- **`Supplier` scope (FR-INV-040).** Firm decision: `Supplier` in this module denotes a goods/
  inventory vendor only. Modules 16 (Asset Maintenance) and 25 (Field Service) keep their own
  `Vendor`/`VendorTechnician` concepts for service dispatch, and this version does not unify them.
  Rationale: the two concepts have materially different lifecycles (goods procurement vs. service
  dispatch/SLA) and forcing a shared abstraction now would add indirection with no near-term benefit;
  a tenant needing to represent the same legal entity as both simply creates both records and links
  them informally via a shared `integration_ref` — a future "business partner" unification is a
  distinct architecture-review decision, not a gap in this spec.
