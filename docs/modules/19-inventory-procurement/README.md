# Module 19 — Inventory, Procurement & Supplier Management

> This module gives retail and hospitality operations a governed system of record for stock and
> spending: what is on the shelf, what was actually counted, what needs reordering, who approved
> the purchase, and whether the supplier delivered on time. It replaces paper count sheets and
> email-based purchase approvals with mobile stock counts and threshold-routed purchase orders.
> It is mainly used by frontline associates (counting, receiving), store managers (ordering,
> approving, resolving variances), finance (high-value approvals), and operations admins
> (catalog and supplier management).

| | |
|---|---|
| **Module code** | INV (`FR-INV-###`) |
| **Owning service** | Inventory/Procurement Service |
| **Tier** | 2 |

## In plain terms
- Count stock on a phone: cycle counts, full counts, or quick spot checks, with barcode scanning
  to jump straight to the right item and a running tally of what has been counted so far.
- Flag variances automatically: when counted quantity differs from expected by more than a
  tenant-set threshold (default 5% or $50), the line must get a reason or a photo before the
  count can be submitted.
- Create purchase orders that route themselves: orders under $500 are auto-approved, mid-range
  orders go to one approver, and high-cost orders walk the full Store Manager → District Manager
  → Finance chain — the thresholds are tenant-configurable data, not hardcode.
- Receive deliveries against the purchase order, including partial deliveries, and flag price or
  quantity mismatches for review.
- Track supplier performance: on-time delivery rate, fulfillment accuracy, lead times, and
  quality issues, rolled up into a scorecard dashboard.

## Key concepts
| Concept | What it means |
|---|---|
| `SKU` | One stock-keeping product in the tenant's catalog (code, barcode, category, unit of measure, reorder thresholds). |
| `StockCount` / `StockCountLine` | A count job (a `STOCK_COUNT` WorkItem) and its per-SKU lines: expected vs. counted quantity and the computed variance. |
| `PurchaseOrder` / `PurchaseOrderLine` | The buy request to a supplier, with its own lifecycle (`DRAFT` through `CLOSED`) including post-approval states like `SENT_TO_SUPPLIER` and `PARTIALLY_RECEIVED`. |
| `PURCHASE_APPROVAL` WorkItem | The approval step of a PO's life — a Work Orchestration WorkItem that reuses the platform's multi-level Approval Routing engine. |
| `Supplier` | A goods supplier record (a goods vendor — not a repair/service vendor, which live in modules 16 and 25). |
| `SupplierScorecard` | A rolling aggregate of a supplier's on-time delivery, fulfillment accuracy, and quality-issue rate. |
| `GoodsReceipt` | The record of what physically arrived for a PO, reconciled line by line against what was ordered. |

## How it connects to the rest
- **Work Orchestration (module 02 / EPIC-03)** — owns everything schedulable about counts and the
  entire multi-level Approval Routing mechanism (`approval_routes` / `work_item_approval_steps`).
  This module explicitly reuses it and never stores its own "who approves next" logic or a second
  approval engine.
- **Forms & Workflow Builder (module 03 / EPIC-04)** — a stock count is a template-backed
  `STOCK_COUNT` WorkItem; the count's on-screen form is a `TemplateVersion` like any other.
- **Issues & CAPA (module 06 / EPIC-07)** — supplier quality problems (damaged, wrong, or expired
  goods) become `Issue`s carrying a `supplier_id`; scorecard quality metrics consume those issues'
  events.
- **Evidence (module 05 / EPIC-06)** — damage photos, count-sheet scans, and delivery signatures
  are referenced by ID from count and receipt lines.
- **Integrations & Extensibility (module 13 / EPIC-13)** and **Analytics (module 11 / EPIC-12/15)**
  — ERP/POS sync and PO transmission run through the Integration Hub; shrinkage and stock-out KPIs
  are computed in Analytics from this module's events.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-INV-001–055 covering SKU catalog, stock counts, purchase orders, suppliers, receiving, plus UX states, edge cases, dependencies, events, and resolved decisions. |
| [high-level-design.md](./high-level-design.md) | Service responsibilities, component diagram, data ownership table, and the key design call: PO is its own aggregate, approval is a reused `PURCHASE_APPROVAL` WorkItem. |
| [low-level-design.md](./low-level-design.md) | Full schema (17 tables), REST endpoints, sequence diagrams, and pseudocode for cost-threshold route resolution and variance calculation. |
| [architecture-diagram.md](./architecture-diagram.md) | Focused Mermaid set: component, approval and count sequence diagrams, ER diagram, and the PurchaseOrder lifecycle state machine. |
