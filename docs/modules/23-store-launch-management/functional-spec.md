# New-Location Launch Management — Functional Specification

Module corresponds to **EPIC-03 (Work Core) extension**, addendum Module 23 (Tier 3). Source:
`_context/source-spec.md` §7 (Work Orchestration), §17.1 (WorkItem, OrganizationNode entities),
`_context/new-modules-addendum.md` Module Map row 23. This module has no dedicated Epic of its own
in Appendix E; its stories are tracked under **EPIC-03.S-LAUNCH-*** (Work Core extension) plus
cross-references into **EPIC-01** (Org Graph) and **EPIC-13** (Integration Hub).

## 1. Purpose and Scope

### 1.1 Purpose
New-Location Launch Management gives Operations Admins and Regional/District Managers a purpose-
built way to plan, template, instantiate, and track the multi-week, multi-phase project of opening
(or relaunching/remodeling) a physical location — construction and permitting, staffing, inventory
and merchandising setup, marketing, and grand opening — as a single governed project with a
countdown, phase rollups, and portfolio-level visibility across many simultaneous openings.

This module is explicitly **a configuration and orchestration layer on top of Work Orchestration**,
not a new execution engine. A "launch" is a structured **Campaign** (source-spec §7.2 "Seasonal/
campaign schedules," FR-WORK-007 Campaign Task, FR-WORK-027) whose member WorkItems are governed by
Work Orchestration's existing state machine, scheduling, assignment, and dependency-sequencing
machinery end to end. This module's job is to know **what a launch looks like** (the reusable
template), **turn a template into a real project** bound to a location, and **roll up progress** in
launch-specific language (phases, milestones, days-to-launch, at-risk) that a generic Work
Orchestration campaign view would not otherwise express.

### 1.2 In Scope
- **Launch Template authoring** — a named, ordered, versioned checklist of phases → milestones →
  tasks spanning construction/permitting, staffing, inventory, marketing, and grand-opening, with
  cross-phase dependency declarations (e.g., "staffing cannot start until the permitting milestone
  is complete").
- **Launch Project instantiation** — binding a template to a specific target location: either a
  **not-yet-existing OrganizationNode** created as part of instantiation, or an **existing
  OrganizationNode** being converted/relaunched (remodel, rebrand, re-opening after closure).
- **Progress tracking and rollup** in launch-native terms: phase completion percentage, days-to-
  launch countdown, milestone status, and **at-risk flagging** when a milestone is behind the
  template's expected schedule.
- **Multi-launch portfolio view** — a Regional/District Manager's Gantt-style rollup across dozens
  of concurrent store openings in their territory.
- **Template versioning** consistent with the platform's general immutability principle
  (source-spec P7, Template/TemplateVersion state machine): an in-flight launch keeps the template
  version it was instantiated from.
- **Launch-specific reporting/rollup entities** (`LaunchProject`, `LaunchTemplate`,
  `LaunchMilestone`) and their own lifecycle metadata layered over Work Orchestration WorkItems.

### 1.3 Out of Scope (owned by other modules, referenced here)
- **Task execution, assignment, and dependency-based sequencing of the actual checklist items** —
  owned entirely by **Work Orchestration (EPIC-03)**. Every leaf task inside a launch is a
  `WorkItem` of type `LAUNCH_TASK` (an extension of the `CAMPAIGN_TASK` family per the addendum's
  new-WorkItem-types list), created, scheduled, assigned, submitted, approved, and verified through
  Work Orchestration's existing state machine and APIs. This module never re-implements assignment,
  the `DRAFT → ... → VERIFIED` state machine, offline sync of task execution, or SLA/escalation
  timers — it only **declares** the dependency graph and **calls** Work Orchestration to
  materialize it.
- **OrganizationNode creation mechanics, effective-dating, and RBAC/scope propagation** — owned by
  **Identity/Org (EPIC-01/02)**. This module calls Identity/Org's create-node API and reacts to its
  result; it does not own hierarchy validity, node typing rules, or scope inheritance.
- **Form schema/template design for individual launch tasks** (e.g., a "final health inspection"
  form) — owned by **Forms & Workflow Builder**; a `LaunchTemplateTask` may reference a
  `TemplateVersion` the same way any other WorkItem does.
- **New-store master-data feeds** (POS setup, ERP store record creation, planogram feed, supplier
  activation) — owned by **Integrations & Extensibility (EPIC-13)**. This module can trigger an
  outbound integration event at a configured milestone but does not own connector configuration,
  retries, or field mapping.
- **Launch-pipeline analytics/warehouse rollups beyond the module's own portfolio dashboard** —
  owned by **Analytics & Control Tower (EPIC-12/15)**; this module emits events and exposes a
  dashboard read API that Analytics can also consume via CDC, but does not own the lakehouse layer.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Operations Admin | Authors and versions Launch Templates; configures cross-phase dependencies and the OrganizationNode-creation trigger point; publishes templates for use |
| Regional/District Manager | Instantiates launch projects for locations in their territory; owns the multi-launch portfolio view across concurrent openings; resolves at-risk milestones |
| Store Manager-designate | Executes on-site launch tasks (the underlying `LAUNCH_TASK` WorkItems) from the mobile "Today" screen like any other assigned work; sees a launch-specific countdown and phase view layered on top |
| Executive | Tracks the enterprise launch pipeline — how many stores are opening this quarter, which are behind, aggregate days-to-launch trend — via the portfolio dashboard and Analytics |
| Enterprise Admin | Governs which Operations Admins may publish Launch Templates enterprise-wide vs. brand/region-scoped |

## 3. Functional Requirements

### 3.1 Launch Template Authoring

- **FR-LAUNCH-001.** An Operations Admin creates a `LaunchTemplate` with a `name`, `description`,
  and an ordered list of **phases** drawn from a configurable enum
  (`CONSTRUCTION`, `PERMITTING`, `STAFFING`, `INVENTORY_MERCHANDISING`, `MARKETING`,
  `GRAND_OPENING`, plus tenant-defined custom phases). Phase order is a default display order only;
  actual sequencing is governed by explicit dependencies (FR-LAUNCH-004), not phase order alone.
- **FR-LAUNCH-002.** Each phase contains one or more `LaunchTemplateMilestone` rows — a named
  checkpoint (e.g., "Lease Signed," "Permits Approved," "Store Fully Staffed," "Opening Day") with
  an `expected_offset_days` relative to a template-defined **anchor date type**
  (`TARGET_OPEN_DATE` counting backward, or `PROJECT_START` counting forward — see FR-LAUNCH-011 for
  how the anchor is resolved at instantiation).
- **FR-LAUNCH-003.** Each milestone contains one or more `LaunchTemplateTask` rows — the leaf
  checklist items (e.g., "Submit building permit application," "Post store manager job req," "Run
  planogram reset"). A task optionally references a `template_version_id` (Forms module) when
  execution requires a structured form/evidence submission, and optionally an
  `integration_trigger_code` (Integrations module) when completion should fire an outbound feed
  (e.g., POS provisioning request).
- **FR-LAUNCH-004.** Cross-phase and same-phase task/milestone **dependencies** are declared at
  template authoring time as `(predecessor_ref, successor_ref, dependency_mode)` edges, where refs
  may point at a milestone (all tasks under it) or an individual task, and
  `dependency_mode ∈ {ALL, ANY, THRESHOLD(n)}` mirrors Work Orchestration's
  `work_item_dependencies` semantics (FR-WORK-060/063) so the graph translates 1:1 at instantiation.
  Example: the `STAFFING` phase's first milestone declares a dependency on the `PERMITTING` phase's
  "Permits Approved" milestone reaching `VERIFIED`.
- **FR-LAUNCH-005.** The template author designates exactly one milestone as the
  **OrganizationNode-creation trigger point** (default: the first milestone of the template, e.g.
  "Lease Signed") when the template is intended for brand-new locations. This trigger point is
  configurable per template, defaulting to "at a configured milestone" rather than at instantiation —
  see §8 Resolved Decisions for the rationale.
- **FR-LAUNCH-006.** Dependency graphs are validated acyclic at template save time, reusing the same
  validation approach as Work Orchestration FR-WORK-061 (the template authoring UI calls a shared
  cycle-detection check before allowing publish).
- **FR-LAUNCH-007.** A `LaunchTemplate` follows the platform's standard
  `DRAFT → REVIEW → APPROVED → PUBLISHED → SUPERSEDED → RETIRED` lifecycle (conventions.md
  TemplateVersion state machine), reusing the same immutable-after-publish principle: publishing a
  template produces an immutable `LaunchTemplateVersion` snapshot of every phase/milestone/task/
  dependency row.
- **FR-LAUNCH-008.** An admin may **duplicate** a published template as the starting point for a new
  DRAFT (e.g., "QSR Standard Launch v2 copied from v1") rather than editing a published version in
  place, consistent with P7 (Historical truth) and P10 (Design for change).
- **FR-LAUNCH-009.** Template scope: a template is authored at a given `OrganizationNode` scope
  (e.g., a specific brand or business unit) and is available for instantiation only at or below that
  scope, reusing the standard `UserScope`/org-scope enforcement.

### 3.2 Launch Project Instantiation

- **FR-LAUNCH-010.** A Regional/District Manager or Operations Admin instantiates a `LaunchProject`
  by selecting a `PUBLISHED LaunchTemplateVersion` and one of two target-location modes:
  - **New location mode** — no existing `OrganizationNode`; the project carries provisional location
    attributes (name, address, planned brand/format) until the configured milestone triggers actual
    node creation (FR-LAUNCH-005, FR-LAUNCH-013).
  - **Existing location mode** — an existing `OrganizationNode` is bound immediately (relaunch,
    remodel, rebrand, reopening-after-closure scenario); no node-creation step occurs, and any
    milestone/task designated as "OrganizationNode-creation trigger" is skipped as already-satisfied.
- **FR-LAUNCH-011.** At instantiation, the caller supplies a `target_open_date` (or a
  `project_start_date` for templates anchored `PROJECT_START`). The template-expansion process
  (§ LLD Pseudocode) resolves every milestone's `expected_offset_days` against this anchor to
  produce a concrete `expected_date` per `LaunchProjectMilestoneProgress` row, and generates the
  underlying Work Orchestration WorkItem graph with `due_at` values derived the same way.
- **FR-LAUNCH-012.** Instantiation performs a **bulk WorkItem creation** call into Work Orchestration
  (§ LLD API) that creates one `WorkItem` of type `LAUNCH_TASK` per `LaunchTemplateTask`, wires
  `work_item_dependencies` rows per FR-LAUNCH-004's translated graph, and tags every created
  WorkItem with the `LaunchProject.id` as its `campaign_id` (reusing FR-WORK-007's campaign grouping
  key) so Work Orchestration's own campaign rollup queries also work against a launch out of the box.
- **FR-LAUNCH-013.** When the target is **new-location mode**, the WorkItem/task corresponding to
  the configured OrganizationNode-creation trigger milestone carries a
  `post_verification_action = CREATE_ORG_NODE` marker. On that WorkItem reaching `VERIFIED`, this
  module calls Identity/Org's `POST /api/v1/orgs/nodes` (§ LLD sequence diagram) using the
  provisional attributes captured at instantiation, then patches the `LaunchProject` with the
  resulting real `location_id` and propagates it onto every remaining not-yet-materialized WorkItem
  in the campaign (already-materialized WorkItems keep their placeholder `location_id` reference for
  historical accuracy per P7, resolved via the project's `provisional_location_ref` join at query
  time — see LLD §2).
- **FR-LAUNCH-014.** Instantiating a launch project is **idempotent** on a client-supplied
  `Idempotency-Key`, consistent with conventions.md; a duplicate instantiation request returns the
  original `LaunchProject` rather than creating a second one.
- **FR-LAUNCH-015.** A `LaunchProject` may be **cancelled** at any point before `COMPLETE`. Cancelling
  bulk-cancels every non-terminal underlying WorkItem in the campaign (Work Orchestration's own
  cancel semantics) and transitions the `LaunchProject` to `CANCELLED` with a mandatory reason. A
  cancelled project's OrganizationNode (if already created) is **not** automatically deleted or
  deactivated — that is a separate, explicit Identity/Org action an admin takes deliberately.

### 3.3 Progress Tracking, Rollup, and At-Risk Detection

- **FR-LAUNCH-020.** `LaunchProject.status` derives from, but is a distinct state machine from, the
  aggregate status of its underlying WorkItems: `PLANNED → IN_PROGRESS → AT_RISK → COMPLETE`, with a
  `CANCELLED` branch reachable from any non-terminal state (see architecture-diagram.md §4 for the
  full `stateDiagram-v2`).
- **FR-LAUNCH-021.** **Phase completion percentage** = (count of `VERIFIED` WorkItems in phase) /
  (total WorkItems in phase), recomputed on every relevant `task.completed`/`task.verified` event
  (event-driven) and reconciled by a periodic background sweep (§ LLD Background Jobs) to guard
  against missed events.
- **FR-LAUNCH-022.** **Days-to-launch countdown** = `target_open_date - current_date`, always shown
  alongside the current phase and next incomplete milestone on both the single-launch detail view
  and the portfolio view.
- **FR-LAUNCH-023.** **At-risk flagging.** A milestone is flagged `AT_RISK` when either: (a) its
  `expected_date` has passed and it has not reached completion, or (b) a configurable
  look-ahead window (default 5 days, tenant-configurable) shows the milestone's completion velocity
  (tasks completed so far vs. tasks remaining vs. days remaining) will miss `expected_date` at the
  current pace (see LLD §6 pseudocode `detectAtRisk`). A `LaunchProject` rolls up to `AT_RISK` status
  when any of its non-complete milestones is `AT_RISK`.
- **FR-LAUNCH-024.** At-risk state surfaces as a **badge** on both the single-launch view and every
  row of the portfolio view (§4 UX States), and emits a `launch.milestone_at_risk` event so
  Communications/Notification Hub can alert the Regional Manager and Operations Admin without this
  module owning delivery mechanics.
- **FR-LAUNCH-025.** `LaunchProject` reaches `COMPLETE` when every WorkItem in its campaign is
  `VERIFIED` (or explicitly marked `SKIPPED` by an admin override with audit-logged reason for
  genuinely inapplicable tasks, e.g. a remodel skipping a "hire store manager" task). Reaching
  `COMPLETE` triggers the grand-opening closeout UX (§4) and a `launch.completed` event.

### 3.4 Multi-Launch Portfolio View

- **FR-LAUNCH-030.** A Regional/District Manager's portfolio view lists every `LaunchProject` in
  their org scope (standard `UserScope` filtering) as a Gantt-style timeline row: template name,
  target location (or provisional name), target open date, days-to-launch, current phase, phase
  completion %, and at-risk badge.
- **FR-LAUNCH-031.** The portfolio view supports filtering/grouping by status, template, region, and
  target open date range, and must render responsively for **20+ simultaneous launches** without a
  perceptible load-time regression (see NFRs).
- **FR-LAUNCH-032.** Drilling into a single launch from the portfolio view opens the single-launch
  detail view (phase/milestone breakdown, task list delegated to Work Orchestration's own WorkItem
  list UI filtered by `campaign_id`).
- **FR-LAUNCH-033.** An Executive-facing rollup (via Analytics, not owned here) aggregates portfolio
  data enterprise-wide: launches this quarter, average days-to-launch variance, at-risk rate by
  template/region.

### 3.5 Template Versioning

- **FR-LAUNCH-040.** An in-flight `LaunchProject` stores an immutable reference
  (`launch_template_version_id`) to the exact template version it was instantiated from. Editing or
  publishing a new version of the source `LaunchTemplate` **never** alters an in-flight project's
  phases, milestones, tasks, or dependency graph — consistent with P7 (Historical truth) and the
  platform-wide TemplateVersion immutability rule.
- **FR-LAUNCH-041.** A published `LaunchTemplate` update (e.g., adding a new phase) produces a new
  `LaunchTemplateVersion`; the prior version is marked `SUPERSEDED` but remains readable for any
  project still referencing it. New instantiations default to the latest `PUBLISHED` version.
- **FR-LAUNCH-042.** The template list UI shows, for each version, how many active `LaunchProject`
  rows still reference it — an "impact preview" analogous to the Forms module's Impact Preview
  (source-spec §8.4) — so an admin retiring a version understands the blast radius before doing so.

## 4. UX States

| State | Behavior |
|---|---|
| Launch portfolio (Gantt-style) view | Default landing for Regional/District Manager; rows sorted by days-to-launch ascending by default; at-risk rows visually distinguished (badge + row highlight) |
| At-risk-milestone badge | Amber badge with tooltip explaining the specific trigger (behind expected date / velocity projection) and a "View blocking tasks" action that deep-links into the filtered Work Orchestration task list |
| Phase-locked-pending-dependency | A phase/milestone not yet unlocked shows a distinct locked UX (padlock affordance) naming the specific predecessor milestone still outstanding — mirrors Work Orchestration's `BLOCKED` WorkItem UX (FR-WORK-062) but expressed at the launch-milestone level for a manager audience |
| Launch-complete celebratory closeout | On `COMPLETE`, the single-launch view replaces the phase tracker with a celebratory summary (total days elapsed vs. planned, on-time milestones vs. missed, link to the new location's live Work Orchestration/Control Tower views) — a deliberate moment-of-completion UX distinct from routine task completion |
| Empty state (no launches) | Portfolio view for a manager/tenant with zero active launches shows a call-to-action to browse published templates and start a new launch |
| Loading | Skeleton rows for the portfolio Gantt view; skeleton phase tracker for single-launch detail |
| Offline | Launch authoring/instantiation/portfolio views are admin/manager web-console surfaces and are **not** required to function offline (they are not part of the frontline mobile offline surface); underlying `LAUNCH_TASK` WorkItems executed by frontline/store-manager-designate users on mobile fully inherit Work Orchestration's offline behavior |
| Permission-denied | An Operations Admin without template-authoring scope at the target org node sees a read-only template view with an explicit "request access" affordance rather than a raw 403 |
| Cancelled | A cancelled `LaunchProject` is shown greyed out in the portfolio with the cancellation reason and date, retained for historical reporting, not deleted |

## 5. Validation and Edge Cases

- **FR-LAUNCH-050.** Editing a `LaunchTemplate` while one or more `LaunchProject` rows are in-flight
  against a published version **must not** retroactively alter those projects' instantiated tasks or
  dependency graph (see FR-LAUNCH-040). Enforced structurally: edits always target a new `DRAFT`
  version; there is no in-place mutation path for a `PUBLISHED` version's phase/milestone/task rows.
- **FR-LAUNCH-051.** **Relaunch/remodel vs. brand-new** is an explicit instantiation-time choice
  (FR-LAUNCH-010), not inferred. Instantiating against an existing `OrganizationNode` that already
  has an active (non-`COMPLETE`/`CANCELLED`) `LaunchProject` is rejected with
  `409 LAUNCH_ALREADY_ACTIVE_FOR_LOCATION` unless the caller explicitly passes
  `allow_concurrent=true` (rare, tenant-configurable, e.g. a remodel running alongside an unrelated
  compliance-only mini-launch).
- **FR-LAUNCH-052.** A `LaunchProject` **abandoned/cancelled mid-way** (FR-LAUNCH-015) leaves its
  partially-created OrganizationNode (if any) intact and its already-`VERIFIED` WorkItems intact for
  audit purposes; only non-terminal WorkItems are bulk-cancelled. Re-attempting a launch for the same
  location after cancellation requires a fresh instantiation (a new `LaunchProject`), which is
  allowed once the cancelled one is in a terminal state.
- **FR-LAUNCH-053.** **Milestone dependencies spanning two different launches are out of scope.**
  Only same-launch, cross-phase dependencies are supported (FR-LAUNCH-004). A template cannot
  declare "Store B's staffing milestone depends on Store A's grand opening" — that class of
  cross-project sequencing, if ever needed, belongs to a future Work Orchestration cross-campaign
  dependency capability, not this module.
- **FR-LAUNCH-054.** If the OrganizationNode-creation trigger WorkItem is cancelled or fails
  verification, no node is created and all downstream WorkItems that would reference the new node
  remain blocked (`dependency_mode` unsatisfied) rather than silently proceeding without a location.
- **FR-LAUNCH-055.** Changing a `LaunchProject`'s `target_open_date` after instantiation
  (`PATCH /api/v1/launch-projects/{id}`) recomputes every not-yet-`VERIFIED` milestone's
  `expected_date` and the corresponding WorkItems' `due_at` (a bulk update call into Work
  Orchestration), but never alters already-`VERIFIED` milestones' historical expected/actual dates.
- **FR-LAUNCH-056.** A template with a dependency cycle, or a task referencing a nonexistent
  `template_version_id`/`integration_trigger_code`, is rejected at `PUBLISH` time with a structured
  validation error listing every offending edge/reference (batched, not fail-fast on the first one).
- **FR-LAUNCH-057.** Instantiating a template whose target open date is in the past, or fewer days
  away than the template's longest dependency-chain lead time, is allowed but immediately flags the
  project `AT_RISK` at creation with a warning surfaced to the instantiating manager before
  confirmation (non-blocking, since real-world compressed timelines happen).

## 6. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Portfolio dashboard read latency | p95 < 300 ms for up to 100 concurrent launch rows per tenant (conventions.md general API NFR), backed by the cache in LLD §7 |
| Instantiation bulk-creation | A 200-task launch template expands and bulk-creates its full WorkItem + dependency graph in Work Orchestration in a single logical operation completing in < 5 s p95 |
| Consistency | `LaunchProject` progress rollup is eventually consistent with underlying WorkItem state (event-driven, reconciled by periodic sweep — see LLD §8); never blocks WorkItem completion on rollup availability |
| Multi-tenancy | Every table carries `tenant_id`; template and project scoping additionally enforced via `OrganizationNode`/`UserScope` per conventions.md |
| Auditability | Every template publish, instantiation, cancellation, target-date change, and node-creation trigger is `AuditLog`-recorded per source-spec §17.1 |
| Availability | Inherits the platform's 99.9%+ core API target; this module has no independent availability tier since it is a thin layer over Work Orchestration/Identity-Org, both of which must already meet that target |
| Scale | Designed for tenants running dozens of concurrent multi-week launches, each with 50–200 tasks (hundreds to low thousands of `LAUNCH_TASK` WorkItems live at once per large tenant) |

## 7. Dependencies

- **Work Orchestration (EPIC-03)** — owns the `WorkItem`/`work_item_dependencies` execution
  substrate for every `LAUNCH_TASK`; this module calls its bulk-creation, dependency-wiring, and
  cancel APIs and consumes its `task.completed`/`task.verified`/`task.overdue` events for rollup.
- **Identity/Org (EPIC-01/02)** — owns `OrganizationNode` creation, typing, and effective-dated
  hierarchy; this module calls `POST /api/v1/orgs/nodes` at the configured trigger milestone and
  consumes the resulting `location_id`.
- **Integrations & Extensibility (EPIC-13)** — receives outbound events/triggers for new-store
  master-data feeds (POS/ERP setup) fired from configured `LaunchTemplateTask.integration_trigger_code`
  completions; this module does not own connector configuration or delivery guarantees.
- **Forms & Workflow Builder** — supplies `TemplateVersion` references for any launch task requiring
  a structured form/evidence submission (e.g., a final walkthrough checklist).
- **Analytics & Control Tower (EPIC-12/15)** — consumes this module's events and dashboard data via
  CDC for enterprise launch-pipeline reporting (launches per quarter, average variance to plan,
  at-risk rate).
- **Communications/Notification Hub** — delivers at-risk and completion notifications triggered by
  this module's events; delivery mechanics are out of scope here.

## 8. Resolved Decisions

- **OrganizationNode creation timing: configurable per template, defaulting to a configured milestone
  (FR-LAUNCH-005/013), never immediately at instantiation.** Enterprises frequently begin
  planning/permitting work before a lease is legally signed or a site is committed, and creating a
  live `OrganizationNode` (with RBAC scope, visibility in every location-scoped report, and potential
  accidental assignment of real frontline work) too early creates operational noise and premature
  reporting inclusion. Templates for pure remodels/relaunches (existing-location mode) never exercise
  this trigger since the node already exists.
- **"Template mid-flight upgrade" (moving an in-flight `LaunchProject` onto a newer template version)
  is not supported in this version.** It would contradict FR-LAUNCH-040's immutability guarantee.
  Deferred as a possible Phase 2 capability if customers request it, to be modeled then as an
  explicit, audited "re-baseline" operation rather than silent version drift — no partial or implicit
  version-migration path exists in this baseline.
- **Multi-currency/multi-region budget tracking for launch capital spend is out of scope for this
  module** (no `LaunchBudget` entity is defined). Launch progress tracking (phases/milestones/tasks)
  and capital budget tracking are different concerns with different owners; if this capability is
  requested, it belongs to a future Finance/Procurement-adjacent module (Module 19 Inventory/
  Procurement is the closest existing analog) rather than being bolted onto this module's rollup
  model.
- **A `LaunchTemplate`'s tasks are always 1:1 with `LAUNCH_TASK` WorkItems; no template task maps to
  zero or many WorkItems in this version.** This keeps the expansion engine (LLD §4.1) and the
  progress-rollup math (FR-LAUNCH-021) simple and auditable. A future requirement to fan a single
  template task out to N WorkItems (e.g., "post job req" fanning out per open role) will be modeled
  as an explicit template-level repeat/fan-out declaration in a later revision; it is not supported
  implicitly today.
