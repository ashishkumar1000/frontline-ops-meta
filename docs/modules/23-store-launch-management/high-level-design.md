# New-Location Launch Management — High-Level Design

Corresponds to **EPIC-03 (Work Core) extension**, addendum Module 23. Read alongside
`functional-spec.md` (this module) and `02-work-orchestration/high-level-design.md` /
`01-identity-organization/high-level-design.md`, which own the systems this module orchestrates.

## 1. Module Responsibilities and Boundaries

The **Launch Orchestration Service** is a thin configuration/orchestration layer. It owns exactly
three things:

1. **Launch Template authoring and versioning** — the reusable definition of what a launch project
   looks like (phases, milestones, tasks, dependency edges, anchor/offset rules, integration and
   node-creation trigger points).
2. **Launch Project lifecycle and progress rollup** — instantiating a template against a location,
   tracking phase/milestone progress and at-risk state in launch-native terms, and exposing the
   multi-launch portfolio view.
3. **Orchestration calls outward** — expanding a template into a Work Orchestration WorkItem +
   dependency graph at instantiation, and calling Identity/Org's node-creation API at the configured
   trigger point.

It explicitly does **not** own: a task execution engine, an assignment model, a scheduling engine, an
SLA/escalation engine, or a second state machine for "work." Those are Work Orchestration's job for
every `LAUNCH_TASK` WorkItem the template expands into. This is the same "configuration layer over a
shared execution substrate" pattern already used by other Tier 2/3 addendum modules that mint new
`WorkItem` types (e.g., Module 16 CMMS's `MAINTENANCE_WORK_ORDER`, Module 19's `STOCK_COUNT`) rather
than building a parallel work engine.

## 2. Component Diagram

```mermaid
flowchart TD
    subgraph Clients
        WEB[Web Admin / Control Tower — Launch Portfolio & Template Builder]
    end

    GW[API Gateway / BFF]

    subgraph LaunchService [Launch Orchestration Service]
        API[Launch API Layer]
        TPL[Template Authoring & Versioning Engine]
        EXP[Template-to-WorkItem Expansion Engine]
        ROLL[Progress Rollup & At-Risk Engine]
        PORT[Portfolio Query Service]
        OUT[Transactional Outbox]
    end

    PG[(Postgres — launch_templates, launch_template_milestones,\nlaunch_template_tasks, launch_projects,\nlaunch_project_milestone_progress)]
    REDIS[(Redis — portfolio dashboard cache, idempotency keys)]
    KAFKA[(Kafka — Event Backbone)]

    WORK[Work Orchestration Service\n(owns WorkItem, work_item_dependencies)]
    ORG[Identity/Org Service\n(owns OrganizationNode)]
    INTEG[Integration Hub]
    ANALYTICS[Analytics Platform]
    NOTIF[Notification Hub]

    WEB --> GW --> API
    API --> TPL
    API --> EXP
    API --> PORT
    TPL --> PG
    EXP --> PG
    EXP -- "bulk-create WorkItems +\nwork_item_dependencies\n(campaign_id = launch_project.id)" --> WORK
    EXP -- "create OrganizationNode\nat configured trigger milestone" --> ORG
    ROLL --> PG
    ROLL --> REDIS
    PORT --> REDIS
    PORT --> PG
    WORK -- "task.completed / task.verified /\ntask.overdue events" --> KAFKA
    KAFKA --> ROLL
    OUT --> KAFKA
    KAFKA -- "launch.milestone_at_risk,\nlaunch.completed" --> NOTIF
    KAFKA --> ANALYTICS
    EXP -. "integration_trigger_code\non task completion" .-> INTEG
```

Narrative: the Template Authoring engine and the Expansion engine are the only two components that
write outside this module's own tables — the former never does (templates are fully local), the
latter is the sole caller of Work Orchestration's bulk-creation API and Identity/Org's node-creation
API. The Rollup engine is a pure consumer of Work Orchestration's own domain events; it never queries
Work Orchestration's tables directly, preserving the service-boundary discipline from
conventions.md (each bounded context owns its transactional state).

## 3. Data Ownership

| Entity | Owned here | Notes |
|---|---|---|
| `LaunchTemplate`, `LaunchTemplateVersion` | Yes | Fully owned; immutable after publish per TemplateVersion pattern |
| `LaunchTemplateMilestone`, `LaunchTemplateTask`, template-level dependency edges | Yes | Owned as children of a `LaunchTemplateVersion` snapshot |
| `LaunchProject` | Yes | Owned; carries status, target dates, provisional-location attributes, `launch_template_version_id` FK |
| `LaunchProjectMilestoneProgress` | Yes | This module's own progress-rollup metadata (expected/actual dates, at-risk flag, completion %) — **derived** from, not a copy of, the underlying WorkItem states |
| `WorkItem` (type `LAUNCH_TASK`) | **No** — Work Orchestration | Referenced by ID (`work_item_id` FK) from `LaunchProjectMilestoneProgress`'s child task rows; this module never mutates WorkItem rows directly, only via Work Orchestration's API |
| `work_item_dependencies` | **No** — Work Orchestration | Wired by this module's expansion call but owned/enforced by Work Orchestration thereafter |
| `OrganizationNode` | **No** — Identity/Org | Created via API call; this module stores only the resulting `location_id` reference |

This "own the plan and the rollup, never the execution record" split is the core design decision of
the module and is what keeps it a configuration layer rather than a second work engine.

## 4. External Interfaces

**Outbound calls made by this module:**
- `POST /api/v1/work-items/bulk` (Work Orchestration) — bulk WorkItem + dependency creation at
  instantiation and on target-date recompute (§ LLD for payload shape).
- `POST /api/v1/work-items/bulk` with `operation: CANCEL` — on launch cancellation.
- `POST /api/v1/orgs/nodes` (Identity/Org) — at the configured OrganizationNode-creation trigger
  milestone.
- Outbound integration events (via the standard Event Backbone / Integration Hub webhook path) keyed
  by a task's `integration_trigger_code` — this module publishes the event; Integration Hub owns
  delivery.

**Inbound events consumed:**
- `task.completed`, `task.verified`, `task.overdue`, `task.cancelled` (Work Orchestration, filtered
  to `type_code = LAUNCH_TASK`) — drive progress rollup and at-risk recomputation.
- (No inbound calls from Identity/Org or Integrations are required; this module is the initiator of
  those interactions, not a responder.)

**Events produced by this module:**
- `launch.instantiated`, `launch.milestone_completed`, `launch.milestone_at_risk`,
  `launch.completed`, `launch.cancelled`, `launch.template_published` — all in the canonical event
  envelope (source-spec §18.4), consumed by Notification Hub and Analytics.

**APIs exposed** (full detail in low-level-design.md §3): template CRUD/publish, launch project
instantiate/get/cancel/patch, portfolio dashboard query.

## 5. Scalability

- A large enterprise tenant running dozens of concurrent launches, each with 50–200 tasks, is at
  most low tens of thousands of `LaunchProjectMilestoneProgress`-linked WorkItem references live at
  once — small relative to Work Orchestration's own scale target (millions of WorkItems/day platform
  -wide), so this module's own Postgres footprint is modest and does not need partitioning at
  current scale; revisit if a tenant runs hundreds of concurrent multi-hundred-task launches.
- The expansion engine's bulk-creation call is the one potentially expensive operation (up to ~200
  WorkItems + their dependency edges in one instantiation); it is implemented as a single bulk API
  call into Work Orchestration (not N individual calls) specifically to keep instantiation latency
  bounded and transactional on the Work Orchestration side (see NFR target, functional-spec §6).
- The portfolio dashboard is the module's main read-scaling concern (a manager viewing 20+ launches,
  each needing rolled-up phase %/at-risk state); it is served from a Redis-backed cache
  (§ LLD Caching) refreshed by the event-driven rollup engine rather than computed synchronously per
  request.

## 6. Multi-Tenancy

Standard platform pattern (conventions.md): every table carries `tenant_id`; Postgres RLS policies
as defense-in-depth behind application-layer scoping. Template and project visibility is additionally
scoped by `OrganizationNode`/`UserScope` — a template authored at a brand's org-node scope is only
instantiable at or below that scope; a Regional Manager's portfolio view is implicitly filtered to
launches whose (real or provisional) location falls within their scoped subtree.

## 7. Failure Modes and Degradation

| Failure | Behavior |
|---|---|
| Work Orchestration bulk-creation call fails mid-instantiation | The whole instantiation is treated as failed (not partially committed): `LaunchProject` is created in a `PLANNED` state with `provisioning_status = FAILED`, no partial WorkItem set is left orphaned client-side since the call is a single bulk operation on Work Orchestration's transactional side; the admin can retry the same idempotency key safely (FR-LAUNCH-014) |
| Identity/Org node-creation call fails at the trigger milestone | The triggering WorkItem stays `VERIFIED` (it genuinely was completed) but the `post_verification_action` remains `PENDING`; a background retry job (§ LLD Background Jobs) retries with backoff; the launch surfaces a distinct "location provisioning pending" banner rather than silently blocking |
| Work Orchestration event stream lag/outage | Progress rollup becomes stale but never blocks WorkItem execution (rollup is a pure read-side consumer); the periodic reconciliation sweep (§ LLD §8) catches up once the event stream recovers, and the portfolio UI shows a "last updated" timestamp so staleness is visible rather than silently wrong |
| Redis cache unavailable | Portfolio dashboard falls back to direct Postgres aggregation query (slower, but functionally correct) rather than failing the request |
| Integration Hub delivery failure for a task's trigger | Does not block or roll back the underlying task's `VERIFIED` state; Integration Hub owns its own retry/dead-letter handling per its module spec |

## 8. Technology Choices (Justified)

- **PostgreSQL**, same managed fleet as the rest of the platform — this module's data volume is
  small and relational (templates, projects, milestone rollups); no case for a specialized store.
- **No dedicated workflow engine (Temporal) instance for this module** — deliberate choice: the
  actual scheduling/SLA/escalation timers for each `LAUNCH_TASK` already run inside Work
  Orchestration's existing Temporal-backed engine (per its HLD). Standing up a second Temporal
  namespace here would duplicate that machinery for zero benefit; this module's own background needs
  (progress-rollup recompute, at-risk sweep, node-creation retry) are simple periodic jobs, not
  long-running durable workflows, so a standard scheduled-worker pattern (Kubernetes CronJob /
  existing worker pool) is sufficient and consistent with "use the simplest tool that satisfies the
  requirement" (conventions.md architectural decision principles).
- **Redis** for the portfolio dashboard cache and idempotency-key store, consistent with the
  platform-wide cache choice.
- **Kafka** (existing Event Backbone) for both consuming Work Orchestration's task events and
  producing this module's own `launch.*` events — no new event infrastructure introduced.
- This module is deliberately **not** a separate deployable service boundary candidate for
  independent scaling in the near term (conventions.md §16.2: "split services only where scale,
  deployment independence, security boundaries, or team ownership justify it") — it can reasonably
  ship as a bounded context inside the Work Service deployable, or as its own small Spring Boot
  service if template-authoring ownership sits with a different team; either is acceptable, and the
  data-ownership boundary above holds either way.
