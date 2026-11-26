# Issues & CAPA — Functional Specification

Module corresponds to **EPIC-07 (Issues & CAPA)**. Source: source-spec.md §10.2-10.4 (Issue
Lifecycle, CAPA, Root Cause Methods), §17.1 (Issue entity), §17.3 (Issue State Machine),
Appendix A (Golden End-to-End Scenario), Appendix B (Example Rule Specification), Appendix E
EPIC-07. Stays fully consistent with `conventions.md` (Issue state machine reused verbatim,
Temporal for SLA/escalation, Postgres for transactional state, Kafka event backbone, API
conventions).

## 1. Purpose and Scope

### 1.1 Purpose
Issues & CAPA is the exception-management domain of the platform: the system of record for every
managed deviation from an operating standard, from detection through root cause analysis,
corrective and preventive action, effectiveness verification, and closure. It operationalizes the
Product Promise line "Every deviation can become a managed exception. Every repeated exception can
become an automated intervention" (source-spec §2.2), and is the backbone of the
`EXCEPTION -> CORRECT -> RE-VERIFY -> LEARN` segment of the core product loop. This module owns the
`Issue` entity (source-spec §17.1) and the canonical `Issue` state machine (§17.3), which is reused
verbatim by every other module that references issue state (Work Orchestration, Analytics, Agentic
Ops).

### 1.2 In Scope
- Issue creation from all sources: manual (frontline/manager-reported), rule-triggered (deterministic
  rule engine evaluating a form submission, per Appendix B), escalated from an overdue/failed
  `WorkItem`, and agent-created (Resolution Agent / Compliance Monitor Agent via `create_issue` tool).
- Triage and severity classification (initial and re-triage).
- Assignment and ownership transfer, including reassignment on organizational change.
- The full CAPA step model: immediate correction, containment, root cause analysis, corrective
  action, preventive action, effectiveness verification, reopening.
- Structured data models for all five root cause methods: 5 Whys, Fishbone (Ishikawa), Pareto,
  FMEA, and tenant-defined custom methods.
- Effectiveness verification scheduling and outcome recording, including re-inspection linkage.
- Reopening rules and the `RESOLVED/VERIFICATION -> REOPENED -> INVESTIGATING` branch.
- SLA policy per severity and escalation ladder integration (via Work Orchestration's Temporal-based
  escalation infrastructure, keyed off `Issue` rather than `WorkItem` state).
- Issue list, kanban-by-stage, and root-cause-method-picker UX states; evidence attachment at each
  CAPA step.
- `issue.created` / `issue.escalated` / `issue.resolved` events and their payload contracts.

### 1.3 Out of Scope (owned by other modules, referenced here)
- The deterministic rule engine that evaluates form submissions and decides `create_issue: true`
  (Appendix B) — owned by the **Form Platform module (EPIC-04)**; this module only consumes the
  resulting `issue.created` trigger contract and owns everything from issue creation onward.
- The `WorkItem` created for a corrective/preventive action task (type `ISSUE_CORRECTIVE_ACTION`),
  its assignment queueing, due-date tracking, and completion UX — owned by **Work Orchestration
  (EPIC-03)**. Issues & CAPA creates and references these WorkItems by ID but does not own their
  state machine; it listens for `task.completed` to progress the CAPA step.
- Evidence capture, storage, hashing, retention — **Evidence module (EPIC-06)**. This module
  references `Evidence` records by ID at each CAPA step.
- Training/microlearning content, course assignment mechanics, certification — **Learning module
  (EPIC-09)**. This module creates a `LearningAssignment` reference when a corrective/preventive
  action is training-linked but does not own course content or completion tracking.
- Agent reasoning, tool policy, and autonomy-level enforcement for the Compliance Monitor and
  Resolution Agents — **Agent Runtime (EPIC-11)**. This module exposes the typed tools
  (`create_issue`, `get_recent_failures`, `schedule_reinspection`) the agents call and is the
  authoritative state owner when they act, but does not own agent policy or approval routing.
- Notification delivery mechanics — **Notification Hub / Communications module (EPIC-08)**. This
  module emits events and calls `send_notification`-class interfaces only.
- Enterprise/regional/executive rollups of exception KPIs (Repeat Failure Rate, Time to
  Intervention, Exception Resolution Time) — **Analytics & Control Tower (EPIC-12/15)**, consuming
  this module's events and a read-replica/CDC feed.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Store/Site Manager | Primary issue owner at the location: triages, assigns, executes CAPA steps, submits root cause analysis, requests verification, closes or reopens. |
| Regional/District Manager | Monitors exceptions and CAPA health across territory; reassigns across sites after reorg; escalation-ladder recipient for unresolved critical issues. |
| Operations Admin | Defines CAPA templates (which steps/fields are mandatory per issue category/severity), SLA policy per severity, escalation ladders, and root-cause-method availability per tenant. |
| Quality/Compliance role | Reviews closed CAPA records for audit defensibility; runs repeat-failure/pattern analysis; approves method changes mid-investigation for regulated categories. |
| Frontline Associate | Reports an issue ad hoc (source `MANUAL`); may be assigned an immediate-correction or containment step. |
| Auditor/Inspector | Source of rule-triggered issues from audit/inspection submissions; may perform effectiveness verification re-inspections. |
| AI Agent (Compliance Monitor, Resolution Agent) | Detects repeat-failure patterns and creates issues (`create_issue` tool, source `AGENT`); proposes corrective/preventive actions (L1 Recommend by default); schedules re-inspection (`schedule_reinspection`). |

## 3. Issue Creation Sources

- **FR-CAPA-001 — Manual creation.** Any authenticated user with `issue:create` scope on a location
  may create an Issue directly (mobile or web). Required fields at creation: `title`, `description`,
  `location_id`, `category` (tenant-configured taxonomy, e.g. `FOOD_SAFETY`, `EQUIPMENT`, `SAFETY`,
  `BRAND_STANDARD`, `OTHER`), and at least one initial evidence reference or a documented reason for
  its absence. Source = `MANUAL`. Initial state = `OPEN`.
- **FR-CAPA-002 — Rule-triggered creation.** The Form Platform's rule engine evaluates a submission
  against tenant rules (Appendix B shape: `trigger`, `condition`, `actions` including
  `create_issue: true`). On a matching rule, Form Platform calls this module's internal
  `POST /internal/issues` (service-to-service, not the public API) with `source = RULE_TRIGGERED`,
  `rule_id`, `submission_id`, `template_version_id`, computed `severity` seed (from `set_risk`
  action), and any `require_field` values captured on the submission (e.g. `corrective_action` free
  text) copied into the Issue's `initial_finding` field. Idempotency: keyed on
  `(tenant_id, rule_id, submission_id)` — a replayed rule evaluation must not create a duplicate
  Issue (see FR-CAPA-041 duplicate detection).
- **FR-CAPA-003 — Escalated from a WorkItem.** When a `WorkItem` transitions to `OVERDUE ->
  ESCALATED` (source-spec §17.2) and the tenant's escalation policy for that work-item type
  specifies `escalate_to_issue: true`, Work Orchestration emits `task.overdue` with an escalation
  flag; this module creates an Issue with `source = WORK_ESCALATION`, `escalated_work_item_id` FK,
  and severity derived from the WorkItem's configured risk weight.
- **FR-CAPA-004 — Agent-created.** The Compliance Monitor Agent, operating at autonomy L1/L2 per its
  policy (Appendix C), calls the typed `create_issue` tool after detecting a recurring or contextual
  failure pattern (see LLD repeat-failure pseudocode). Source = `AGENT`, with mandatory
  `agent_run_id` FK for audit traceability into `AgentRun`. Severity `CRITICAL` issues created by an
  agent require human approval per the agent's policy (`approval_required_for: severity=CRITICAL`)
  before the Issue leaves a provisional `PENDING_AGENT_APPROVAL` sub-state gating entry into `OPEN`
  (implementation: an `Issue.approval_status` column; the Issue row exists but is not visible in
  manager queues nor SLA-timed until approved, to avoid a phantom SLA clock on a rejected
  recommendation).
- **FR-CAPA-005.** Every Issue, regardless of source, is created inside a single transactional
  Postgres write plus an `issue.created` outbox event (transactional outbox pattern) so the event is
  never lost nor duplicated relative to the row.

## 4. Triage and Severity Classification

- **FR-CAPA-010.** On creation, every Issue receives a severity from the set `LOW`, `MEDIUM`,
  `HIGH`, `CRITICAL`. Rule-triggered and agent-created issues arrive with a seed severity (from the
  rule's `set_risk` action or the agent's confidence-weighted classification); manual issues default
  to `MEDIUM` pending triage.
- **FR-CAPA-011 — Triage transition (`OPEN -> TRIAGED`).** A user with `issue:triage` scope
  (typically the Store/Site Manager or an Operations Admin acting centrally) confirms or overrides
  severity, confirms/edits `category`, and may attach a `triage_note`. Triage is mandatory before
  assignment; the system blocks `ASSIGNED` transition without a completed triage record
  (`triaged_by`, `triaged_at`, `severity` confirmed non-null).
- **FR-CAPA-012 — Auto-triage.** Tenant policy may configure auto-triage for specific rule-triggered
  categories (e.g. temperature exceptions per Appendix A/B): the system auto-confirms severity and
  transitions `OPEN -> TRIAGED` immediately, attributing `triaged_by = SYSTEM`, when the rule
  specifies a deterministic severity and the category is on the tenant's auto-triage allowlist. This
  keeps the Golden Scenario (Appendix A) moving without a manual step gating a mandatory corrective
  action.
- **FR-CAPA-013 — Re-triage.** Severity may be revised after `TRIAGED` (e.g. investigation reveals
  greater impact). A re-triage emits `issue.severity_changed` (internal event, not in the canonical
  list but following the same envelope) and recalculates the SLA clock per FR-CAPA-060; it does not
  reset the lifecycle state.

## 5. Assignment and Ownership Transfer

- **FR-CAPA-020 — Assignment (`TRIAGED -> ASSIGNED`).** An Issue must have exactly one `owner_user_id`
  (individual accountability; unlike WorkItem's role/queue assignment options, an Issue's `owner` is
  always a specific person, though a `contributor_user_ids[]` list supports collaborators). Default
  assignee: the location's Store/Site Manager (resolved via `UserScope` at `location_id`), overridable
  at assignment time.
- **FR-CAPA-021 — Reassignment.** Any state from `ASSIGNED` through `VERIFICATION` supports
  reassignment by the current owner, the owner's manager (parent `OrganizationNode` scope), or an
  Operations Admin. Reassignment does not change lifecycle state; it records an `issue_events` row
  (`REASSIGNED`, `from_owner`, `to_owner`, `actor`, `reason`).
- **FR-CAPA-022 — Ownership after organizational change.** When a location's manager assignment
  changes (org reorg, `UserScope` end-dated), open Issues owned by the departing user are not
  auto-reassigned silently. A nightly job (`orphaned-issue-owner sweeper`, LLD §Background Jobs)
  flags Issues whose `owner_user_id` no longer has an active `UserScope` covering the Issue's
  `location_id`, sets `owner_status = ORPHANED`, and creates a notification to the new location
  manager and the next org level up. An orphaned Issue keeps its last state and SLA clock running;
  it does not auto-pause, since the deviation itself is unaffected by the reorg (see Edge Cases
  §9.1).

## 6. CAPA Steps

Each CAPA step below is modeled as a `CapaStep` row (LLD §Schema) scoped to one Issue, with a
`step_type`, ordered `sequence`, `status`, required fields, and zero or more `Evidence` references.
Tenant admins define, per issue `category` and `severity`, which steps are mandatory vs. optional
via a `capa_template` configuration (see FR-CAPA-050).

- **FR-CAPA-030 — Immediate correction.** Available from `OPEN` onward, independent of lifecycle
  state (an immediate fix should not wait for formal triage). Required fields: `action_taken`
  (text), `performed_by`, `performed_at`, at least one Evidence reference when `category` requires
  photographic proof (tenant-configurable per category). Does not itself advance lifecycle state.
- **FR-CAPA-031 — Containment.** Required when `severity` is `HIGH`/`CRITICAL` or category is
  configured as containment-required (e.g. food safety, safety incidents). Required fields:
  `containment_action`, `scope_contained` (e.g. "all units from lot X pulled"), `performed_by`,
  `performed_at`, Evidence. Containment must be recorded before `INVESTIGATING -> CORRECTIVE_ACTION`
  transition for containment-required categories (hard gate).
- **FR-CAPA-032 — Root cause analysis.** Entered during `INVESTIGATING`. See §7 for the five
  structured method data models. Exactly one `RootCauseAnalysis` record is active per Issue, but its
  `method` may be changed mid-investigation (FR-CAPA-046) producing a new record version. Required
  before `INVESTIGATING -> CORRECTIVE_ACTION`: a `RootCauseAnalysis` with `status = COMPLETED` and
  at least one identified `root_cause_statement`.
- **FR-CAPA-033 — Corrective action.** Entered during `CORRECTIVE_ACTION`. One or more corrective
  actions may be defined; each spawns a Work Orchestration `WorkItem` (type
  `ISSUE_CORRECTIVE_ACTION`) via internal API when `requires_field_work = true`, or is recorded
  directly as a completed administrative action otherwise. Required fields: `description`,
  `target_completion_date`, `owner` (may differ from Issue owner), linkage to the root cause it
  addresses (`root_cause_analysis_id`).
- **FR-CAPA-034 — Preventive action.** Optional but strongly recommended (tenant-configurable as
  mandatory for `CRITICAL` severity or specific categories); addresses systemic recurrence risk
  rather than the immediate instance. Same required-field shape as corrective action, plus an
  optional `training_assignment_id` FK when the preventive action is training-linked (Learning
  module dependency, §11).
- **FR-CAPA-035 — Effectiveness verification.** See §8.
- **FR-CAPA-036 — Reopening.** See §9.
- **FR-CAPA-037.** Every CAPA step transition is recorded as an immutable `issue_events` row
  (append-only), independent of the mutable `capa_steps` current-state row, to preserve full audit
  history per Product Principle P7 (Historical truth).

## 7. Root Cause Methods — Structured Data Models

- **FR-CAPA-040 — Method selection.** At root cause entry, the user picks one of `FIVE_WHYS`,
  `FISHBONE`, `PARETO`, `FMEA`, `CUSTOM`. Tenant admins configure which methods are available per
  category (e.g. `FMEA` may be restricted to Quality/Compliance role for regulated categories). Each
  method's structured payload is stored in `root_cause_method_data` as a polymorphic JSONB payload
  validated against a per-method JSON Schema (LLD §Schema), with common fields
  (`analysis_id`, `method`, `created_by`, `created_at`) factored into the parent
  `root_cause_analyses` row.
  - **5 Whys**: ordered list of `{ level: 1..N, question, answer }`, terminating in a
    `root_cause_statement`. Minimum 3 levels enforced (a single "why" is not accepted as root cause).
  - **Fishbone (Ishikawa)**: `categories[]` (default 6M: Man, Machine, Method, Material,
    Measurement, Mother Nature/Environment; tenant-editable category set), each with `causes[]`
    (free text + optional weight/likelihood), and a `selected_root_cause_id` pointing at the
    specific cause entry chosen as root cause.
  - **Pareto**: `factors[]` of `{ factor, frequency }` used to rank contributing factors by
    frequency; system computes cumulative-percentage and 80/20 cut line; `root_cause_statement`
    must reference a factor above the cut line (soft validation warning if not, hard block
    configurable per tenant).
  - **FMEA**: `failure_modes[]` of `{ failure_mode, effect, severity(1-10), occurrence(1-10),
    detection(1-10), rpn (computed = severity*occurrence*detection) }`, sorted by `rpn` descending;
    `root_cause_statement` must reference the highest-RPN failure mode or provide an explicit
    override justification.
  - **Custom**: tenant-defined JSON Schema registered per category via Operations Admin
    configuration; the module validates submitted payloads against the tenant's registered schema at
    submit time and stores it opaquely (search/reporting on custom fields is best-effort, exposed via
    a generic key-value index rather than method-specific columns).
- **FR-CAPA-041 — Duplicate issue detection.** At creation (all sources) and at root-cause-analysis
  time, the system runs a similarity check against open Issues at the same `location_id` (and
  optionally sibling locations for a systemic pattern) within a configurable lookback window
  (default 7 days), matching on `category` + fuzzy title/description similarity + (for
  rule-triggered) same `rule_id`. A match surfaces as a non-blocking `possible_duplicate_of`
  suggestion in the UI, never an automatic merge (P7 auditability — merging would erase a distinct
  event's evidence trail). A user may explicitly link one Issue as a duplicate of another
  (`duplicate_of_issue_id`), which auto-closes the duplicate with a `CLOSED` reason of
  `DUPLICATE` while preserving its full history.
- **FR-CAPA-046 — Root cause method changed mid-investigation.** Changing `method` after a
  `RootCauseAnalysis` has been started creates a new `root_cause_analyses` row
  (`version = previous + 1`, `superseded_analysis_id` FK back to the prior version) rather than
  mutating the existing payload in place. The prior version is retained, not deleted, so any
  corrective action already linked to it remains traceable. The Issue's "active" analysis is always
  the highest-version row with `status != SUPERSEDED`.

## 8. Effectiveness Verification and Re-Inspection Scheduling

- **FR-CAPA-050 — Verification requirement.** Every Issue must pass through `VERIFICATION` before
  `RESOLVED`. The `EffectivenessVerification` record specifies `method` (`RE_INSPECTION`,
  `DOCUMENT_REVIEW`, `METRIC_THRESHOLD`, `MANAGER_ATTESTATION`), `scheduled_for` (date/time or
  relative offset, e.g. "7 days after corrective action completion" — tenant/category-configurable
  default), `verifier_user_id` (must differ from the Issue owner for `HIGH`/`CRITICAL` severity —
  segregation-of-duties rule), and an outcome of `PASSED` or `FAILED` with supporting Evidence.
- **FR-CAPA-051 — Re-inspection scheduling.** When `method = RE_INSPECTION`, the system creates a
  Work Orchestration `WorkItem` (type `ASSET_INSPECTION` or `AUDIT_INSPECTION` depending on category)
  scheduled for `scheduled_for`, referencing `verification_id`; the agent tool `schedule_reinspection`
  performs the same action for agent-initiated verification scheduling. Verification cannot be marked
  `PASSED`/`FAILED` until the linked WorkItem reaches `SUBMITTED`/`VERIFIED`, unless the verification
  `method` does not require field work.
- **FR-CAPA-052 — Passing verification (`VERIFICATION -> RESOLVED`).** Requires outcome `PASSED`,
  `verified_by`, `verified_at`, and Evidence. Auto-transitions to `RESOLVED`; a separate,
  time-boxed `CLOSED` transition (FR-CAPA-053) follows either automatically after a configurable
  grace period (default 3 days, allowing late objections) or immediately if tenant policy sets
  `auto_close_on_resolve = true`.
- **FR-CAPA-053 — Closing (`RESOLVED -> CLOSED`).** Manual or auto (per above). Once `CLOSED`, the
  Issue and all its CAPA step and root-cause records become immutable except for the reopening
  branch (§9). `CLOSED` requires no further fields beyond `closed_by`/`closed_at`
  (`SYSTEM` for auto-close).

## 9. Reopening Rules

- **FR-CAPA-060 — Reopening on failed verification.** If `EffectivenessVerification.outcome =
  FAILED`, the Issue transitions `VERIFICATION -> REOPENED -> INVESTIGATING` automatically (per the
  canonical state machine's documented branch). The failed verification record is retained; a new
  `root_cause_analyses` version is expected (the prior root cause is presumed incomplete or
  incorrect) but not force-created — the owner may instead add a new corrective action against the
  existing root cause if the verification failure indicates incomplete execution rather than wrong
  diagnosis (`reopen_reason` enum: `ROOT_CAUSE_INCORRECT`, `CORRECTIVE_ACTION_INCOMPLETE`,
  `RECURRED`, `OTHER`).
- **FR-CAPA-061 — Reopening after RESOLVED (pre-CLOSED).** Within the `RESOLVED` grace period
  (FR-CAPA-052), any user with `issue:triage`-or-higher scope may manually reopen (e.g. a
  stakeholder raises an objection before auto-close). This follows the same
  `RESOLVED -> REOPENED -> INVESTIGATING` branch.
- **FR-CAPA-062 — Reopening after CLOSED.** A `CLOSED` Issue is not reopened via the standard
  transition. Instead, a `RECURRENCE` link is created: the system creates a **new** Issue with
  `source = RECURRENCE_OF`, `recurrence_of_issue_id` FK to the closed Issue, copying category,
  location, and a reference to the prior root cause analysis for investigator context. This
  preserves the closed record's immutability (P7) while still connecting the two events for
  repeat-failure pattern detection (LLD pseudocode). Rationale: reopening a formally closed,
  potentially audited record would break the tamper-evident audit trail; a linked new Issue achieves
  the same operational goal (resume corrective work) without mutating history.
- **FR-CAPA-063.** Reopen count and recurrence-chain depth are tracked
  (`Issue.reopen_count`, and via `recurrence_of_issue_id` traversal) and exposed to the Compliance
  Monitor Agent and Analytics as first-class fields, since repeat-failure rate is a top-level KPI
  (source-spec §14.2).

## 10. SLA per Severity and Escalation Ladder

- **FR-CAPA-070 — SLA policy.** Each tenant configures an `escalation_policy` per `severity` x
  `category` combination (falling back to a per-severity tenant default), defining: `time_to_triage`,
  `time_to_assign`, `time_to_investigate`, `time_to_corrective_action`, `time_to_verification`, each
  as a duration, plus an ordered escalation ladder of `{ after_duration, notify_role_or_user,
  action }` steps (`action` ∈ `NOTIFY`, `REASSIGN_TO_MANAGER`, `ESCALATE_SEVERITY`,
  `CREATE_EXECUTIVE_ALERT`). Default targets (recommended, tenant-overridable):
  `CRITICAL` triage ≤ 15 min, assign ≤ 30 min, corrective action started ≤ 4 hrs, verification
  scheduled ≤ 7 days; `HIGH` triage ≤ 1 hr, assign ≤ 4 hrs, corrective action ≤ 24 hrs, verification
  ≤ 14 days; `MEDIUM`/`LOW` triage ≤ 1 business day, remaining steps ≤ configurable multi-day windows.
- **FR-CAPA-071 — SLA timer implementation.** Each Issue's active SLA clock is modeled as a Temporal
  workflow (one workflow execution per Issue, per `conventions.md` — Temporal for SLA/escalation),
  started on `issue.created` (or on approval, for agent-created Critical issues) and reset/re-armed
  on each qualifying state transition. This is the same Temporal-based escalation infrastructure
  pattern used by Work Orchestration's WorkItem SLA timers, but keyed on `Issue` state rather than
  `WorkItem` state (LLD §Sequence Diagrams).
- **FR-CAPA-072 — Escalation integration with Work Orchestration.** When an escalation ladder step
  fires `ESCALATE_SEVERITY` or `CREATE_EXECUTIVE_ALERT`, this module emits `issue.escalated` and
  Work Orchestration's Notification Hub delivers the configured notification; when the action is
  itself a corrective-action WorkItem going overdue, Work Orchestration's own WorkItem escalation
  ladder fires independently and reports back via `task.overdue`, which this module surfaces as an
  `issue_events` entry without duplicating the escalation logic.

## 11. UX States

- **FR-CAPA-080 — Issue list.** Filterable by `severity`, `status` (lifecycle state), `owner`,
  `category`, `location`/location-subtree, `overdue`/`escalated` flag, and `created source`.
  Default sort: severity desc, then SLA-remaining asc (most at-risk first). Supports saved filter
  views per user/role.
- **FR-CAPA-081 — Kanban-by-stage view.** Columns = lifecycle states (`OPEN` through `CLOSED`,
  `REOPENED` shown as a distinct lane feeding back into `INVESTIGATING`). Cards show severity badge,
  SLA countdown/overdue badge, owner avatar, category icon. Drag-to-transition is only permitted
  where the target transition is valid and the user holds the required scope/role for that
  transition; an invalid drag is rejected client-side with the same validation the API enforces
  server-side (no client-only business rules).
- **FR-CAPA-082 — Root-cause-method picker.** Presented on entering root cause analysis; shows
  method name, one-line description, and (for regulated categories) which methods are
  admin-restricted with a lock icon and the reason. Selecting a method renders its structured form
  (§7). Switching methods after data entry warns with an explicit confirmation ("this starts a new
  root cause version; your prior analysis is preserved but no longer active") per FR-CAPA-046.
- **FR-CAPA-083 — Evidence attachment at each CAPA step.** Every CAPA step screen (immediate
  correction, containment, corrective/preventive action, effectiveness verification) exposes the
  standard Evidence capture control (photo/video/document/signature per Evidence module) scoped to
  that specific step (`capa_step_id` on the Evidence linkage), not just the Issue as a whole, so the
  evidence trail is attributable to the exact step it substantiates.
- **FR-CAPA-084 — Overdue/escalated visual treatment.** An Issue whose current SLA stage has passed
  its target duration is shown with an amber "at risk" treatment; once an escalation ladder step has
  actually fired, it shows a red "escalated" treatment with the escalation level (e.g.
  "Escalated to Regional Manager, step 2 of 3") visible on hover/detail. This treatment is
  consistent with the WorkItem `OVERDUE`/`ESCALATED` visual language used elsewhere in the platform.
- **FR-CAPA-085 — Empty/loading/offline/permission-denied states.** Empty issue list shows a
  zero-state distinguishing "no issues" from "no issues match filter." Manager Workspace issue
  widgets support cached-stale display when offline (mobile), with a visible "last synced" timestamp;
  CAPA step submission from a mobile device queues via the standard sync protocol (§19 conventions)
  when offline, with idempotent submission keyed on `client_uuid`. A user without `issue:read` scope
  at a location sees a scope-denied state, not a 404, to avoid leaking existence of issues at
  inaccessible locations to users who lack even list-level visibility (list results are pre-filtered
  server-side; a direct deep link to an out-of-scope Issue ID returns `403 SCOPE_DENIED`).

## 12. Validation and Edge Cases

- **9.1 — Issue with no owner after org reorg.** See FR-CAPA-022. Orphaned Issues never silently
  lose SLA tracking; the sweeper both flags ownership and escalates to the next org level
  immediately (does not wait for the next natural escalation ladder step) since an ownerless Issue
  is itself an SLA risk.
- **9.2 — Duplicate issue detection for same root event.** See FR-CAPA-041. Duplicate suggestions are
  never auto-merged; the "same root event" case most likely to occur (multiple frontline users
  reporting the same visible defect) is handled via explicit `duplicate_of_issue_id` linkage, and
  rule-triggered duplicates from a replayed/retried rule evaluation are prevented at the source via
  the `(tenant_id, rule_id, submission_id)` idempotency key (FR-CAPA-002), not via similarity
  matching (similarity matching is a human-facing aid, not the primary dedup mechanism for
  machine-originated issues).
- **9.3 — Reopening after CLOSED vs. after RESOLVED.** See FR-CAPA-061/062. The distinction is
  deliberate and load-bearing: `RESOLVED` is a provisional, still-mutable state (grace period);
  `CLOSED` is the immutable terminal state. Any UI or API path that would mutate a `CLOSED` Issue's
  lifecycle state directly must be rejected with `409 CONFLICT` / error code `ISSUE_CLOSED_IMMUTABLE`
  and redirected to the recurrence-linking flow.
- **9.4 — Concurrent CAPA edits by two managers.** Optimistic concurrency (per conventions.md API
  conventions) applies at both the `Issue` row (`version`/`ETag`) and independently at each
  `CapaStep` row, so two managers editing different steps of the same Issue concurrently do not
  false-conflict; two managers editing the *same* step concurrently receive a `409 Conflict` with the
  current server version on the losing write, consistent with the platform-wide rule against silent
  last-write-wins for materially important values (§19.3 convention). The UI surfaces a merge view
  showing both versions' field-level diffs rather than forcing a blind overwrite-or-discard choice.
- **9.5 — Root cause method changed mid-investigation.** See FR-CAPA-046. Any corrective action
  already created against the superseded analysis version remains valid and linked; the UI flags it
  ("based on a superseded root cause analysis") but does not auto-cancel it, since the corrective
  action may still be appropriate even if the formal root-cause narrative changes.
- **9.6 — Severity downgrade after escalation has already fired.** A re-triage that lowers severity
  does not retroactively un-fire an escalation notification already sent, but does recompute the
  remaining SLA clock against the new severity's targets going forward and records the downgrade
  reason for audit.
- **9.7 — Corrective-action WorkItem rejected/failed.** If the linked Work Orchestration WorkItem is
  rejected (`PENDING_REVIEW -> REJECTED -> IN_PROGRESS`) or itself escalates, the parent Issue does
  not auto-advance past `CORRECTIVE_ACTION`; it surfaces the WorkItem's state inline and blocks the
  `CORRECTIVE_ACTION -> VERIFICATION` transition until the WorkItem reaches a terminal
  success state (`VERIFIED`) or the owner explicitly marks the corrective action complete via an
  alternate (non-WorkItem) path with justification.
- **9.8 — Issue created for a location with no assignable manager.** If triage cannot resolve a
  default assignee (no active `UserScope` at the location or its parent chain within a configurable
  number of levels), the Issue remains in `TRIAGED` with `owner_status = UNASSIGNABLE` and raises an
  immediate admin alert rather than blocking creation.

## 13. Non-Functional Requirements

- **NFR-CAPA-001 — Time to triage.** For auto-triage-eligible categories, `OPEN -> TRIAGED` completes
  within 5 seconds of issue creation (p95), since it is a synchronous rule evaluation, not a
  human-paced step. For manual triage, the product target (measured, not enforced) is p50 ≤ 15
  minutes for `CRITICAL`/`HIGH` severity issues, tracked as the "Time to Intervention" KPI
  contributor (source-spec §14.2).
- **NFR-CAPA-002 — SLA timer accuracy.** Escalation ladder steps fire within 60 seconds of their
  configured `after_duration` boundary (p99), bounded by Temporal's timer granularity and workflow
  worker capacity; timer drift is monitored and alerted (Observability, §22 conventions) since a
  late-firing SLA escalation is itself a governance/compliance defect for regulated tenants.
- **NFR-CAPA-003 — API performance.** Issue and CAPA-step mutation endpoints meet the platform p95 <
  300 ms target under normal load (source-spec §23); the root-cause-analysis submit endpoint,
  which may validate a larger JSONB payload against a per-tenant custom JSON Schema, is allowed a
  relaxed p95 < 600 ms.
- **NFR-CAPA-004 — Auditability.** Every state transition, reassignment, severity change, and CAPA
  step edit produces an immutable `issue_events` row sufficient to reconstruct the full Issue
  timeline without relying on mutable current-state tables, supporting regulated-industry audit
  requirements (food safety, safety incidents).
- **NFR-CAPA-005 — Scale.** The design supports the platform's stated scale target (100k locations,
  millions of work items/day, source-spec §23); Issues are a materially smaller volume than
  WorkItems but must support high-volume bursts during systemic events (e.g. a single equipment
  recall generating thousands of Issues within minutes) without SLA-timer fan-out degrading other
  tenants (see HLD §Scalability).

## 14. Dependencies

| Module | Dependency |
|---|---|
| Work Orchestration (EPIC-03) | Creates/tracks `WorkItem`s of type `ISSUE_CORRECTIVE_ACTION`/inspection type for corrective action and effectiveness re-inspection; supplies `task.completed`/`task.overdue` events this module consumes; supplies the underlying Temporal escalation-ladder pattern this module's SLA timers reuse. |
| Evidence (EPIC-06) | Every CAPA step's evidence capture references `Evidence` records by ID; this module never stores media directly. |
| Learning (EPIC-09) | Preventive actions may link a `LearningAssignment`/`Course`; repeated-failure patterns detected here can trigger the `REPEATED FAILURE -> ... -> RE-INSPECTION` loop (source-spec §11.3), coordinated via events, not direct writes into Learning's schema. |
| Agentic Ops / Agent Runtime (EPIC-11) | Resolution Agent and Compliance Monitor Agent call this module's typed tools (`create_issue`, `get_recent_failures`, `schedule_reinspection`); this module is the tool provider and the authoritative state owner for any resulting Issue. |
| Communications (EPIC-08) | All Issue notifications (assignment, escalation, verification due) are delivered via the Notification Hub; this module only emits events/calls the send interface. |
| Form Platform (EPIC-04) | Source of rule-triggered issue creation (Appendix B rule shape); this module consumes the resulting trigger, not the rule engine itself. |
| Identity/Org (EPIC-01/02) | `UserScope`/`OrganizationNode` resolve default assignees, reassignment eligibility, and the orphaned-owner sweeper's escalation target. |
| Analytics & Control Tower (EPIC-12/15) | Consumes `issue.*` events and CDC feed for Repeat Failure Rate, Exception Resolution Time, Time to Intervention KPIs. |

## 15. Resolved Decisions

- **Custom root-cause method authoring (source-spec §10.4)**: The five system root-cause methods are
  the full initial set. A tenant's "custom enterprise method" is modeled as a tenant-registered JSON
  Schema owned and validated inside this module, **not** authored via the Form Platform's general
  form builder. This is the firm decision for this version: the Form Platform's builder brings
  conditional-logic and branching capability this module's RCA capture does not need, and routing
  custom-method authoring through it would add a cross-module coupling (Issues & CAPA depending on
  Form Platform's builder/runtime) for no functional gain. Revisit only if a tenant need for the
  builder's conditional-logic capability materializes.
- **`RESOLVED` grace period before auto-close**: Fixed default of **3 days**, tenant-configurable.
  3 days is chosen as the firm default because it is long enough to allow a verifier to reopen a
  wrongly-marked-resolved Issue during the same or next work cycle, without leaving genuinely
  resolved Issues open indefinitely and inflating open-Issue counts on manager dashboards.
- **`CRITICAL` agent-created issues without human approval within the bounded window**: Resolved as
  follows — a `CRITICAL` agent-created issue awaiting approval that has not received a human decision
  within **30 minutes** triggers escalation of **the approval request itself** (not the Issue) via
  the standard notification escalation ladder (FR-WORK-060-equivalent pattern), and continues to
  hold indefinitely thereafter until a human decides. The agent action is never auto-approved on
  timeout, consistent with Product Principle P5 (Policy before AI) and with the EU AI Act Article 14
  human-oversight rule (`engineering-decisions.md` §12) for any agent output materially affecting a
  named individual — this Issue-approval path is treated the same way even where the specific Issue
  is not employee-performance-related, for one consistent, auditable approval-timeout behavior
  platform-wide rather than a case-by-case exception.
- **`duplicate_of_issue_id` cross-location linkage**: User-facing duplicate suggestions are
  restricted to **same-location** matches only. Cross-location correlation (e.g., the same defective
  equipment model failing at multiple sites) is surfaced exclusively through the separate
  repeat-failure pattern signal (low-level-design.md pseudocode), never through the `duplicate_of`
  relationship. This is firm because conflating "duplicate" (the same event, one location) with
  "pattern" (related events across locations) would weaken the per-site audit distinctness of each
  location's evidence trail, which regulated tenants rely on for site-specific compliance reporting.
