# Asset & Equipment Maintenance (CMMS) — Architecture Diagrams

Corresponds to **EPIC-16**. Focused diagram set for this module only; narrative detail lives in
`functional-spec.md`, `high-level-design.md`, `low-level-design.md`.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Clients
        MOB[Mobile App — Technician/Manager]
        WEB[Web Admin / Control Tower]
    end

    GW[API Gateway / BFF]

    subgraph CmmsService [Asset & Maintenance Service]
        API[Asset/Inventory API Layer]
        REG[Asset Registry Manager]
        PMS[PM Scheduler]
        INV[Parts Inventory Manager]
        TECH[Technician/Vendor Directory]
        CONS[Event Consumer]
        OUT[Transactional Outbox]
    end

    PG[(Postgres)]
    TMP[Temporal]
    KAFKA[(Kafka)]
    REDIS[(Redis)]

    WORK[Work Orchestration]
    EVID[Evidence Service]
    ISSUE[Issues & CAPA]
    ORG[Identity/Org]
    PROC[Inventory/Procurement — Module 19]
    FSVC[Field Service — Module 25]

    MOB --> GW --> API
    WEB --> GW
    API --> REG --> PG
    API --> INV --> PG
    API --> TECH --> PG
    PMS --> PG
    PMS --> TMP
    PMS -- "create_work_item()" --> WORK
    CONS -- "task.completed / task.overdue" --> KAFKA
    CONS --> REG
    CONS --> INV
    OUT --> KAFKA
    API --> REDIS
    REG --> EVID
    REG -- "recurring failure" --> ISSUE
    API --> ORG
    INV -- "low-stock" --> PROC
    TECH -- "vendor handoff" --> FSVC
```

Narration: this is the same module boundary described in `high-level-design.md` §2 — the Asset &
Maintenance Service owns the Asset Registry, PM Scheduler, and Parts Inventory Manager, and the
diagram makes the one-way relationship with Work Orchestration explicit: the PM Scheduler only ever
*calls out* to create work (`create_work_item`), and the Event Consumer only ever *listens* for
completion/overdue signals. No component in this service writes to Work Orchestration's `work_items`
table directly.

## 2. Sequence Diagram — PM Schedule Materializes a MaintenanceWorkOrder

```mermaid
sequenceDiagram
    participant TWF as Temporal: MaintenanceScheduleWorkflow
    participant PMS as PM Scheduler
    participant WORK as Work Orchestration
    participant DEV as Technician (Mobile)

    TWF->>PMS: evaluateSchedule(schedule_id) at checkpoint
    PMS->>PMS: compute due_at (calendar RRULE or meter-based estimate, LLD §4.1)
    PMS->>WORK: POST /internal/work-items (type_code=MAINTENANCE_WORK_ORDER, asset_id, due_at)
    WORK-->>PMS: 201 work_item created (status=AVAILABLE)
    WORK-->>DEV: task.created -> push notification, appears in technician's Today list
```

Narration: the full transactional detail (high-water-mark persistence, idempotency key shape,
retry-safety under Temporal's at-least-once activity semantics) is in `low-level-design.md` §3(a).
This condensed view exists to show the module boundary at a glance: everything left of the arrow
into Work Orchestration is this module's responsibility; everything from that point forward
(assignment, execution, completion) is Work Orchestration's.

## 3. Sequence Diagram — Breakdown Report to Completed Repair with Parts Usage

```mermaid
sequenceDiagram
    actor Mgr as Store/Site Manager
    participant CMMS as Asset & Maintenance Service
    participant WORK as Work Orchestration
    participant DEV as Technician (Mobile)

    Mgr->>CMMS: report breakdown (asset OUT_OF_SERVICE)
    CMMS->>WORK: create MaintenanceWorkOrder (origin=AD_HOC)
    WORK-->>DEV: assigned, notified
    DEV->>WORK: start work (IN_PROGRESS)
    WORK-->>CMMS: task.status_changed -> asset UNDER_MAINTENANCE
    DEV->>CMMS: record parts usage (stock decremented)
    DEV->>WORK: complete (SUBMITTED -> ... -> VERIFIED)
    WORK-->>CMMS: task.completed -> asset back to IN_SERVICE
```

Narration: full detail, including the duplicate-breakdown check, the parts-pending completion block,
and the stock-decrement concurrency guard, is in `low-level-design.md` §3(b)/§4.2/§6. This view
highlights that asset operational status is driven entirely by consuming Work Orchestration's status
events, never by this module guessing at execution state independently.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    ASSET_TYPES ||--o{ ASSETS : classifies
    ASSETS ||--o{ ASSETS : "parent_asset_id"
    ASSETS ||--o{ ASSET_STATUS_EVENTS : "downtime history"
    ASSETS ||--o{ ASSET_LOCATION_HISTORY : "location moves"
    ASSETS ||--o{ METER_READINGS : "meter history"
    ASSETS ||--o{ MAINTENANCE_SCHEDULES : "targets"
    ASSET_TYPES ||--o{ MAINTENANCE_SCHEDULES : "targets (by type)"
    PART_INVENTORY_ITEMS ||--o{ PART_USAGES : "consumed by"
    VENDORS ||--o{ PART_INVENTORY_ITEMS : "preferred vendor"

    ASSET_TYPES {
        uuid id PK
        text code
        jsonb custom_attributes_schema
    }
    ASSETS {
        uuid id PK
        uuid asset_type_id FK
        uuid parent_asset_id FK
        uuid location_id
        text operational_status
        text criticality
        smallint active_work_order_count
    }
    ASSET_STATUS_EVENTS {
        uuid id PK
        uuid asset_id FK
        text status
        timestamptz started_at
        timestamptz ended_at
    }
    ASSET_LOCATION_HISTORY {
        uuid id PK
        uuid asset_id FK
        uuid to_location_id
        timestamptz effective_at
    }
    MAINTENANCE_SCHEDULES {
        uuid id PK
        uuid target_asset_id FK
        text schedule_basis
        text meter_type
        numeric meter_interval_value
        text status
    }
    METER_READINGS {
        uuid id PK
        uuid asset_id FK
        text meter_type
        numeric value
        timestamptz recorded_at
    }
    PART_INVENTORY_ITEMS {
        uuid id PK
        text part_number
        uuid location_id
        integer quantity_on_hand
        integer reorder_point
    }
    PART_USAGES {
        uuid id PK
        uuid part_inventory_item_id FK
        uuid work_item_id
        integer quantity_used
    }
    VENDORS {
        uuid id PK
        text name
    }
```

Narration: `Asset` is the aggregate root of this module, with append-only history tables
(`asset_status_events`, `asset_location_history`, `meter_readings`) hanging off it to preserve
Product Principle P7 (historical truth) independent of the mutable `assets` current-state row.
`MaintenanceSchedule` targets either a single `Asset` or an `AssetType`+location-scope combination
(exactly one, enforced by a check constraint — `low-level-design.md` §1.1). `PartUsage` and
`MeterReading` both carry a `work_item_id` — a loose, unenforced cross-service reference into Work
Orchestration's `work_items` table, deliberately not modeled as a foreign key here since that table
is owned by a different bounded context.

## 5. State Diagram — Asset Operational Status

```mermaid
stateDiagram-v2
    [*] --> IN_SERVICE
    IN_SERVICE --> OUT_OF_SERVICE : breakdown reported / manual flag
    IN_SERVICE --> UNDER_MAINTENANCE : scheduled PM work started
    OUT_OF_SERVICE --> UNDER_MAINTENANCE : technician starts work (WorkItem -> IN_PROGRESS)
    UNDER_MAINTENANCE --> IN_SERVICE : last active work order VERIFIED
    IN_SERVICE --> RETIRED : decommissioned
    OUT_OF_SERVICE --> RETIRED : decommissioned while down
    UNDER_MAINTENANCE --> RETIRED : decommissioned mid-repair
    RETIRED --> [*]
```

Narration: this is the `Asset.operational_status` lifecycle owned entirely by this module —
`IN_SERVICE → OUT_OF_SERVICE → UNDER_MAINTENANCE → IN_SERVICE`, plus the terminal `RETIRED` state
reachable from any non-terminal status (functional-spec §6, FR-CMMS-030/005). **This is deliberately
not a work-order state diagram.** The `MaintenanceWorkOrder` that typically drives these transitions
(via the Event Consumer reacting to `task.status_changed`/`task.completed`) has its own execution
state machine — `DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW →
APPROVED → VERIFIED`, with `BLOCKED`/`OVERDUE`/`ESCALATED`/`REJECTED` branches — and that machine is
the canonical `WorkItem` state machine owned and diagrammed by Work Orchestration
(`02-work-orchestration/architecture-diagram.md`). Re-drawing it here would create a second source
of truth for the same execution state; this module instead only diagrams the asset-side effect of
that execution (a physical piece of equipment's availability), which is conceptually and temporally
distinct from — though correlated with — the work order acting on it (e.g., an asset can be manually
flagged `OUT_OF_SERVICE` with no work order yet created, and a `MaintenanceWorkOrder` can complete
without ever having taken the asset fully down, per functional-spec FR-CMMS-030).
