# New-Location Launch Management — Architecture Diagrams

Corresponds to addendum Module 23 (EPIC-03 extension). Focused diagram set for this module only;
narrative detail lives in `functional-spec.md`, `high-level-design.md`, `low-level-design.md`.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Clients
        WEB[Web Admin / Control Tower —\nTemplate Builder & Launch Portfolio]
        MOB[Mobile App — executes LAUNCH_TASK WorkItems\nlike any other assigned work]
    end

    GW[API Gateway / BFF]

    subgraph LaunchService [Launch Orchestration Service]
        API[Launch API Layer]
        TPL[Template Authoring & Versioning]
        EXP[Template-to-WorkItem Expansion Engine]
        ROLL[Progress Rollup & At-Risk Engine]
        PORT[Portfolio Query Service]
    end

    PG[(Postgres — launch_* tables)]
    REDIS[(Redis — portfolio cache)]
    KAFKA[(Kafka — Event Backbone)]

    WORK[Work Orchestration Service\nowns WorkItem, work_item_dependencies]
    ORG[Identity/Org Service\nowns OrganizationNode]
    INTEG[Integration Hub]
    ANALYTICS[Analytics Platform]
    NOTIF[Notification Hub]

    WEB --> GW --> API
    MOB -.->|executes LAUNCH_TASK WorkItems directly against Work Orchestration, not this service| WORK
    API --> TPL --> PG
    API --> EXP
    API --> PORT --> REDIS
    PORT --> PG
    EXP --> PG
    EXP -->|bulk-create WorkItems + dependencies| WORK
    EXP -->|create OrganizationNode at trigger milestone| ORG
    WORK -->|task.completed/verified/overdue| KAFKA
    KAFKA --> ROLL --> PG
    ROLL --> REDIS
    KAFKA -->|launch.milestone_at_risk, launch.completed| NOTIF
    KAFKA --> ANALYTICS
    EXP -.->|integration_trigger_code| INTEG
```

This module sits beside, not inside, Work Orchestration: the only write path into `WorkItem`/
`work_item_dependencies` is the Expansion Engine's bulk-creation call, and the only write path into
`OrganizationNode` is that same engine's node-creation call at the configured trigger milestone.
Frontline execution of the generated `LAUNCH_TASK` WorkItems flows entirely through Work
Orchestration's own mobile/offline path — this service never sits in that runtime loop.

## 2. Sequence Diagram — Launch Instantiation and Deferred OrganizationNode Creation

```mermaid
sequenceDiagram
    participant Mgr as Regional/District Manager
    participant LS as Launch Orchestration Service
    participant WO as Work Orchestration
    participant ORG as Identity/Org
    participant KAFKA as Event Backbone

    Mgr->>LS: POST /api/v1/launch-projects (template + target_open_date)
    LS->>LS: expand template -> WorkItem graph (phases/milestones/tasks/deps)
    LS->>WO: POST /api/v1/work-items/bulk (campaign_id = launch_project.id)
    WO-->>LS: created work_item_ids
    LS-->>Mgr: 201 LaunchProject (status=PLANNED)
    Note over WO: frontline executes tasks over following weeks
    WO->>KAFKA: task.verified (trigger task, post_verification_action=CREATE_ORG_NODE)
    KAFKA-->>LS: consume event
    LS->>ORG: POST /api/v1/orgs/nodes (provisional attributes)
    ORG-->>LS: 201 { node_id }
    LS->>WO: patch remaining WorkItems.location_id = node_id
    LS->>KAFKA: launch.location_provisioned
```

The OrganizationNode is deliberately created only once the configured milestone task is verified —
not at instantiation time — so provisional/uncommitted openings never pollute live location-scoped
reporting or RBAC scope prematurely (functional-spec §8 Resolved Decisions).

## 3. Sequence Diagram — Progress Rollup and At-Risk Detection

```mermaid
sequenceDiagram
    participant WO as Work Orchestration
    participant KAFKA as Event Backbone
    participant ROLL as Rollup Engine
    participant CRON as At-Risk Sweep (periodic)
    participant NOTIF as Notification Hub

    WO->>KAFKA: task.verified
    KAFKA-->>ROLL: consume
    ROLL->>ROLL: recompute milestone/phase completion %
    alt milestone fully complete
        ROLL->>KAFKA: launch.milestone_completed
    end
    loop every 15 min
        CRON->>CRON: detectAtRisk() over non-terminal milestones
        alt behind expected date or projected to miss
            CRON->>KAFKA: launch.milestone_at_risk
            KAFKA-->>NOTIF: alert Regional Manager
        end
    end
```

Rollup is purely event-driven off Work Orchestration's own canonical events, with a periodic sweep
(reconciliation and at-risk detection) as the backstop for missed events and for the time-based
projection logic that no single event can trigger on its own.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    LAUNCH_TEMPLATES ||--o{ LAUNCH_TEMPLATE_VERSIONS : versions
    LAUNCH_TEMPLATE_VERSIONS ||--o{ LAUNCH_TEMPLATE_MILESTONES : milestones
    LAUNCH_TEMPLATE_MILESTONES ||--o{ LAUNCH_TEMPLATE_TASKS : tasks
    LAUNCH_TEMPLATE_VERSIONS ||--o{ LAUNCH_TEMPLATE_DEPENDENCIES : edges
    LAUNCH_TEMPLATE_VERSIONS ||--o{ LAUNCH_PROJECTS : instantiated_as
    LAUNCH_PROJECTS ||--o{ LAUNCH_PROJECT_MILESTONE_PROGRESS : rollup
    LAUNCH_PROJECT_MILESTONE_PROGRESS ||--o{ LAUNCH_PROJECT_TASK_LINKS : task_refs
    LAUNCH_PROJECT_TASK_LINKS }o--|| WORK_ITEMS : "references (Work Orchestration owns)"
    LAUNCH_PROJECTS }o--|| ORGANIZATION_NODES : "references (Identity/Org owns)"
```

Only `LaunchTemplate`/`LaunchTemplateVersion`/`LaunchTemplateMilestone`/`LaunchTemplateTask`/
`LaunchProject`/`LaunchProjectMilestoneProgress`/`LaunchProjectTaskLinks` are owned here; `WorkItem`
and `OrganizationNode` are drawn as external references to make the ownership boundary explicit in
the diagram itself.

## 5. State Diagram — LaunchProject Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PLANNED
    PLANNED --> IN_PROGRESS: first LAUNCH_TASK moves past AVAILABLE
    IN_PROGRESS --> AT_RISK: any non-complete milestone flagged AT_RISK
    AT_RISK --> IN_PROGRESS: all milestones recover (no longer behind/projected-miss)
    IN_PROGRESS --> COMPLETE: every WorkItem VERIFIED or admin-SKIPPED
    AT_RISK --> COMPLETE: every WorkItem VERIFIED or admin-SKIPPED (recovers on completion)
    PLANNED --> CANCELLED: admin cancels before work starts
    IN_PROGRESS --> CANCELLED: admin cancels mid-launch
    AT_RISK --> CANCELLED: admin cancels mid-launch
    COMPLETE --> [*]
    CANCELLED --> [*]
```

This state machine is **independent of, but entirely derived from**, the underlying WorkItem
completion states that Work Orchestration owns: `LaunchProject.status` is never set directly by a
user action on the project itself (except the explicit `CANCELLED` transition) — it is always a
computed rollup of the aggregate status of the `LAUNCH_TASK` WorkItems and
`LaunchProjectMilestoneProgress` rows underneath it, recomputed by the Rollup Engine and At-Risk
Sweep described in low-level-design.md §3.2–3.3. The platform's canonical `WorkItem` state machine
(`DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW → APPROVED → VERIFIED`,
plus `BLOCKED`/`OVERDUE`/`ESCALATED`/`REJECTED` branches) continues to govern every individual task
exactly as it does for any other WorkItem type; this diagram is the launch-level abstraction over
many concurrent instances of that machine, not a replacement for it.
