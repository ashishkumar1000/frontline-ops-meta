# Field Service & Third-Party Vendor Scheduling — Architecture Diagrams

Focused diagram set for this module only. Full narrative in `functional-spec.md`,
`high-level-design.md`, `low-level-design.md`.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Clients
        MOB[Internal Mobile/Web]
        VPORTAL[Vendor Portal]
        FALLBACK[Email/SMS Scoped Link]
    end

    GW[API Gateway / BFF]
    VGW[Vendor-Facing Gateway]

    subgraph FsvcService [Field Service Service]
        API[Internal API]
        VAPI[Vendor-Facing API]
        GUARD[Vendor Scope Guard]
        DIR[Directory Manager]
        MATCH[Vendor Matcher]
        SCHED[Appointment Negotiation]
        SLA[SLA & Scorecard Engine]
    end

    PG[(Postgres)]
    WORK[Work Orchestration]
    ORG[Identity/Org]
    CMMS[Asset Maintenance — Module 16]

    MOB --> GW --> API
    VPORTAL --> VGW --> GUARD --> VAPI
    FALLBACK --> VGW
    CMMS -- "create_service_request" --> API
    API --> DIR & MATCH & SCHED --> PG
    VAPI --> SCHED
    SCHED -- "create_work_item / reassign" --> WORK
    GUARD -- "scope-kind + membership check" --> ORG
    DIR -- "provision restricted principal" --> ORG
    SLA -- "escalation_policy_id reused" --> WORK
```

Internal traffic and vendor-facing traffic enter through physically separate gateway routes. Every
vendor-facing call passes through the Vendor Scope Guard before reaching the Vendor-Facing API layer
— there is no code path from `VGW` to Postgres that skips it. The Appointment Negotiation component
is the only one that calls Work Orchestration to create or reassign the underlying `WorkItem`; this
module never writes `work_items.status` itself, matching the non-duplication pattern Module 16
already established for `MaintenanceWorkOrder`.

## 2. Sequence Diagram — Service Request to Confirmed Appointment

```mermaid
sequenceDiagram
    participant CMMS as Module 16 / Internal Requester
    participant FSVC as Field Service Service
    participant MATCH as Vendor Matcher
    participant VENDOR as Vendor (portal or fallback)
    participant WORK as Work Orchestration

    CMMS->>FSVC: create ServiceRequest
    FSVC->>MATCH: match by category + coverage + SLA tier
    MATCH-->>FSVC: top-ranked vendor
    FSVC->>VENDOR: notify (portal push or scoped email/SMS)
    VENDOR->>FSVC: propose/confirm window
    FSVC->>WORK: create_work_item / reassign (assignee_type=VENDOR)
    WORK-->>FSVC: WorkItem id
    FSVC-->>CMMS: ServiceRequest CONFIRMED, work_item linked
```

This is the "happy path" flow underlying FR-FSVC-010/011/020/021/030–032: matching and negotiation
are entirely this module's own state; the `WorkItem` is created or reassigned only at the moment a
window is actually confirmed, never earlier, so no orphaned `WorkItem` exists for a request that
never finds a vendor.

## 3. Sequence Diagram — Vendor Technician Scoped Access Check

```mermaid
sequenceDiagram
    participant TECH as Vendor Technician
    participant GUARD as Vendor Scope Guard
    participant ORG as Identity/Org
    participant PG as service_appointment_technician_history

    TECH->>GUARD: request appointment sr_1001
    GUARD->>ORG: confirm principal scope_kind = ASSIGNED_WORK_ITEM_ONLY
    GUARD->>PG: is sr_1001 in this technician's assigned set (currently or historically)?
    alt assigned
        PG-->>GUARD: match
        GUARD-->>TECH: 200 — scoped fields only (this appointment)
    else not assigned
        PG-->>GUARD: no match
        GUARD-->>TECH: 403 SCOPE_DENIED (logged as security event)
    end
```

This is the module's central security guarantee: a vendor technician's authorization predicate is
"is this specific `WorkItem` id in my dynamically computed assigned set," never "am I granted this
location/tenant" — narrower than the platform's narrowest internal role. The check runs on every
vendor-facing call, including list queries (the assigned-set is injected as the query's `WHERE`
clause, not applied as a post-fetch filter), so there is no code path that can leak an unassigned
appointment.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    VENDORS ||--o{ VENDOR_SERVICE_CATEGORIES : offers
    VENDORS ||--o{ VENDOR_TECHNICIANS : employs
    VENDORS ||--o{ VENDOR_COVERAGE_AREAS : covers
    VENDORS ||--o{ SERVICE_REQUESTS : "assigned to"
    VENDOR_TECHNICIANS ||--o{ SERVICE_APPOINTMENT_TECHNICIAN_HISTORY : "assigned to"
    SERVICE_REQUESTS ||--o{ SERVICE_APPOINTMENT_TECHNICIAN_HISTORY : "assignment history"
    VENDORS ||--o{ VENDOR_SLAS : "contracted targets"
    VENDORS ||--o{ VENDOR_SCORECARDS : "performance rollup"

    VENDORS {
        uuid id PK
        text name
        text sla_tier
        text status
        boolean has_platform_access
    }
    VENDOR_TECHNICIANS {
        uuid id PK
        uuid vendor_id FK
        uuid restricted_user_id "external, scoped Identity/Org User"
        text status
    }
    SERVICE_REQUESTS {
        uuid id PK
        uuid source_work_item_id
        uuid vendor_id FK
        uuid work_item_id "SERVICE_APPOINTMENT WorkItem, owned by Work Orchestration"
        text status
    }
    VENDOR_SLAS {
        uuid id PK
        uuid vendor_id FK
        integer response_time_target_minutes
        uuid escalation_policy_id "reused from Work Orchestration"
    }
    VENDOR_SCORECARDS {
        uuid id PK
        uuid vendor_id FK
        numeric on_time_arrival_rate
        numeric first_time_fix_rate
    }
```

Note `service_requests.work_item_id` and `vendor_slas.escalation_policy_id` are the two explicit
seams into Work Orchestration's own schema (`work_items`, `escalation_policies`) — referenced by id
only, never duplicated, per the module's non-duplication rule.

## 5. State Diagram — ServiceRequest Lifecycle

```mermaid
stateDiagram-v2
    [*] --> REQUESTED
    REQUESTED --> VENDOR_MATCHED: vendor assigned (auto or manual)
    REQUESTED --> NO_VENDOR_AVAILABLE: no eligible vendor found
    NO_VENDOR_AVAILABLE --> VENDOR_MATCHED: admin widens coverage / new vendor onboarded / manual match

    VENDOR_MATCHED --> APPOINTMENT_PROPOSED: window proposed (either party)
    APPOINTMENT_PROPOSED --> APPOINTMENT_PROPOSED: reject/counter-propose
    APPOINTMENT_PROPOSED --> CONFIRMED: window accepted by other party

    CONFIRMED --> APPOINTMENT_PROPOSED: reschedule before work starts
    CONFIRMED --> VENDOR_MATCHED: vendor cancels late (re-match, excluding this vendor)

    CONFIRMED --> IN_PROGRESS: underlying WorkItem -> IN_PROGRESS (vendor starts work)
    IN_PROGRESS --> COMPLETED: underlying WorkItem -> SUBMITTED (vendor marks done)
    COMPLETED --> SIGNED_OFF: on-site contact confirms + captures evidence

    VENDOR_MATCHED --> ESCALATED: SLA response-time breach (Work Orchestration escalation ladder)
    APPOINTMENT_PROPOSED --> ESCALATED: SLA breach / vendor unresponsive
    CONFIRMED --> ESCALATED: vendor no-show
    ESCALATED --> VENDOR_MATCHED: manager reassigns to a different vendor
    ESCALATED --> IN_PROGRESS: vendor resumes engagement after escalation contact

    REQUESTED --> [*]: cancelled
    VENDOR_MATCHED --> [*]: cancelled
    SIGNED_OFF --> [*]
```

This is this module's **own** lifecycle — it wraps but never replaces the underlying `WorkItem`
state machine (`DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW → APPROVED →
VERIFIED`, owned by Work Orchestration). `IN_PROGRESS`/`COMPLETED` here are driven one-way by that
underlying WorkItem's own transitions (via `task.status_changed` events); `SIGNED_OFF` is the one
state this module adds independently, representing the vendor-scheduling-specific concept of an
on-site attestation layered on top of (and typically preceding or coinciding with) the WorkItem's own
`VERIFIED` status. The `ESCALATED` branch is likewise driven by Work Orchestration's reused
escalation-ladder mechanism (`task.overdue`/`task.escalated` events against the `VendorSLA`-configured
`escalation_policy_id`), not a second SLA-timer implementation.
