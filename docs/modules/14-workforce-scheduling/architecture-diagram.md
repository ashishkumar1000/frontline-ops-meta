# Workforce Scheduling & Labor Management — Architecture Diagrams

## 1. Component Diagram

```mermaid
flowchart TD
    subgraph Clients
        WEBMGR[Web Admin - Schedule Builder]
        MOBASSOC[Mobile App - Availability/Swaps]
    end

    GW[API Gateway / BFF]

    subgraph SchedulingService[Scheduling Service]
        API[Scheduling API Layer]
        TMPL[Template & Generation Engine]
        PUB[Publish & Diff Engine]
        AVAIL[Availability Manager]
        ASSIGN[Assignment & Auto-Fill Engine]
        SWAP[Swap/Pickup Engine]
        GAP[Coverage Gap Detector]
        POLICY[Labor Policy Engine]
        CACHE[(Redis Cache)]
    end

    subgraph Temporal[Temporal.io Cluster]
        TGEN[Template Generation Workflows]
        TGAP[Gap Recompute Workflows]
        TEXP[Swap-Expiry Timers]
    end

    PG[(PostgreSQL)]
    KAFKA[[Event Backbone - Kafka]]
    ORG[Identity/Org Service]
    WO[Work Orchestration - SHIFT_BASED mode]
    TA[Time & Attendance]
    NOTIF[Notification Hub]

    WEBMGR & MOBASSOC --> GW --> API
    API --> TMPL & PUB & AVAIL & ASSIGN & SWAP & GAP & CACHE
    TMPL <--> TGEN
    GAP <--> TGAP
    SWAP <--> TEXP
    ASSIGN & SWAP --> POLICY
    TMPL & PUB & AVAIL & ASSIGN & SWAP --> PG
    PUB & SWAP & GAP -->|events| KAFKA
    KAFKA --> WO
    API -.scope.-> ORG
    TA -.reads planned shifts.-> PG
    PUB & SWAP & GAP --> NOTIF
```

The Scheduling API layer fronts six engines sharing one Postgres store: **Template & Generation**
(expands `ShiftTemplate`s into `DRAFT` `Shift` rows via a per-template Temporal workflow), **Publish &
Diff** (owns the `DRAFT → PUBLISHED` transition and the changes-since-last-publish view), **Availability
Manager** (recurring/one-off/time-off windows), **Assignment & Auto-Fill** (manual and rule-based
assignment, delegating validation to the **Labor Policy Engine**), **Swap/Pickup** (the full
swap-request lifecycle, also policy-gated), and the **Coverage Gap Detector** (near-real-time gap
recomputation via per-location/role Temporal workflows). This service is intentionally a peer of, not
a subordinate to, Work Orchestration: the only coupling is the `shift.*` event contract flowing one
way into Work Orchestration's `SHIFT_BASED` schedule mode, and a narrow read API that Time &
Attendance uses for its planned-vs-actual comparison.

## 2. Sequence Diagram — Publish to Work Orchestration Consumption

```mermaid
sequenceDiagram
    participant MGR as Manager
    participant SS as Scheduling Service
    participant PG as Postgres
    participant KAFKA as Event Backbone
    participant WO as Work Orchestration
    participant DEV as Associate Device

    MGR->>SS: POST /shifts/publish
    SS->>PG: UPDATE shifts SET status=PUBLISHED (range)
    SS->>KAFKA: shift.published (per shift)
    KAFKA-->>DEV: push notification -> sync fetch
    KAFKA-->>WO: shift.published
    WO->>WO: SHIFT_BASED mode materializes shift-anchored WorkItem(s)
    WO->>KAFKA: task.created
    KAFKA-->>DEV: push notification for the new task
```

Publishing is the single event that turns a roster plan into two independent, asynchronous
downstream effects: the associate sees their shift, and Work Orchestration anchors whatever
checklist/task work belongs to that shift. Neither downstream effect blocks the publish response —
the manager's publish call returns as soon as the `Shift` rows are committed and the events are
handed to the outbox/Kafka producer.

## 3. Sequence Diagram — Swap Request to Both-Shift Update

```mermaid
sequenceDiagram
    participant A as Associate A (requester)
    participant SS as Scheduling Service
    participant POLICY as Labor Policy Engine
    participant B as Associate B (peer)
    participant MGR as Manager
    participant KAFKA as Event Backbone

    A->>SS: POST /swap-requests (source_shift_id)
    SS->>KAFKA: swap.requested
    B->>SS: POST /swap-requests/{id}/accept
    SS->>POLICY: evaluate hypothetical swap
    POLICY-->>SS: clear or violation flags
    alt approval required (policy or violation)
        SS->>MGR: swap.pending_approval (queue entry)
        MGR->>SS: POST /manager-decision APPROVE
        SS->>SS: atomically swap both shift_assignments
    else auto-approved
        SS->>SS: atomically swap both shift_assignments
    end
    SS->>KAFKA: shift.reassigned (x2), swap.approved
```

A swap only ever resolves through one of two paths — auto-approval or explicit manager approval —
but both paths converge on the same atomic both-shifts update, so the eventual `shift.reassigned`
events (and the resulting Work Orchestration re-materialization, if applicable) are identical
regardless of which approval path was taken. Full branch detail, including rejection and the
labor-law-forced-approval case, is in `low-level-design.md` §3.2.

## 4. ER Diagram

```mermaid
erDiagram
    SHIFT_TEMPLATES ||--o{ SHIFTS : generates
    SHIFTS ||--o{ SHIFT_ASSIGNMENTS : "assignment history"
    SHIFTS ||--o{ SWAP_REQUESTS : "source of"
    SHIFTS ||--o{ SWAP_REQUESTS : "target of"
    LABOR_POLICIES ||--o{ SHIFTS : "governs (by org_node_scope)"
    LABOR_POLICIES ||--o{ SWAP_REQUESTS : governs
    SHIFT_TEMPLATES ||--o{ COVERAGE_GAPS : "required-headcount source"

    SHIFT_TEMPLATES {
        uuid id PK
        uuid location_id
        uuid role_id
        time start_time
        time end_time
        integer headcount_required
        integer version
    }
    SHIFTS {
        uuid id PK
        uuid shift_template_id FK
        timestamptz start_at
        timestamptz end_at
        text status
        boolean open_for_pickup
        bigint version
    }
    SHIFT_ASSIGNMENTS {
        uuid id PK
        uuid shift_id FK
        uuid user_id
        text assignment_source
        boolean is_current
    }
    AVAILABILITY_WINDOWS {
        uuid id PK
        uuid user_id
        text availability_type
        boolean is_recurring
        text status
    }
    SWAP_REQUESTS {
        uuid id PK
        text swap_type
        uuid source_shift_id FK
        uuid target_shift_id FK
        text status
        bigint version
    }
    LABOR_POLICIES {
        uuid id PK
        uuid org_node_scope
        jsonb rules
        integer version
    }
    COVERAGE_GAPS {
        uuid id PK
        uuid location_id
        timestamptz slot_start_at
        integer gap
    }
```

`Shift` is the hub entity, generated from `ShiftTemplate` (or authored ad hoc) and pointing forward
to its current and historical `ShiftAssignment` rows. `SwapRequest` references `Shift` twice (source
and, for a `SWAP` type, target) rather than owning assignment state itself — the actual assignment
change only happens through `ShiftAssignment` inserts once a swap resolves, keeping one single
source of truth for "who is on this shift right now." `LaborPolicy` is scoped by `OrganizationNode`
subtree (`org_node_scope`), a narrower axis than tenant isolation, and is evaluated — never stored
as a denormalized flag — against both `Shift` assignment and `SwapRequest` acceptance. `CoverageGap`
is a derived, incrementally-maintained table rather than a pure computed view, trading a small
write-amplification cost for the sub-minute read freshness the coverage-gap UX needs.

## 5. State Diagram — Shift Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DRAFT: generated from template or authored ad hoc
    DRAFT --> PUBLISHED: manager publish (FR-WFS-011)
    DRAFT --> CANCELLED: removed before publish
    PUBLISHED --> PUBLISHED: reassigned via manual edit or approved swap (republish-diff, FR-WFS-012/013)
    PUBLISHED --> CANCELLED: manager cancel with reason (FR-WFS-015)
    CANCELLED --> [*]
    PUBLISHED --> [*]: shift time elapses (terminal, retained for history/Time & Attendance comparison)
```

A `Shift` has a deliberately small state set compared to `WorkItem` — it models a roster slot, not
an executable unit of work, so it has no `IN_PROGRESS`/`SUBMITTED`/`VERIFIED` concept of its own;
whatever execution state exists during the shift belongs to the `WorkItem`(s) Work Orchestration
materializes against it. `PUBLISHED → PUBLISHED` self-loops represent in-place changes (reassignment,
swap resolution) that do not change the shift's top-level status but do trigger the diff/notification
machinery (functional-spec §4) and, when the assignee changes, a `shift.reassigned` event.

## 6. State Diagram — Swap Request Lifecycle

```mermaid
stateDiagram-v2
    [*] --> OPEN: requester creates swap/pickup request
    OPEN --> PENDING_MANAGER_APPROVAL: peer accepts, approval required (policy or violation, FR-WFS-043/046)
    OPEN --> APPROVED: peer accepts, no approval required (auto-approve path)
    OPEN --> EXPIRED: no acceptance before expires_at (FR-WFS-045)
    OPEN --> CANCELLED: requester withdraws before acceptance
    PENDING_MANAGER_APPROVAL --> APPROVED: manager decision = APPROVE (both shifts updated atomically)
    PENDING_MANAGER_APPROVAL --> REJECTED: manager decision = REJECT (reason required, no shift change)
    APPROVED --> [*]
    REJECTED --> [*]
    EXPIRED --> [*]
    CANCELLED --> [*]
```

Only two paths reach `APPROVED` — direct auto-approval from `OPEN`, or explicit manager approval from
`PENDING_MANAGER_APPROVAL` — and both apply the identical atomic both-shift update (low-level-design
§6). A labor-law violation detected at acceptance time (FR-WFS-046) forces the `OPEN →
PENDING_MANAGER_APPROVAL` transition even under a tenant policy that would otherwise auto-approve,
which is why that edge is drawn unconditionally from `OPEN` rather than gated purely by the
tenant-level `SwapApprovalPolicy` setting. `REJECTED`, `EXPIRED`, and `CANCELLED` are all terminal
with no shift-side effect, preserving the original assignment unchanged.
