# Work Orchestration — Functional Specification

Module corresponds to **EPIC-03 (Work Core)**. Source: source-spec.md §7 (Work Orchestration),
§17.1 (WorkItem entity), §17.2 (Work State Machine), Appendix E EPIC-03.

## 1. Purpose and Scope

### 1.1 Purpose
Work Orchestration is the central domain of the platform. Every unit of frontline execution —
whatever its origin (a published form template, an audit program, a triggered corrective action, a
training assignment, a mandatory announcement, a campaign rollout, a scheduled site visit, an asset
PM cycle, a survey, or an incident report) — becomes a **WorkItem**: a single, governed, universal
abstraction with one state machine, one assignment model, one scheduling engine, and one SLA/
escalation framework. This module is what makes "every standard becomes work" (Product Promise,
source-spec §2.2) operationally real.

### 1.2 In Scope
- The `WorkItem` abstraction and its 11 supported types (§7.1) and their distinguishing behavior.
- All scheduling modes that produce WorkItem instances (§7.2): calendar recurrence,
  opening/closing-relative, shift-based, event/location/role-triggered, dependency-based,
  seasonal/campaign, plus SLA timers and escalation ladders that govern any WorkItem post-creation.
- Assignment models: individual, role-based, location-based, and queue/pool-based, including
  reassignment and mass-reassignment.
- Approval routing for WorkItems that require review (`PENDING_REVIEW → APPROVED/REJECTED`).
- Dependency-based sequencing between WorkItems (blocking/unblocking).
- The frontline "Today" home screen and the Manager Exceptions / Approvals / Team / Location Health
  views (§7.3).
- The canonical `WorkItem` state machine (§17.2) as the single lifecycle all work-item types share.
- Work-item-scoped events on the Event Backbone (§18.3) and their payload shapes.

### 1.3 Out of Scope (owned by other modules, referenced here)
- Form schema design, field-level validation logic, template versioning — **Forms module (EPIC-04)**.
  Work Orchestration only references a `TemplateVersion` when a WorkItem requires a form submission.
- Evidence capture, storage, hashing — **Evidence module (EPIC-06)**. Work Orchestration references
  `Evidence` records by ID for verification gating only.
- Issue/CAPA root-cause and corrective-action sub-workflow — **Issues & CAPA module (EPIC-07)**.
  An `Issue` may spawn one or more WorkItems (type `ISSUE_CORRECTIVE_ACTION`) but issue lifecycle
  state (OPEN → ... → CLOSED) is owned by that module.
- Course/certification content and learning outcome measurement — **Learning module (EPIC-09)**.
  Work Orchestration only owns the `TRAINING_ASSIGNMENT` WorkItem wrapper and its due/completion
  tracking.
- Notification delivery mechanics (push/email/SMS, retries, preferences) — **Notification Hub**.
  Work Orchestration emits events and calls a `send_notification`-class interface; it does not own
  delivery channels.
- Organization hierarchy, role definitions, effective-dated scope — **Identity/Org module (EPIC-01/02)**.
  Work Orchestration consumes `OrganizationNode` and `UserScope` for assignment and visibility.

## 2. Personas Served
All eight personas from source-spec §3 interact with Work Orchestration; it is the universal work
surface:

| Persona | Interaction |
|---|---|
| Enterprise Admin | Configures global escalation policy defaults, SLA policy libraries |
| Operations Admin | Designs schedules/campaigns; deploys recurring work programs to location sets |
| Regional/District Manager | Views Location Health, cross-site exceptions, escalations in territory |
| Store/Site Manager | Views Manager Workspace (Exceptions, Approvals, Team, Location health); reassigns; approves |
| Frontline Associate | Uses "Today" screen; executes and submits WorkItems, incl. offline |
| Auditor/Inspector | Executes `AUDIT_INSPECTION`/`ASSET_INSPECTION` WorkItems; may operate across locations |
| Executive | Consumes aggregated on-time/verified-completion metrics (via Analytics module) |
| AI Agent | Creates/reassigns/escalates WorkItems within policy via typed tools (`create_task`, `assign_task`, `schedule_reinspection`, `get_task_status`) |

## 3. Work Item Types and Distinguishing Behavior

All types share the canonical `WorkItem` entity and state machine (§6). Each type layers
type-specific fields and rules on top via a `type_config` (JSONB) payload validated against a
per-type JSON Schema, plus type-specific business rules below.

- **FR-WORK-001 — Task.** Generic unit of work. No mandatory sub-object. May optionally reference a
  `TemplateVersion` for a structured checklist. Default type when none of the below applies.
- **FR-WORK-002 — Audit / Inspection.** Requires a `TemplateVersion` reference (mandatory, not
  optional). Produces a `Submission` on completion. Supports a `score` and `risk_band` rollup read
  from the submission's scoring version. Typically recurring (calendar or shift-based).
- **FR-WORK-003 — Approval.** Has no "execution" step; its only transition is
  `AVAILABLE → PENDING_REVIEW → APPROVED/REJECTED`. Always created referencing a `subject_ref`
  (the entity being approved — e.g. another WorkItem, a Submission, a schedule change, a policy
  exception request). Approver is resolved via approval routing (§7).
- **FR-WORK-004 — Issue / Corrective Action.** Created by the Issues & CAPA module
  (`issue.created` event) or by rule/agent action. Carries `issue_id` FK. Its `VERIFIED` transition
  triggers `issue.resolved`-eligible check (module boundary: Work Orchestration reports completion;
  Issues module decides root Issue state).
- **FR-WORK-005 — Training Assignment.** References a `Course`/`LearningAssignment` ID (Learning
  module). Completion criteria may include a passing knowledge-check score, sourced from Learning
  module callback event `training.completed`, not from a local Submission.
  Can be system-generated in response to a repeated failure pattern (§11.3 REPEATED FAILURE loop).
- **FR-WORK-006 — Announcement Acknowledgement.** No form/evidence submission — completion is a
  single `ACKNOWLEDGE` action, timestamp + user-agent captured. Supports `expires_at`; past
  expiry an unacknowledged item transitions to `OVERDUE` but is excluded from SLA escalation ladders
  by default (acknowledgement lateness is reported, not escalated, unless tenant policy overrides).
- **FR-WORK-007 — Campaign Task.** Always created as part of a `Schedule` of `schedule_type =
  SEASONAL_CAMPAIGN` with a `campaign_id` grouping key. Supports bulk progress rollup queries by
  `campaign_id` across locations for campaign-level completion dashboards.
- **FR-WORK-008 — Site Visit.** Assignment is typically to a Regional/District Manager rather than a
  site-local user; `location_id` is the visited site while `assignee` scope may be a parent
  OrganizationNode. May bundle multiple checklist WorkItems as `dependency` children executed during
  the visit window.
- **FR-WORK-009 — Asset Inspection.** References an `Asset` entity (asset_id FK) in addition to
  `location_id`. Scheduling is frequently condition/usage-triggered (event-triggered schedule keyed
  off asset telemetry) in addition to calendar-based PM cycles.
- **FR-WORK-010 — Survey.** Similar to Audit but `is_anonymous` flag suppresses `assignee`
  attribution in analytics exports (survey responses aggregate without exposing respondent identity
  to managers by default; admin-configurable).
- **FR-WORK-011 — Incident / Observation.** Can be created ad hoc by any frontline user without a
  pre-existing schedule (`origin = AD_HOC`), unlike other types which are normally schedule- or
  event-originated. Skips `SCHEDULED` state — created directly into `AVAILABLE`. May auto-escalate
  to `Issue` creation based on severity field in its form submission.

- **FR-WORK-012.** The system must support adding new WorkItem types via configuration
  (type registration: display name, icon, allowed transitions subset, required sub-object FK,
  default SLA policy) without a schema migration, since this is an enterprise-configurable list
  in practice (tenant admins commonly ask for named sub-types, e.g. "Safety Walk", "Cash Audit").
  Implementation: a `work_item_type` catalog table (tenant-global + tenant-override rows) rather
  than a hardcoded enum only. The 11 types above ship as system-seeded catalog rows.

## 4. Scheduling

Scheduling is the mechanism that turns a `Schedule` configuration into concrete `WorkItem` rows.
Every `Schedule` has exactly one `schedule_mode` (below) and one materialization rule.

- **FR-WORK-020 — Calendar-based recurrence.** RRULE-compatible (RFC 5545 subset: `FREQ`,
  `INTERVAL`, `BYDAY`, `BYMONTHDAY`, `BYMONTH`, `COUNT`/`UNTIL`). Evaluated per `location_id` in the
  **location's configured timezone**, not tenant or server timezone.
- **FR-WORK-021 — Opening/closing-relative schedules.** Anchored to a location's operating-hours
  reference data (`location_hours` — owned by Identity/Org, referenced here) rather than a fixed
  clock time, e.g. "45 minutes before close, every operating day." Re-evaluated whenever
  operating hours change (see FR-WORK-052).
- **FR-WORK-022 — Shift-based schedules.** Anchored to shift boundaries (start/mid/end-of-shift) from
  a `shift_definition` reference (Identity/Org or a Workforce integration). One instance per shift
  instance, scoped to the users rostered on that shift where the assignment model is
  `ASSIGNEE_MODEL=ROLE` at that location.
- **FR-WORK-023 — Event-triggered schedules.** Materialize a WorkItem in direct response to a
  platform event (e.g. `asset.telemetry_threshold_breached`, `submission.completed` with a specific
  answer value, `issue.created`). Configured as `(trigger_event_type, filter_expression) →
  work_item_template`. Deduplicated by `(trigger_event_id, schedule_id)` to guarantee idempotent
  materialization on event redelivery.
- **FR-WORK-024 — Location-triggered schedules.** Fire when a location changes state relevant to
  work eligibility: newly opened/activated location, location type change, location temporarily
  closed (suppresses future instances until reopened).
- **FR-WORK-025 — Role-triggered schedules.** Fire when a `UserScope` grants a role at a location
  (e.g., new hire onboarding checklist WorkItem generated on role assignment) or revokes it
  (triggers cleanup/handover WorkItem).
- **FR-WORK-026 — Dependency-based schedules.** A WorkItem's `AVAILABLE` transition is gated on one
  or more predecessor WorkItems reaching a configured state (default `VERIFIED`, configurable to
  `SUBMITTED` or `APPROVED`). Modeled as rows in `work_item_dependencies` (§ LLD). No independent
  time trigger; availability is purely graph-driven.
- **FR-WORK-027 — Seasonal / campaign schedules.** A bounded-date-range, multi-location rollout,
  potentially one-shot (`COUNT=1` within the campaign window) or recurring within the window.
  Always tags generated WorkItems with `campaign_id` (see FR-WORK-007).
- **FR-WORK-028 — SLA timers.** Every WorkItem has a `due_at` (computed at materialization from the
  schedule's due-offset rule) and may additionally have a `sla_target_at` distinct from `due_at`
  when the tenant wants a tighter internal SLA than the displayed due date (e.g. displayed due
  "end of day," internal SLA "within 2 hours of availability"). SLA timers are implemented as
  Temporal durable timers (see HLD §6) so they survive process restarts and require no polling.
- **FR-WORK-029 — Escalation ladders.** An `EscalationPolicy` defines an ordered list of steps, each
  with `(after_minutes_past_sla, notify_targets[], action)` where `action ∈ {NOTIFY, REASSIGN,
  CREATE_ISSUE, NOTIFY_PARENT_SCOPE}`. Steps fire in order; each step's timer is independent so a
  ladder can have differently-spaced rungs (e.g., 15 min → notify assignee; 60 min → notify manager;
  240 min → auto-create Issue + notify district manager). A ladder stops firing further rungs once
  the WorkItem reaches `SUBMITTED` or later, or is explicitly acknowledged/paused by a manager with
  audit-logged reason.

- **FR-WORK-030 — Schedule materialization window.** Schedules materialize WorkItems on a rolling
  window (default: 14 days ahead, tenant-configurable 1–90 days) rather than generating the entire
  recurrence series at authoring time, to keep the table bounded and allow schedule edits to apply
  cleanly to not-yet-materialized instances (see HLD/LLD for the Temporal workflow-per-series
  pattern and recurrence materializer job).
- **FR-WORK-031 — Schedule edit semantics.** Editing a `Schedule` never mutates already-materialized
  WorkItem rows in a state at or beyond `IN_PROGRESS`. Rows still in `DRAFT`/`SCHEDULED` are
  regenerated (deleted + rematerialized) unless the edit is flagged `apply_to_existing=false`.

## 5. Assignment Models

- **FR-WORK-040 — Individual assignment.** `assignee_type=USER`, `assignee_id` = specific `User.id`.
  Visible only to that user (plus their scope-inheriting managers) on their Today screen.
- **FR-WORK-041 — Role-based assignment.** `assignee_type=ROLE`, `assignee_role_id` +
  `location_id`. Visible to *any* user holding that role at that location ("first-to-claim" or
  "all-see-all" per tenant setting `role_assignment_mode ∈ {CLAIM, BROADCAST}`). `CLAIM` transitions
  the WorkItem to `assignee_type=USER` on first claim (optimistic-concurrency guarded, §LLD).
- **FR-WORK-042 — Location-based assignment.** `assignee_type=LOCATION`, no specific role required —
  any user with any active scope at the location can see and claim it. Used for open-pool tasks
  (e.g. "whoever is on shift").
- **FR-WORK-043 — Queue-based assignment.** `assignee_type=QUEUE`, `queue_id` — used for
  cross-location pools (e.g. a roving maintenance team, a regional audit pool). Queue membership is
  a list of eligible users/roles; a claim or round-robin/least-loaded auto-assignment strategy is
  configurable per queue.
- **FR-WORK-044 — Reassignment.** Any assignment model supports manual reassignment by a user with
  `work:reassign` permission at the relevant scope. Reassignment is audit-logged
  (`before_assignee`, `after_assignee`, `reason`, `actor`) and emits `task.assigned`.
- **FR-WORK-045 — Mass reassignment on user deactivation.** When a `User` is deactivated
  (Identity/Org event `user.deactivated`), all of that user's open WorkItems
  (`AVAILABLE`/`IN_PROGRESS`/`OVERDUE`) with `assignee_type=USER` are reassigned per a configurable
  policy: default to the role/location pool that originally produced the assignment (fallback to
  manager-of-record if none), never silently dropped. This is a QA acceptance scenario in
  source-spec §24.1 ("Deactivated user cannot create new work but historical actions remain
  attributable") — historical `WorkItemAssignment` rows are preserved (append-only), only the
  *current* assignment pointer changes.

## 6. Approval Routing

### 6.1 Route Definition

- **FR-WORK-050.** An `Approval`-type WorkItem (or any WorkItem type whose flow requires review,
  e.g. an Audit template flagged `requires_approval=true`) computes its approver(s) via an
  `ApprovalRoute`: an **ordered list of one or more steps**, each
  `(step_index, approver_type ∈ {SPECIFIC_USER, ROLE_AT_LOCATION, ROLE_AT_PARENT_NODE(n_levels_up),
  MANAGER_OF_SUBMITTER}, escalate_after_minutes)`. Multi-step (multi-level) routes are a first-class,
  fully supported configuration — e.g. **Site Manager → District Manager → Regional VP** for a
  high-risk audit, or **Store Manager → Loss Prevention → Regional Director** for a shrinkage
  write-off approval. A route with a single step behaves exactly as a single-approver flow; nothing
  about single-level approval changes.
- **FR-WORK-050.1 — Sequential, not parallel.** Steps execute strictly in `step_index` order. Step
  `n+1` is not created/activated until step `n` reaches a terminal `APPROVED` decision. (Parallel/
  quorum approval — e.g. "2 of 3 regional peers" — is explicitly out of scope for this version; see
  §14 Resolved Decisions.)
- **FR-WORK-050.2 — Per-WorkItem step instantiation.** When a WorkItem enters `PENDING_REVIEW`, the
  Work Service instantiates one `WorkItemApprovalStep` row per route step (§ LLD §1.1), all in
  status `PENDING` except step 0, which is immediately resolved to a concrete approver and moved to
  `ACTIVE`. This gives a durable, queryable, auditable record of the entire approval chain for this
  specific WorkItem, independent of later edits to the shared `ApprovalRoute` template.
- **FR-WORK-050.3 — Overall WorkItem state vs. step state.** `WorkItem.status` remains
  `PENDING_REVIEW` for the *entire* duration of a multi-step route — it does not have an
  intermediate per-level WorkItem status. Progress through the chain is exposed via
  `WorkItem.current_approval_step_index` and the `WorkItemApprovalStep` rows, not via additional
  top-level WorkItem states. `WorkItem.status` only transitions to `APPROVED` when the **final**
  step in the route is approved, and to `REJECTED` immediately if **any** step rejects.

### 6.2 Step Progression

- **FR-WORK-053 — Advancing to the next step.** When the `ACTIVE` step's resolved approver submits
  an `APPROVE` decision, that step is marked `APPROVED` (decision_by, decision_at, optional comment
  recorded). If a next step exists, it is resolved (approver looked up fresh at *this* moment, not
  at WorkItem-creation time — see FR-WORK-052) and transitioned `PENDING → ACTIVE`, its own SLA/
  escalation timer starts, and the newly-active approver is notified. If no next step exists, the
  WorkItem transitions `PENDING_REVIEW → APPROVED` (`approved_at` set, `task.completed`-class event
  emitted) and all approvers in the chain are notified of the final outcome.
- **FR-WORK-054 — Rejection at any level.** A `REJECT` decision at **any** step (not only the first)
  immediately marks that step `REJECTED`, marks every remaining `PENDING` step `SKIPPED` (they never
  activate), and transitions the WorkItem `PENDING_REVIEW → REJECTED → IN_PROGRESS` per the
  canonical state machine (FR-WORK-051, unchanged). The rejection reason and rejecting approver are
  visible to the original assignee **and** to every approver whose step had already been approved
  earlier in the chain, so an upstream approver knows their approval was ultimately overturned
  downstream.
- **FR-WORK-055 — Delegation.** The resolved approver of the currently `ACTIVE` step may delegate
  their decision to another eligible user at or above their own scope level (reason required, fully
  audited as a `DELEGATED` sub-event on that step row); delegation does not create a new route step,
  it reassigns the current one.
- **FR-WORK-056 — Request more evidence.** The `ACTIVE` step's approver may return the WorkItem to
  `IN_PROGRESS` with a `request_more_evidence` reason instead of approving/rejecting outright; this
  behaves identically to a rejection for state-machine purposes (all pending steps `SKIPPED`) but is
  recorded with a distinct `decision = MORE_EVIDENCE_REQUESTED` so Analytics and the UI can
  distinguish "sent back for more information" from "rejected outright." Re-submission re-enters the
  approval route at step 0 (a fresh instantiation), not at the step that requested more evidence.
- **FR-WORK-057 — Self-approval prevention.** A resolved approver who is also the original submitter/
  assignee of the WorkItem is automatically skipped (that step auto-resolves to the next eligible
  approver per approver_type resolution, e.g. the next person up `ROLE_AT_PARENT_NODE`), and this
  auto-skip is itself an audited `AUTO_SKIPPED_SELF_APPROVAL` step event, never a silent skip.
- **FR-WORK-058 — Per-step SLA and escalation.** Each `ACTIVE` step carries its own
  `sla_target_at` (`activated_at + route_step.escalate_after_minutes`) and is governed by the same
  Temporal-backed escalation mechanism as FR-WORK-028/029/§4.2 (LLD), scoped to that one step rather
  than the WorkItem as a whole. Escalating a step (§ LLD Escalation Ladder pattern, reused) can
  notify a configured `escalation_target` (e.g. the *next* level's role, or a fixed enterprise
  escalation contact) without advancing the approval chain itself — approval authority for that step
  still rests with the originally resolved approver (or their delegate) until a decision is made.
- **FR-WORK-059 — Full audit trail.** `GET` on a WorkItem's approval history returns every step with
  its resolved approver, status, decision, decision timestamp, comment, and any delegation/auto-skip/
  escalation events on that step, in `step_index` order — sufficient on its own to answer "who
  approved this, at what level, when, and was anyone escalated in the process" for audit/compliance
  review (source-spec §21, audit-log requirement).
- **FR-WORK-051.** Rejection (`PENDING_REVIEW → REJECTED`) requires a reason and returns the
  WorkItem to `IN_PROGRESS` (per canonical state machine) with the rejection reason and rejecting
  approver visible to the original assignee. (See FR-WORK-054 for multi-step-specific rejection
  behavior, which is additive to this rule, not a replacement.)
- **FR-WORK-052.** Approval routing must re-resolve when the organization changes (a role holder at
  the target node changes) — the route stores a *resolved-at-creation* target for audit stability
  for step 0 only; every subsequent step re-resolves its approver at the moment it activates
  (FR-WORK-053), not at WorkItem-creation time, and additionally re-resolves before each escalation
  fires so escalations always reach the current role holder even if a reorg happens mid-approval.

### 6.3 Approval Queue UX (Multi-Level)

- **FR-WORK-081** (Manager Approvals queue, §9) is extended: a manager's Approvals queue shows only
  WorkItems where **that manager is the resolved approver of the currently `ACTIVE` step** — never
  earlier or later steps in the chain. Each card additionally shows a compact step-progress
  indicator (e.g. "Step 2 of 3 — awaiting you") so the approver has context on the chain without the
  queue being cluttered with items not yet or no longer actionable by them.
- Approvers whose step has already completed (`APPROVED`/`REJECTED`) can still view — read-only —
  the full chain and the current live step from a "My Approval History" view, including the eventual
  outcome even if the item continued past their step.

## 7. Dependency-Based Sequencing

- **FR-WORK-060.** A WorkItem may declare 1..N predecessor WorkItems via `work_item_dependencies`.
  It cannot transition out of `SCHEDULED`/`DRAFT` into `AVAILABLE` until all predecessors reach
  their configured unlocking state (default `VERIFIED`).
- **FR-WORK-061.** Dependency graphs are validated acyclic at authoring time (schedule/template
  configuration) and additionally guarded at materialization time (see Validation §10).
- **FR-WORK-062.** A blocked WorkItem is visible to the frontline user in a distinct `BLOCKED` UX
  state (§8) showing which predecessor(s) are outstanding, not simply hidden, so users understand
  why work has not appeared.
- **FR-WORK-063.** Cross-location dependencies are supported (e.g., a regional rollup WorkItem
  depends on all site-level WorkItems in its region reaching `VERIFIED`) using an `N-of-M` unlock
  rule (`dependency_mode ∈ {ALL, ANY, THRESHOLD(n)}`).

## 8. Frontline "Today" Home Screen (§7.3)

- **FR-WORK-070.** The Today screen is the network-independent home screen (NFR alignment with
  source-spec §23 "Mobile startup... network-independent home screen"). It renders entirely from
  the local WatermelonDB `work_items` table; no network call is required to display it.
- **FR-WORK-071.** Summary counters: `N urgent` (due within a tenant-configured urgency window,
  default 2 hours, or flagged high-priority), `N due` (remaining open work due today),
  `N overdue` (state = `OVERDUE` or `ESCALATED`).
- **FR-WORK-072.** "Next" list: ranked queue of actionable WorkItems in state
  `AVAILABLE`/`IN_PROGRESS`, ordered by (1) overdue first, (2) urgent, (3) `due_at` ascending,
  (4) manually-set `priority`. Blocked items are excluded from "Next" but visible in a separate
  "Blocked" section.
- **FR-WORK-073.** Tapping a list item opens the work item detail/execution surface (owned jointly
  with Forms module for template-backed types).
- **FR-WORK-074.** Pull-to-refresh triggers a sync-module call but never blocks initial render.

## 9. Manager Views (§7.3)

- **FR-WORK-080 — Exceptions.** List of WorkItems in `OVERDUE`/`ESCALATED`/`BLOCKED` (long-blocked
  beyond a threshold) across the manager's scope, plus `Issue` exceptions (via Issues module link).
  Filterable by location, type, severity, assignee.
- **FR-WORK-081 — Approvals.** Queue of WorkItems in `PENDING_REVIEW` where the manager is the
  resolved approver, supporting inline approve/reject/request-more-evidence actions and bulk
  approve for low-risk batches (tenant policy gated).
- **FR-WORK-082 — Team.** Roster view of direct/indirect reports (per OrganizationNode scope) with
  per-person open/overdue/completed-today counts; supports drill-in and bulk reassignment.
- **FR-WORK-083 — Location Health.** Aggregated on-time/verified-completion rate, open exception
  count, and trend for each location in scope — the manager-scope slice of the Control Tower
  (full cross-tenant Control Tower is owned by the Analytics module; this view is the
  operational, near-real-time subset backed directly by Work Service data, not the warehouse).
- **FR-WORK-084 — Bulk actions.** Managers with `work:bulk_action` permission can select multiple
  WorkItems in Exceptions/Team views and apply: reassign, extend due date (with reason, audit
  logged), escalate manually, or cancel (only for `DRAFT`/`SCHEDULED` items). Bulk actions execute
  as a single idempotent batch request (`Idempotency-Key` covers the whole batch) and return a
  per-item result array so partial failures are visible.

## 10. UX States

| State | Behavior |
|---|---|
| Today — empty | "All caught up" state distinct from "no work assigned"; shows next scheduled item's due time if one exists within 24h, otherwise a neutral empty illustration/copy. Never shown while sync is in-flight — that is a distinct loading state. |
| Today — loading | Only shown on true cold start before local DB is hydrated (first app run); subsequent loads always render from cache instantly per FR-WORK-070. |
| Today — offline | No distinct "offline" empty/error state per Offline Design Rule (source-spec §9.2) — the same list renders; a small persistent (non-blocking) sync-status indicator shows "offline — N changes pending" when the outbound queue is non-empty. |
| Overdue badge | Two visually distinct badge states: `OVERDUE` (amber, still actionable) vs `ESCALATED` (red, indicates escalation ladder has fired at least one rung) — never collapse them into one visual treatment, since escalation implies a manager/other party has already been notified. |
| Blocked-item UX | Shown in a separate "Blocked" section on Today with the specific unmet predecessor(s) named (not just "blocked"); tapping shows the dependency chain and (if the user has visibility) the predecessor's current state. |
| Reassignment flow | Manager selects item(s) → picks new assignee (typeahead scoped to eligible users/roles at that location) → optional reason (required for cross-role reassignment) → confirm → optimistic UI update with rollback on server rejection (e.g., target user lacks scope). |
| Bulk action (manager) | Multi-select checkbox mode on list views; action bar appears; confirmation dialog states item count and, for reassignment, requires a single target; result summary shown after completion (`N succeeded, M failed` with failed-item drill-in). |
| Permission-denied | A WorkItem visible in a list (e.g. cross-location Exceptions rollup for a district manager) but not actionable at click-through shows a read-only detail view with an explicit "You do not have permission to act on this item at this location" message rather than a generic error. |
| Error (submit failure) | Local draft is preserved; user sees a retry affordance; underlying reason surfaced only if actionable (e.g. "template version superseded, refresh required") — see Validation below. |
| Multi-level approval — step progress | Approval detail view (mobile and web) renders a horizontal/vertical stepper (e.g. "Site Manager ✓ → District Manager (awaiting) → Regional VP") with completed steps showing decision-maker name, decision timestamp, and comment; the active step highlighted; future steps shown greyed/pending with their resolved-approver-type (not yet a named person until activated, unless `approver_type=SPECIFIC_USER`). |
| Multi-level approval — not your turn | An approver viewing a WorkItem whose active step belongs to someone else (e.g. they approved step 1, item is now at step 2) sees a read-only "Approved by you on {date}. Currently awaiting {step 2 approver}." banner instead of action buttons. |
| Multi-level approval — escalated mid-chain | A step that has bred an escalation shows a distinct badge ("Escalated — no response from {approver} in {N}h") on the approver-facing queue card and on the submitter's tracking view, separate from the generic `ESCALATED` WorkItem-level badge (§ Overdue badge row above), since here the WorkItem itself is not overdue — only one step's SLA has lapsed. |

## 11. Validation and Edge Cases

- **FR-WORK-090 — Schedule collision.** Two active schedules producing overlapping instances for the
  same `(location_id, work_item_type, template_version_id)` within the same due window are flagged
  at authoring time (Impact Preview-style dry run) and, if allowed to proceed, materialize both
  instances but surface a "duplicate work" indicator to the assignee (not silently deduplicated,
  since business intent for two similar-but-distinct schedules may be legitimate — e.g., corporate
  audit + local audit on the same day). Tenant setting `dedupe_identical_schedules=true` collapses
  exact duplicates (identical type+template+location+time) at materialization time.
- **FR-WORK-091 — DST-crossing schedules.** Calendar/shift schedules are evaluated against
  IANA timezone rules at the location's `timezone` field. A daily recurrence at a wall-clock time
  (e.g., "08:00 local") always fires once per calendar day even across a DST transition (i.e., the
  scheduler computes wall-clock-correct instants, not fixed UTC offsets). A schedule expressed as a
  fixed interval from a triggering event (e.g., "every 8 hours from last completion") uses actual
  elapsed duration and is unaffected by DST by construction.
- **FR-WORK-092 — Timezone-per-location scheduling.** `OrganizationNode`/location carries an
  authoritative `timezone` (IANA id). All `due_at`/`sla_target_at` computation, "Today" boundary
  calculation, and escalation-ladder timer offsets are computed in that location's timezone, not
  the acting user's device timezone or tenant default — critical for traveling managers/auditors
  who must see a site's "today" as that site's today, not their own.
- **FR-WORK-093 — Dependency cycles.** Rejected at configuration time with a `422
  DEPENDENCY_CYCLE_DETECTED` error identifying the cycle path; additionally guarded at
  materialization time (defensive check) since dependencies can be authored across templates/
  schedules that are individually acyclic but jointly cyclic once combined for a location.
- **FR-WORK-094 — Mass reassignment on user deactivation.** See FR-WORK-045; edge case: if the
  fallback role/location pool for a deactivated user's WorkItem no longer exists (e.g. the role was
  also deleted), the item routes to a `manager_of_record` fallback chain resolved via
  `OrganizationNode` parent walk; if no fallback resolves at all, the item is flagged
  `NEEDS_ASSIGNMENT` and surfaced in the manager Exceptions view rather than left with a dangling
  assignee reference.
- **FR-WORK-095 — Stale template version at submission time.** If a `TemplateVersion` referenced by
  an in-progress WorkItem is superseded before submission, the WorkItem continues against the
  original (immutable) version it was materialized against (source-spec §17.2/§8.3 immutability
  principle) — the newer version applies only to WorkItems materialized after publish.
- **FR-WORK-096 — Clock skew / offline completion ordering.** Device-reported completion timestamps
  are recorded but never authoritative for SLA/escalation computation; server receipt time plus
  the client's `created_at_client` are both stored, with server time used for all SLA math (per
  source-spec §19.4 "client clock cannot be trusted for authoritative ordering").
- **FR-WORK-097 — Duplicate offline completion submission.** Idempotent on `client_uuid`; a retried
  sync of the same completion returns the original server outcome, never a duplicate state
  transition or duplicate event emission.
- **FR-WORK-098 — Location closure mid-schedule.** A location marked temporarily closed cancels
  future materialization but does not retroactively cancel already-materialized, in-flight
  WorkItems unless an admin explicitly bulk-cancels them.
- **FR-WORK-099 — Role/location assignment race.** Under `CLAIM` mode (FR-WORK-041), concurrent
  claim attempts are resolved via optimistic concurrency on the WorkItem `version` column; the
  loser receives `409 CONFLICT` with a `already_claimed` reason and the current assignee, not a
  generic concurrency error, so the client UI can immediately reflect "claimed by X."
- **FR-WORK-100 — Decision on a non-active step.** A decision submitted against a step that is not
  currently `ACTIVE` (e.g. a stale mobile client retrying an approval after the chain has already
  advanced or been rejected by a later/earlier actor) is rejected with `409
  APPROVAL_STEP_NOT_ACTIVE` and the current chain state, never silently accepted or double-applied.
- **FR-WORK-101 — Approver becomes ineligible mid-chain.** If the resolved approver of an `ACTIVE`
  step is deactivated or loses the qualifying role/scope while their step is pending, the step is
  immediately re-resolved (same resolution logic as FR-WORK-052) to the next eligible holder of that
  `approver_type`; if none can be resolved, the step — and therefore the whole WorkItem — is flagged
  `NEEDS_ASSIGNMENT` and surfaced in the manager Exceptions view (same treatment as FR-WORK-094),
  rather than silently stalling with no eligible approver.
- **FR-WORK-102 — Route edited while a WorkItem is mid-chain.** Editing the shared `ApprovalRoute`
  template (e.g. adding a step) never mutates the `WorkItemApprovalStep` rows already instantiated
  for in-flight WorkItems (source-spec §17.4-style immutability principle applied to approval
  chains) — only WorkItems that enter `PENDING_REVIEW` after the route edit use the new step
  sequence. This is enforced by copying the route's step definition onto each `WorkItemApprovalStep`
  at instantiation time (FR-WORK-050.2) rather than the step rows referencing the mutable route by
  ID + index alone.
- **FR-WORK-103 — All steps' approvers resolve to the same person.** If step *n*'s resolution would
  produce the identical person as an already-`APPROVED` earlier step in the same chain (not only the
  submitter, per FR-WORK-057), that step is likewise auto-skipped with an audited
  `AUTO_SKIPPED_DUPLICATE_APPROVER` event — a person does not approve the same WorkItem twice under
  two different role hats in one chain.

## 12. Non-Functional Requirements (module-specific)

- **NFR-WORK-01 — Creation-to-device latency.** A WorkItem created or reassigned server-side (via
  schedule materialization, event trigger, agent action, or manual admin action) must be visible on
  the affected user's device within: p95 ≤ 5 seconds when the device is online and foregrounded or
  backgrounded with push connectivity; p95 ≤ 30 seconds via push-triggered background sync when
  backgrounded (aligned with the general Sync recovery NFR in source-spec §23); on next
  foreground/manual sync otherwise, with no hard latency budget for fully offline devices (best
  effort on reconnect, per §9.2 Offline Design Rule).
- **NFR-WORK-02 — Scheduling throughput.** The materializer must sustain generating the rolling
  materialization window (FR-WORK-030) for tenants with 100k locations × 10+ recurring workflows/
  location without falling behind its schedule (materialization run completion strictly faster than
  the materialization interval — see LLD background jobs).
- **NFR-WORK-03 — SLA timer accuracy.** Escalation ladder rungs fire within ±60 seconds of their
  configured offset at p99, independent of Work Service restarts (durable Temporal timers, not
  in-memory schedulers).
- **NFR-WORK-04 — Today-list read latency.** Server-side "my work" list endpoint (used for
  reconciliation and web admin) p95 < 300 ms per the platform-wide API performance NFR.
- **NFR-WORK-05 — Read availability under Temporal outage.** WorkItem creation, status transitions,
  and "Today" reads must continue to function (degraded scheduling/escalation only) if Temporal is
  unavailable — see HLD §Failure Modes.
- **NFR-WORK-06 — Approval step advancement latency.** The time from an approver's decision being
  recorded to the next step's approver being notified (or, on the final step, to the WorkItem
  reaching `APPROVED` and the original assignee being notified) must be p95 ≤ 5 seconds — a
  multi-level approval chain must not feel slower per hop than a single-approver flow.

## 13. Dependencies on Other Modules

| Module | Direction | Nature |
|---|---|---|
| Identity/Org | Consumes | `OrganizationNode` (location, timezone, hierarchy), `UserScope`/`Role` for assignment eligibility and visibility; consumes `user.deactivated`, `user.role_changed` events |
| Forms | Consumes | `TemplateVersion` reference for form-backed WorkItem types; consumes `template.published` event to know which templates are eligible for new schedules |
| Evidence | Consumes | Evidence completeness check as a gate on `SUBMITTED`/`VERIFIED` transitions where template requires evidence |
| Issues & CAPA | Bidirectional | Consumes `issue.created` to spawn corrective-action WorkItems; produces WorkItem completion signals the Issues module uses to progress Issue state |
| Learning | Bidirectional | Produces `TRAINING_ASSIGNMENT` WorkItems on repeated-failure trigger; consumes `training.completed` to close the loop |
| Notification Hub | Produces (calls) | Escalation and assignment notifications routed through Notification Hub, not delivered directly |
| Agent Runtime | Bidirectional | Exposes `create_task`, `assign_task`, `get_task_status`, `schedule_reinspection` as typed tools; consumes `agent.action_approved` for agent-initiated WorkItem actions |
| Analytics | Produces | All WorkItem lifecycle events feed the CDC/event pipeline for KPI computation (On-Time Completion Rate, Verified Operational Execution Rate, etc.) |

## 14. Resolved Decisions

- **Urgency window for the "urgent" Today counter (FR-WORK-071)**: Fixed at **2 hours before
  `due_at`**, tenant-configurable within a **30-minute to 4-hour** band. 2 hours is the firm platform
  default because it gives a frontline associate enough lead time to act within a single shift
  without flooding the Today view with items that are merely due later the same day.
- **`Announcement Acknowledgement` lateness escalation**: Does **not** escalate by default — a late
  acknowledgement is reporting-only (visible on the manager's compliance view) and never triggers an
  escalation notification or an `OVERDUE`/`ESCALATED` transition. A tenant may opt in to escalation
  per announcement via the same escalation-ladder configuration used for `WorkItem` (FR-WORK-060),
  because most acknowledgements are informational and escalating all of them by default would create
  alert fatigue that trains managers to ignore escalations generally.
- **Queue-based assignment default strategy (FR-WORK-043)**: Defaults to **least-loaded** (fewest
  current open items assigned to the candidate), tenant-configurable per queue to round-robin or
  skills-weighted. Least-loaded is the firm default because it directly optimizes the platform's
  On-Time Completion Rate KPI by avoiding assignment pile-up on a single associate, which round-robin
  does not account for.
- **`work_item_type` catalog extensibility (FR-WORK-012)**: The 11 system types are non-deletable but
  tenant-hideable. Tenants may define additional sub-types, each of which must inherit the full
  state-machine behavior of exactly one of the 11 base types — a sub-type is a display/configuration
  veneer (label, icon, default SLA, default queue) only, never a new state-machine variant. This is
  firm: allowing sub-types to alter transition behavior would break the platform-wide guarantee that
  every `WorkItem` follows one of the 11 documented lifecycles (conventions.md state machine table).
- **Cross-tenant queue pools**: Not supported in this version. A queue's candidate pool is always
  scoped to a single tenant's `UserScope` set; a third-party facilities vendor serving multiple
  tenants is handled by provisioning that vendor's staff as scoped users inside each tenant they
  serve (via Identity & Organization's normal multi-tenant user model), not via a shared cross-tenant
  pool. This keeps the tenant isolation boundary (engineering-decisions.md, data residency) intact
  without an exception carve-out; true cross-tenant resource sharing is deferred to a future
  Integration Hub capability if a concrete customer need materializes.
- **Parallel/quorum approval**: Out of scope for this version. Only sequential multi-level approval
  routes (FR-WORK-050.1) are built. The `WorkItemApprovalStep` schema reserves a nullable
  `step_group_id` column specifically so a future parallel-fan-out or quorum ("2 of 3") mode can be
  added without a breaking schema migration, but no parallel/quorum evaluation logic ships now,
  because no current functional requirement calls for it and building unused branching logic into the
  approval state machine would add untested surface area to a compliance-critical path.
- **Approval escalation timeout behavior**: A route step's `escalate_after_minutes` timeout sends an
  escalation notification only — it never auto-approves or auto-rejects. Advancing a stalled
  multi-level approval always requires an explicit human decision, or an agent decision at an
  autonomy level with approval authority (per the Agent Runtime's autonomy-level model). This is firm
  for this version: auto-advancing an approval on timeout would silently weaken the approval control
  it exists to enforce. A tenant-configurable "auto-escalate to next approver after timeout" policy
  (which still requires that next approver's affirmative action, not an auto-approve) is deferred to
  a future version and is not built now.
- **Location Health data source (FR-WORK-083)**: Served directly from Work Service's own
  transactional data (not the analytics warehouse) so it stays near-real-time for manager workflows;
  Work Service exposes a lightweight aggregation read path for this purpose (low-level-design.md
  caching strategy). The Analytics module's Control Tower remains the authoritative source for
  historical/trend views. This split is firm: a warehouse-backed view would introduce CDC pipeline
  lag unacceptable for an in-shift manager decision, while the transactional path is unsuitable for
  multi-quarter trend analysis at scale.
