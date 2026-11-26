# Module 02 — Work Orchestration

> The heart of the platform: it turns every operating standard — a daily opening checklist, an audit, a training assignment, a corrective action — into a WorkItem with an owner, a due date, and one shared lifecycle. A store manager can assign the opening checklist to the shift, and if it runs late, escalation ladders automatically remind, reassign, or raise an issue. Frontline associates live on its "Today" screen; managers run approvals, reassignments, and location health from it.

| | |
|---|---|
| **Module code** | None assigned in the spec (identified by folder `02-work-orchestration`) |
| **Owning service** | Work Service |
| **Epics** | EPIC-03 |

## In plain terms
- Represent any unit of frontline work as one of 11 WorkItem types — task, audit/inspection, approval, corrective action, training assignment, announcement acknowledgement, campaign task, site visit, asset inspection, survey, incident — plus tenant-defined sub-types like "Safety Walk".
- Schedule work eight ways: daily/weekly recurrence, "45 minutes before close" relative to store hours, per-shift, triggered by events (e.g. a sensor breach), location or role changes, other work completing, or seasonal campaign windows.
- Assign work to a person, to whoever is on shift with the right role (first-to-claim), to the whole location pool, or to a cross-site queue — and mass-reassign everything when someone leaves.
- Route approvals through multi-level chains (Store Manager → District Manager → Regional VP), with self-approvals auto-skipped, delegations audited, and escalations that notify without ever auto-approving.
- Enforce deadlines with SLA timers and escalation ladders (e.g. 15 min → notify assignee, 60 min → notify manager, 4 h → auto-create an issue), and sequence work with dependencies ("regional rollup unlocks only when all site checks are verified").
- Give associates a network-independent Today screen (urgent / due / overdue counts) and managers Exceptions, Approvals, Team, and Location Health views.

## Key concepts
| Concept | What it means |
|---|---|
| WorkItem | The universal unit of work — one entity, one state machine (`DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW → APPROVED → VERIFIED`, with overdue/escalated/blocked branches), no matter the type. |
| Schedule | The configuration that spawns WorkItems on a rolling window (default 14 days ahead); each has one of 8 schedule modes and maps to a long-lived Temporal workflow. |
| EscalationPolicy | An ordered ladder of steps like "15 min past SLA → notify; 240 min → create issue"; each rung fires from a durable Temporal timer. |
| ApprovalRoute | A reusable, versioned template of sequential approval steps; the WorkItem itself stays `PENDING_REVIEW` until the last step approves. |
| WorkItemApprovalStep | The per-WorkItem copy of a route's chain — a durable, auditable record of who approved what at which level, immune to later route edits. |
| work_item_dependencies | Graph edges that block a WorkItem from becoming available until predecessors reach a state, with ALL/ANY/threshold(n) unlock rules. |
| WorkItemAssignment | Append-only history of who owned a WorkItem and when — so reassignment and user deactivation never erase attribution. |
| Temporal workflow | Durable, restart-safe timers (one per schedule series, one per WorkItem SLA, one per active approval step) that survive process crashes without lost or duplicated escalations. |

## How it connects to the rest
- Consumes Identity & Organization (01) for locations, timezones, and role/scope eligibility, and reacts to its `user.deactivated` event to reassign open work.
- References Forms (03) `TemplateVersion` for form-backed work items, and gates submission/verification on Evidence (05) completeness checks.
- Is bidirectional with Issues & CAPA (06) (issues spawn corrective-action work items; completed work items progress the issue) and Learning (08) (repeated failures spawn training assignments; completions close them).
- Feeds Analytics (11) every lifecycle event for KPIs, routes all notifications through the Notification Hub (07), and exposes typed tools (`create_task`, `assign_task`, `get_task_status`, `schedule_reinspection`) to the Agent Runtime (10).

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-WORK-001–103 defining the 11 work-item types, all 8 scheduling modes, assignment models, multi-level approval routing, dependencies, the Today/Manager screens — plus UX states, edge cases (DST, clock skew, races), and resolved decisions. |
| [high-level-design.md](./high-level-design.md) | The Work Service's internal engines (scheduling, state machine, assignment, approval, SLA), the Temporal workflow-per-series pattern, tenant-hash partitioning of `work_items`, failure degradation (e.g. Temporal outage), and tech choices. |
| [low-level-design.md](./low-level-design.md) | Complete Postgres schema (`work_items`, schedules, approval steps, dependencies, escalation policies), API endpoints, state-machine transition rules, algorithms, caching, background jobs, and error handling. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component diagram, recurrence-to-device and SLA-escalation sequences, the multi-level approval chain sequence, ER diagram, the canonical WorkItem state machine, and the approval-step lifecycle underneath `PENDING_REVIEW`. |
