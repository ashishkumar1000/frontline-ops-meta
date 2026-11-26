# Work Orchestration — Architecture Diagrams

## 1. Component Diagram

```mermaid
flowchart TD
    subgraph Clients
        MOB[Mobile App RN/WatermelonDB]
        WEB[Web Admin / Control Tower]
        AGENT[Agent Runtime]
    end

    GW[API Gateway / BFF]

    subgraph WorkService[Work Service]
        API[Work API Layer]
        SCHED[Scheduling Engine]
        ASSIGN[Assignment & Routing]
        SM[State Machine Engine]
        ESC[Escalation/SLA Engine]
        APPR[Approval Engine - multi-level chains]
        DEP[Dependency Resolver]
        CACHE[Today-List / Approvals-Queue Cache - Redis]
    end

    subgraph Temporal[Temporal.io Cluster]
        TWF[Recurrence Workflows - one per series]
        TSLA[SLA/Escalation Workflows - one per WorkItem]
        TAPP[Approval-Step SLA Workflows - one per active step]
    end

    PG[(PostgreSQL)]
    KAFKA[[Event Backbone - Kafka]]
    ORG[Identity/Org Service]
    FORM[Form Service]
    EVID[Evidence Service]
    ISSUE[Issue/CAPA Service]
    NOTIF[Notification Hub]

    MOB & WEB & AGENT --> GW --> API
    API --> SM & ASSIGN & APPR & CACHE
    SCHED --> PG
    SCHED <--> TWF
    ESC <--> TSLA
    APPR <--> TAPP
    APPR --> PG
    SM --> PG
    ASSIGN --> PG
    DEP --> PG
    SM & SCHED & APPR -->|events| KAFKA
    KAFKA --> SCHED
    API -.scope.-> ORG
    API -.template.-> FORM
    SM -.evidence gate.-> EVID
    SM -.completion.-> ISSUE
    APPR -.approver resolution.-> ORG
    ESC & ASSIGN & APPR --> NOTIF
```

The Work API layer fronts five internal engines that share the same Postgres store: the
**Scheduling Engine** (backed by long-lived Temporal recurrence workflows, one per `Schedule`),
the **State Machine Engine** (sole authority over `WorkItem.status` transitions), the
**Assignment & Routing** component (individual/role/location/queue assignment), the **Approval
Engine** (sequential multi-level approval chains — see §5/§6 below), and the **Dependency
Resolver** (graph-edge gating). The **Escalation/SLA Engine** runs one short-lived Temporal
workflow per WorkItem that carries an SLA, *and* the Approval Engine reuses the identical pattern
one level down — one short-lived Temporal workflow per **active approval step** — decoupling
high-cardinality per-instance/per-step timers from the low-cardinality per-series recurrence
workflows. All state changes are mirrored onto Kafka as canonical `task.*`/`approval.*` events;
other modules (Identity/Org, Form, Evidence, Issue/CAPA, Notification Hub) are referenced by ID and
interacted with via events or narrow synchronous calls, never owned.

## 2. Sequence Diagram — Recurrence to Device

```mermaid
sequenceDiagram
    participant TWF as Temporal Recurrence Workflow
    participant WS as Work Service
    participant PG as Postgres
    participant KAFKA as Event Backbone
    participant NOTIF as Notification Hub
    participant DEV as Mobile Device

    TWF->>WS: computeNextInstances(schedule_id, window)
    WS->>PG: insert work_items (status=AVAILABLE), advance high_water_mark
    WS->>KAFKA: task.created
    KAFKA-->>NOTIF: task.created (assignee resolved)
    NOTIF->>DEV: push notification
    DEV->>WS: GET /work-items?assignee=me&updated_since=...
    WS-->>DEV: new item included
    DEV->>DEV: upsert local DB; Today screen renders offline-first
```

A Temporal recurrence workflow wakes on its own schedule-specific checkpoint, expands the next
window of occurrences transactionally into `work_items`, and emits `task.created`. Delivery to the
device is push-assisted but never required for correctness: the client's own sync cycle (triggered
by push, foreground, or periodic background sync) is the actual mechanism that lands the item in
local storage, consistent with the platform's offline-first design rule.

## 3. Sequence Diagram — SLA Breach and Escalation

```mermaid
sequenceDiagram
    participant TSLA as Temporal SLA Workflow
    participant WS as Work Service
    participant PG as Postgres
    participant KAFKA as Event Backbone
    participant NOTIF as Notification Hub
    participant ISSUE as Issue/CAPA Service

    TSLA->>TSLA: sleep until sla_target_at
    TSLA->>WS: queryStatus(work_item_id)
    WS-->>TSLA: still open
    TSLA->>WS: markOverdue
    WS->>PG: status=OVERDUE (optimistic)
    WS->>KAFKA: task.overdue
    loop each ladder step
        TSLA->>TSLA: sleep until step offset elapsed
        TSLA->>WS: queryStatus + executeStep (NOTIFY/REASSIGN/CREATE_ISSUE)
        WS->>PG: status=ESCALATED (first rung)
        WS->>KAFKA: escalation.rung_fired / task.escalated
        KAFKA-->>NOTIF: notify targets
        opt action=CREATE_ISSUE
            WS->>ISSUE: create Issue
        end
    end
```

Each WorkItem with an `escalation_policy_id` gets its own durable Temporal workflow the moment it
becomes `AVAILABLE`. The workflow re-checks live status before every sleep and every step, so a
completion at any point stops further rungs without any separate cancellation call — the workflow
simply observes the terminal/paused state on its next wake and exits. This durability guarantees no
rung is lost or duplicated across Work Service process restarts.

## 4. Sequence Diagram — Multi-Level Approval Chain

```mermaid
sequenceDiagram
    participant A1 as Approver: Site Manager
    participant WS as Work Service (Approval Engine)
    participant TAPP as Temporal (per-step SLA)
    participant KAFKA as Event Backbone
    participant NOTIF as Notification Hub
    participant A2 as Approver: District Manager
    participant A3 as Approver: Regional VP

    Note over WS: WorkItem -> PENDING_REVIEW instantiates 3 work_item_approval_steps (step 0 ACTIVE)
    WS->>TAPP: start step-0 SLA timer
    WS->>KAFKA: approval.step_activated(0)
    KAFKA-->>NOTIF: notify A1
    A1->>WS: decision APPROVE (step 0)
    WS->>WS: resolve step 1 approver, activate
    WS->>TAPP: start step-1 SLA timer
    WS->>KAFKA: approval.step_decided(0) / approval.step_activated(1)
    KAFKA-->>NOTIF: notify A2
    A2->>WS: decision APPROVE (step 1)
    WS->>WS: resolve step 2 approver, activate
    WS->>TAPP: start step-2 SLA timer
    WS->>KAFKA: approval.step_decided(1) / approval.step_activated(2)
    KAFKA-->>NOTIF: notify A3
    A3->>WS: decision APPROVE (step 2, final)
    WS->>WS: no next step -> WorkItem PENDING_REVIEW -> APPROVED
    WS->>KAFKA: approval.step_decided(2) / approval-outcome APPROVED
    KAFKA-->>NOTIF: notify original submitter + A1 + A2 + A3
```

Each step is a durable, independently-timed link in the chain: only the currently `ACTIVE` step's
resolved approver can act, the next step's approver is resolved fresh at activation time (never at
WorkItem-creation time, so a reorg between steps is reflected correctly), and a rejection at any
step immediately fans out `SKIPPED` to every step still `PENDING` rather than leaving them dangling.
Full detail — including the escalation and rejection branches — is in
`low-level-design.md` §3.4.

## 5. ER Diagram

```mermaid
erDiagram
    WORK_ITEM_TYPES ||--o{ WORK_ITEMS : classifies
    SCHEDULES ||--o{ WORK_ITEMS : materializes
    ESCALATION_POLICIES ||--o{ WORK_ITEMS : governs
    WORK_ITEMS ||--o{ WORK_ITEM_ASSIGNMENTS : "assignment history"
    WORK_ITEMS ||--o{ WORK_ITEM_DEPENDENCIES : "predecessor edges"
    WORK_ITEMS ||--o{ WORK_ITEM_APPROVAL_STEPS : "instantiated chain"
    APPROVAL_ROUTES ||--o{ SCHEDULES : "default for"
    APPROVAL_ROUTES ||--o{ WORK_ITEM_APPROVAL_STEPS : "template for"

    WORK_ITEM_TYPES {
        uuid id PK
        text code
        text base_behavior
    }
    WORK_ITEMS {
        uuid id PK
        uuid tenant_id
        uuid type_id FK
        uuid schedule_id FK
        uuid approval_route_id FK
        smallint current_approval_step_index
        text status
        text assignee_type
        timestamptz due_at
        bigint version
    }
    WORK_ITEM_APPROVAL_STEPS {
        uuid id PK
        uuid work_item_id FK
        uuid approval_route_id FK
        smallint step_index
        text approver_type
        uuid resolved_approver_user_id
        text status
        text decision
        timestamptz sla_target_at
        bigint version
    }
    WORK_ITEM_ASSIGNMENTS {
        uuid id PK
        uuid work_item_id FK
        text assignee_type
        timestamptz effective_from
    }
    SCHEDULES {
        uuid id PK
        text schedule_mode
        text rrule
        text temporal_workflow_id
    }
    WORK_ITEM_DEPENDENCIES {
        uuid work_item_id FK
        uuid depends_on_work_item_id FK
        text unlock_state
    }
    ESCALATION_POLICIES {
        uuid id PK
        jsonb steps
        integer version
    }
    APPROVAL_ROUTES {
        uuid id PK
        jsonb steps
        integer version
    }
```

`WorkItem` is the hub entity: every type in §7.1 of the source spec is one row here, distinguished
by `type_id` and `type_config`. `Schedule` and `EscalationPolicy` are the two configuration entities
that drive, respectively, when a WorkItem is created and how it is timed/escalated once created.
`WorkItemAssignment` is deliberately append-only so historical attribution survives reassignment
and user deactivation (source-spec §24.1). `WorkItemDependencies` models the graph edges that gate
the `SCHEDULED/DRAFT → AVAILABLE` transition independent of any time-based trigger.
`WorkItemApprovalStep` is the multi-level-approval equivalent of `WorkItemAssignment`: an
append-style, per-WorkItem instantiation of the shared `ApprovalRoute` template, so a specific
WorkItem's approval chain remains a stable, audit-durable record even if the underlying route
template is edited or versioned later (FR-WORK-102).

## 6. Work Item State Machine

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> SCHEDULED
    SCHEDULED --> AVAILABLE
    AVAILABLE --> IN_PROGRESS
    IN_PROGRESS --> BLOCKED
    BLOCKED --> IN_PROGRESS
    IN_PROGRESS --> SUBMITTED
    SUBMITTED --> PENDING_REVIEW
    PENDING_REVIEW --> APPROVED
    PENDING_REVIEW --> REJECTED
    REJECTED --> IN_PROGRESS
    APPROVED --> VERIFIED
    AVAILABLE --> OVERDUE
    IN_PROGRESS --> OVERDUE
    OVERDUE --> ESCALATED
    VERIFIED --> [*]
    DRAFT --> CANCELLED
    SCHEDULED --> CANCELLED
    AVAILABLE --> CANCELLED
    CANCELLED --> [*]
```

This is the exact canonical `WorkItem` state machine reused verbatim from conventions.md /
source-spec §17.2: `DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW →
APPROVED → VERIFIED`, with side branches `IN_PROGRESS ↔ BLOCKED`, `PENDING_REVIEW → REJECTED →
IN_PROGRESS`, and `AVAILABLE/IN_PROGRESS → OVERDUE → ESCALATED`. Every one of the 11 WorkItem types
(§7.1) shares this single machine — type-specific behavior (e.g., Approval skipping execution,
Incident/Observation skipping `SCHEDULED`) is expressed as which transitions a type actually uses,
never as an alternate state set. `CANCELLED` is an explicit terminal branch available from any
pre-`IN_PROGRESS` state for admin/bulk cancellation (LLD §2.5), not part of the original diagram's
happy/exception path but required for the cancel bulk-action and location-closure edge cases.

Note that `PENDING_REVIEW` is a single WorkItem-level state that can internally span *several*
sequential approval levels — see §7 below for what happens underneath it during a multi-level
approval chain.

## 7. Approval Step Lifecycle (within `PENDING_REVIEW`)

```mermaid
stateDiagram-v2
    [*] --> PENDING: route step instantiated (FR-WORK-050.2)
    PENDING --> ACTIVE: prior step APPROVED, approver resolved (FR-WORK-053)
    [*] --> ACTIVE: step_index == 0, resolved immediately at instantiation
    ACTIVE --> APPROVED: approver decision = APPROVE
    ACTIVE --> REJECTED: approver decision = REJECT
    ACTIVE --> REJECTED: approver decision = MORE_EVIDENCE_REQUESTED
    ACTIVE --> ESCALATED: sla_target_at elapsed, no decision yet
    ESCALATED --> APPROVED: approver (or delegate) later decides APPROVE
    ESCALATED --> REJECTED: approver (or delegate) later decides REJECT
    ACTIVE --> NEEDS_ASSIGNMENT: no eligible approver resolvable (FR-WORK-101)
    NEEDS_ASSIGNMENT --> ACTIVE: reconciler resolves an eligible approver
    PENDING --> SKIPPED: an earlier step REJECTED / MORE_EVIDENCE_REQUESTED (FR-WORK-054)
    PENDING --> SKIPPED: auto-skip, self-approval or duplicate approver (FR-WORK-057/103)
    APPROVED --> [*]
    REJECTED --> [*]
    SKIPPED --> [*]
```

Every `WorkItemApprovalStep` row moves through this lifecycle independently, but only one step per
WorkItem is ever `ACTIVE` at a time (sequential routing, FR-WORK-050.1) — that invariant, not a
top-level WorkItem status, is what actually represents "which level is this approval currently at."
`ESCALATED` is a *notification* state layered on top of `ACTIVE` (the same approver, or a delegate,
still holds decision authority; escalation only widens who else is told, per FR-WORK-058) rather
than a hand-off of authority, which is why the diagram shows `ESCALATED` resolving directly to
`APPROVED`/`REJECTED` rather than back through `ACTIVE`. The WorkItem-level `PENDING_REVIEW →
APPROVED` transition (§6) fires only when the *last* step in the chain reaches `APPROVED`; a
`REJECTED` at any step fires `PENDING_REVIEW → REJECTED` immediately and fans `SKIPPED` out to
every remaining `PENDING` step in the same transaction (LLD §4.3).
