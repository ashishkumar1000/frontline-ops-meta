# Inventory, Procurement & Supplier Management — Architecture Diagrams

Focused diagram set for this module only; narrative detail lives in `functional-spec.md`,
`high-level-design.md`, `low-level-design.md`.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Clients
        MOB[Mobile App - counts, receiving, barcode scan]
        WEB[Web Admin / Control Tower - catalog, PO, suppliers, scorecards]
    end

    GW[API Gateway / BFF]

    subgraph InvProcService [Inventory/Procurement Service]
        API[Inventory API Layer]
        CAT[SKU Catalog Manager]
        CNT[Stock Count Manager + Variance Calculator]
        PO[Purchase Order Manager]
        ROUTE[Cost-Threshold Route Resolver]
        RCV[Goods Receipt Manager]
        SUP[Supplier / Scorecard Manager]
        ALERT[Low-Stock Alert Sweep]
        OUT[Transactional Outbox]
    end

    PG[(Postgres)]
    REDIS[(Redis)]
    KAFKA[(Kafka Event Backbone)]

    WORK[Work Orchestration Service\napproval_routes / work_item_approval_steps]
    FORM[Forms & Workflow Builder]
    EVID[Evidence Service]
    ISSUE[Issues & CAPA Service]
    INTEG[Integration Hub - ERP/POS]
    ANALYTICS[Analytics Platform]

    MOB --> GW --> API
    WEB --> GW

    API --> CAT --> PG
    CAT --> REDIS
    API --> CNT --> PG
    CNT -->|create_work_item type=STOCK_COUNT| WORK
    CNT --> FORM
    CNT --> EVID

    API --> PO --> ROUTE
    ROUTE -->|resolve ApprovalRoute by total_cost| PG
    PO -->|create_work_item type=PURCHASE_APPROVAL,\napproval_route_id| WORK
    PO --> RCV --> PG
    RCV --> ISSUE

    API --> SUP --> PG
    ALERT --> PG
    ALERT --> PO

    CAT <--> INTEG
    PO --> INTEG
    OUT --> KAFKA --> ANALYTICS
    KAFKA -.->|approval.step_decided, task.completed| PO
```

This module never implements approval logic of its own: the `Cost-Threshold Route Resolver`
computes *which* `ApprovalRoute` applies from a locally-owned config table, then hands off entirely
to Work Orchestration, which owns everything about *how* the multi-step chain executes. The dotted
feedback edge is this service consuming Work Orchestration's approval-outcome events to progress
the PO it owns.

## 2. Sequence Diagram — Purchase order approval (cost-threshold routing, reused multi-level mechanism)

```mermaid
sequenceDiagram
    participant MGR as Store Manager
    participant INV as Inventory Service
    participant WORK as Work Orchestration
    participant DM as District Manager
    participant FIN as Finance

    MGR->>INV: POST /purchase-orders/{id}/submit
    INV->>INV: resolve_approval_route(total_cost) -> multi-level tier
    INV->>WORK: POST /work-items {type=PURCHASE_APPROVAL, approval_route_id}
    WORK-->>INV: PENDING_REVIEW, step 0 ACTIVE
    INV-->>MGR: PO status = PENDING_APPROVAL

    DM->>WORK: approval-steps/1/decision {APPROVE}
    WORK-->>INV: event approval.step_decided
    FIN->>WORK: approval-steps/2/decision {APPROVE}
    WORK-->>INV: event task.completed (final step approved)
    INV->>INV: PO status = APPROVED
```

Every decision call in this flow targets Work Orchestration's approval-step endpoint directly; the
Inventory Service never exposes its own decision endpoint for a PO approval step.

## 3. Sequence Diagram — Stock count to variance-flagged completion

```mermaid
sequenceDiagram
    participant ASSOC as Associate
    participant INV as Inventory Service
    participant WORK as Work Orchestration

    INV->>WORK: create_work_item(STOCK_COUNT)
    ASSOC->>INV: enter counted_quantity per line (barcode-assisted)
    INV->>INV: variance = counted - expected; flag if over threshold
    ASSOC->>INV: submit count
    alt flagged lines present
        INV->>WORK: WorkItem -> PENDING_REVIEW (single-step ApprovalRoute)
    end
    INV->>INV: WorkItem -> VERIFIED; apply stock adjustment
```

A stock count's review gate (when flagged lines require manager sign-off) is itself a single-step
instance of the same `ApprovalRoute` mechanism used for multi-level PO approval — one mechanism,
two usage patterns.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    SKU ||--o{ SKU_SUPPLIER_LINK : "linked via"
    SKU }o--|| SKU_CATEGORY : "belongs to"
    SKU ||--o{ SKU_LOCATION_STOCK : "on-hand per-location"
    SUPPLIER ||--o{ SKU_SUPPLIER_LINK : "supplies"
    SUPPLIER ||--o{ SUPPLIER_SCORECARD : "scored by"
    SUPPLIER ||--o{ PURCHASE_ORDER : "fulfills"
    STOCK_COUNT ||--o{ STOCK_COUNT_LINE : "has"
    STOCK_COUNT_LINE }o--|| SKU : "counts"
    PURCHASE_ORDER ||--o{ PURCHASE_ORDER_LINE : "has"
    PURCHASE_ORDER_LINE }o--|| SKU : "orders"
    PURCHASE_ORDER ||--o{ GOODS_RECEIPT : "reconciled by"
    GOODS_RECEIPT ||--o{ GOODS_RECEIPT_LINE : "has"
    GOODS_RECEIPT_LINE }o--|| PURCHASE_ORDER_LINE : "receives against"
```

`ApprovalRoute` and `WorkItemApprovalStep` are intentionally absent from this diagram: they are
Work Orchestration's tables. `purchase_orders.approval_route_id` / `approval_work_item_id` are
cross-service references, not locally owned entities.

## 5. State Diagram — PurchaseOrder Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> PENDING_APPROVAL: submit (cost >= lowest threshold)
    DRAFT --> APPROVED: submit (cost under auto-approve threshold)
    PENDING_APPROVAL --> APPROVED: final approval step approved
    PENDING_APPROVAL --> REJECTED: any step rejected
    REJECTED --> DRAFT: manager edits and re-submits
    APPROVED --> SENT_TO_SUPPLIER: transmit to supplier
    SENT_TO_SUPPLIER --> PARTIALLY_RECEIVED: goods receipt with backorder remaining
    PARTIALLY_RECEIVED --> PARTIALLY_RECEIVED: additional partial receipt
    PARTIALLY_RECEIVED --> RECEIVED: final backordered quantity received
    SENT_TO_SUPPLIER --> RECEIVED: goods receipt fully satisfies all lines
    RECEIVED --> CLOSED: flags acknowledged / auto-close sweep
    DRAFT --> CANCELLED: manager cancels before submission
    APPROVED --> CANCELLED: cancelled before sending (rare, audited)
    CLOSED --> [*]
    CANCELLED --> [*]
```

`PENDING_APPROVAL` is not a self-contained state machine here — it internally delegates entirely to
Work Orchestration's approval-step mechanism, exactly as a generic `WorkItem`'s `PENDING_REVIEW`
state does (source-spec §17.2). This module only observes that delegated process transition to a
terminal `APPROVED`/`REJECTED` outcome via `approval.step_decided`/`task.completed` events and maps
it onto the two outward arrows shown above; it never re-derives approval progress from its own
logic.
