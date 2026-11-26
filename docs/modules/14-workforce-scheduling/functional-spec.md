# Workforce Scheduling & Labor Management — Functional Specification

Module corresponds to a new **Tier-1 competitive gap closure epic** (not in the original v2.0
baseline — see `_context/new-modules-addendum.md` Module Map row 14). Source: `_context/source-spec.md`
§3 (Personas), §4 (Principles), §7 (Work Orchestration, for the boundary this module must respect),
§16 (FSD Architecture), §17.1 (Domain Model conventions), §23 (NFRs). Shortcode: **WFS**.

## 1. Purpose and Scope

### 1.1 Purpose

Workforce Scheduling & Labor Management schedules **people** — it builds, publishes, and maintains
the roster of who is working which shift, at which location, doing which role, and for how long.
This is the platform's authoritative source of labor-shift data: shift templates, concrete shift
instances, employee availability, shift assignment/eligibility, shift swap and pickup requests, basic
coverage-gap detection, and basic labor forecasting.

This module exists because Work Orchestration's `SHIFT_BASED` schedule mode (source-spec §7.2,
FR-WORK-022) has always assumed "shift instances" as an input but the original baseline never
specified where those shift instances come from. This module is that source. Closing this gap is
also a direct competitive response to Connecteam, Workjam, and Zenput-class competitors whose
scheduling module is core to frontline retail/QSR/hospitality deals.

### 1.2 In Scope

- `ShiftTemplate` authoring: role, location, recurring time window, break rules.
- `Shift` (concrete shift instance) creation, editing, and the draft → published schedule lifecycle,
  including a "changes since last publish" diff view.
- `AvailabilityWindow` management: employees set recurring and one-off availability; time-off
  requests as a distinguished availability type.
- `ShiftAssignment`: manual (drag-and-drop) assignment, and a basic rule-based auto-fill that matches
  availability + role eligibility + a simple labor-cost heuristic.
- `SwapRequest` workflow: associate-initiated shift swap or pickup, eligible-peer acceptance, and
  manager approval gated by tenant policy.
- Coverage gap detection and alerting (required headcount/role vs. currently assigned).
- Basic labor forecasting from historical demand signals (rule-based, not ML, for MVP).
- Multi-location employees: a person eligible to be scheduled at more than one `OrganizationNode`.
- Publishing shift instances that Work Orchestration's `SHIFT_BASED` schedule mode consumes as its
  materialization anchor (FR-WORK-022).

### 1.3 Out of Scope

- **This module does not schedule work items.** Scheduling *what work happens during a shift*
  (a checklist, an audit, an opening/closing task list) is entirely owned by **Work Orchestration**
  (EPIC-03). This module produces `Shift` rows (people, time, location, role); Work Orchestration
  reads those rows to materialize `WorkItem`s anchored to shift boundaries. The two schedulers are
  deliberately distinct services with distinct data models — see High-Level Design §1 for the
  explicit "Scheduling Service vs. Scheduling Engine" naming.
- Time clock punches, geofenced clock-in/out, timesheets, and payroll-ready hours — **Time &
  Attendance module (15)**. This module owns the *plan*; Time & Attendance owns the *actual*.
  Variance reporting (planned vs. actual) is a Time & Attendance capability that reads this module's
  `Shift`/`ShiftAssignment` rows as its planned-hours input.
- Payroll calculation, wage-rule compliance beyond MVP rest-period gating, and pay export — owned by
  Integration Hub (module 12) payroll connectors, fed from Time & Attendance's timesheets, not from
  this module directly.
- Employee identity, role definitions, and location hierarchy — **Identity & Organization**
  (EPIC-01/02). This module consumes `User`, `Role`, `OrganizationNode`, `UserScope`.
- Advanced ML-driven demand forecasting and workforce optimization (multi-week what-if scenario
  planning, ML-based auto-scheduling) — flagged Phase 2/3; see §16 Resolved Decisions.
- Long-term leave/HR case management (FMLA-class, disability accommodation tracking) — out of scope
  entirely for this module; `AvailabilityWindow` time-off requests are operational scheduling inputs
  only, not an HR leave-management system of record.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Store/Site Manager | Primary builder: authors `ShiftTemplate`s, builds and publishes the weekly/period schedule, resolves coverage gaps, approves swaps |
| Frontline Associate | Views own published shifts (and, where enabled, the full location schedule), sets availability/time-off, requests swaps/pickups, accepts peers' swap offers |
| Regional/District Manager | Views labor cost rollups and coverage health across territory; approves swaps only where tenant policy escalates approval above site-manager level (e.g., cross-location swaps) |
| Enterprise Admin | Configures labor policy engine (rest-period rules, overtime thresholds, swap approval policy) at the tenant level |
| AI Agent (Compliance Monitor-class, future) | Out of scope for this module's MVP autonomy — flagged in §14 as a Phase 2 candidate for auto-fill/coverage-gap recommendation, following the platform's L0/L1 autonomy pattern (source-spec §20.3) |

## 3. Shift Template Authoring

- **FR-WFS-001 — Shift template fields.** A `ShiftTemplate` defines: `role_id` (Identity/Org `Role`,
  optional — a template may be role-agnostic, e.g. "any associate"), `location_id`
  (`OrganizationNode`), `start_time`/`end_time` (local wall-clock, HH:MM), `days_of_week` (bitmask or
  RRULE-subset recurrence, consistent with Work Orchestration's calendar-recurrence convention),
  `break_rules` (array of `{type: PAID|UNPAID, duration_minutes, earliest_offset_minutes,
  latest_offset_minutes}`), `headcount_required` (minimum staff needed to satisfy this template's
  coverage), and `labor_cost_weight` (optional priority hint for auto-fill, §6).
- **FR-WFS-002 — Template reuse.** A `ShiftTemplate` is a reusable definition; generating a schedule
  for a period expands one or more templates into concrete `Shift` rows for that period (analogous to
  Work Orchestration's `Schedule` → `WorkItem` materialization pattern, but producing `Shift` rows,
  not `WorkItem` rows).
- **FR-WFS-003 — Break rule validation at template authoring time.** Total break duration must not
  exceed the shift duration; a break's `latest_offset_minutes` must not exceed
  `(end_time - start_time) - duration_minutes`. Violations block save with a `422
  INVALID_BREAK_RULE` error.
- **FR-WFS-004 — Template versioning.** Editing a published `ShiftTemplate` does not retroactively
  change already-generated `Shift` rows in `PUBLISHED` state; it only affects future generation runs,
  mirroring the platform-wide immutability principle (source-spec P7, FR-WORK-031 precedent).

## 4. Schedule Publishing Workflow

- **FR-WFS-010 — Draft schedule.** Generating shifts from templates (or authoring ad hoc shifts
  directly) always produces `Shift` rows in status `DRAFT`. Draft shifts are visible only to
  users with a scheduling-admin permission (`schedule:edit`) at that location — never to the
  frontline associate they may end up assigned to.
- **FR-WFS-011 — Publish action.** A manager with `schedule:publish` permission publishes a date
  range (typically one week/pay period) at a location. Publishing transitions all `DRAFT` shifts in
  that range to `PUBLISHED`, snapshots the pre-publish state for diffing, and makes them visible to
  assigned associates (and, per tenant setting `schedule_visibility`, to the whole location roster).
- **FR-WFS-012 — Changes-since-last-publish diff.** Before confirming a republish (editing an
  already-published range), the manager sees a diff view: added shifts, removed shifts, and modified
  shifts (time change, role change, assignee change) since the last publish of that range, each
  tagged with what changed. This mirrors the platform's Impact Preview pattern (source-spec §8.4)
  applied to shift schedules rather than form templates.
- **FR-WFS-013 — Republish notification.** Republishing a range that changes an already-published,
  already-assigned shift notifies the affected associate(s) with the specific delta (old time → new
  time, or "shift removed," or "reassigned to you") rather than a generic "schedule changed" message.
- **FR-WFS-014 — Publish emits shift-instance events.** Publishing a `Shift` emits `shift.published`
  onto the Event Backbone; this is the event Work Orchestration's `SHIFT_BASED` schedule mode
  subscribes to as the authoritative trigger for materializing shift-anchored `WorkItem`s (per the
  addendum's platform-wide note that this module is now the authoritative source of "shift
  instances"). A `DRAFT` shift never emits this event.
- **FR-WFS-015 — Unpublish / cancel a published shift.** A manager can cancel a published shift
  (e.g., location closure, business need drop) with a required reason; this emits `shift.cancelled`,
  which Work Orchestration consumes to cancel any shift-anchored `WorkItem`s still in a
  pre-`IN_PROGRESS` state (per FR-WORK-098's location-closure precedent — in-flight work is not
  retroactively cancelled).

## 5. Availability Window Management

- **FR-WFS-020 — Recurring availability.** An employee (or, on their behalf, a manager) sets a
  weekly recurring `AvailabilityWindow`: `day_of_week`, `start_time`, `end_time`,
  `availability_type = AVAILABLE | PREFERRED | UNAVAILABLE`, optional `location_id` (defaults to all
  locations the employee is eligible at — see §9).
- **FR-WFS-021 — One-off availability overrides.** A specific-date `AvailabilityWindow`
  (`is_recurring=false`, concrete `date`) overrides the recurring pattern for that date only, without
  mutating the recurring rule.
- **FR-WFS-022 — Time-off requests.** A `TIME_OFF` availability type spans one or more full days (or
  partial-day hours) and requires manager approval (`status: PENDING → APPROVED | DENIED`) before it
  is treated as binding `UNAVAILABLE` for auto-fill and manual-assignment warnings. Until approved, it
  is advisory only — a manager can still assign against it but sees a conflict warning (§10).
- **FR-WFS-023 — Availability change after publish.** An employee changing recurring or one-off
  availability that conflicts with an already-published, already-assigned shift does not
  automatically unassign them; it flags the shift with a `conflicting-availability` warning (§10) on
  the manager's schedule view, requiring explicit manager action (reassign, or dismiss the warning
  with an override reason).
- **FR-WFS-024 — Multi-location availability.** Availability windows are, by default, scoped to
  "all locations this employee is eligible at" (§9) unless the employee sets a location-specific
  window, allowing an employee eligible at two stores to mark themselves unavailable at one without
  affecting the other.

## 6. Shift Assignment

- **FR-WFS-030 — Manual assignment.** A manager assigns an employee to a `DRAFT` (or, with an
  override, `PUBLISHED`) `Shift` via drag-and-drop on a week-grid view. Each assignment is validated
  against eligibility (§9), availability (soft warning, not a hard block, per FR-WFS-023's philosophy
  of surfacing conflicts rather than silently refusing), and the labor-law policy engine (§11) before
  being accepted.
- **FR-WFS-031 — Auto-fill (basic, rule-based).** A manager can trigger auto-fill for some or all
  unassigned `DRAFT` shifts in a range. The algorithm (pseudocode in Low-Level Design §4) ranks
  eligible, available candidates per shift by: (1) role match, (2) availability match
  (`PREFERRED` > `AVAILABLE` > no stated preference > conflicting), (3) rest-period compliance
  (candidates who would violate a configured rest-period rule are excluded, not just deprioritized),
  (4) a basic labor-cost heuristic — lowest blended hourly cost first among remaining ties, with a
  configurable cap on weekly hours per employee to avoid unintended overtime accrual. Auto-fill is
  advisory: results populate the `DRAFT` grid for manager review, never auto-publish.
- **FR-WFS-032 — Reassignment.** Reassigning a shift follows the same eligibility/availability/policy
  validation as initial assignment; reassignment on a `PUBLISHED` shift triggers the republish-diff
  and notification path (FR-WFS-012/013).
- **FR-WFS-033 — Double-booking prevention.** An employee cannot be assigned two `Shift`s at
  overlapping times (including cross-location overlap for multi-location employees) — see Validation
  §12 for the exact overlap rule including midnight-spanning shifts.
- **FR-WFS-034 — Open shift / self-service pickup.** A `DRAFT` or `PUBLISHED` shift with no assignee
  (or an assignee who dropped it via an approved swap, §7) can be flagged `open_for_pickup=true`,
  making it visible to eligible associates in a "Open Shifts" list for direct self-claim, subject to
  the same eligibility/availability/policy checks and to manager approval if tenant policy requires
  it for pickups (distinct from the swap-specific approval policy in §7).

## 7. Swap and Pickup Request Workflow

- **FR-WFS-040 — Swap initiation.** An associate assigned to a `PUBLISHED` shift requests a swap,
  either targeted (naming a specific eligible peer) or open (broadcast to all eligible peers at that
  location/role). A pickup request (associate wants to drop a shift with no direct swap-back) is
  modeled as the same `SwapRequest` entity with `swap_type=PICKUP` (no `target_shift_id`) versus
  `swap_type=SWAP` (both parties' shifts change).
- **FR-WFS-041 — Peer eligibility for swap/pickup.** A candidate peer must satisfy the same
  eligibility rules as any assignment (§9: role, location), must not already be assigned an
  overlapping shift, and must not violate the labor-law policy engine (§11) if the swap were
  accepted — ineligible peers are excluded from the broadcast list, not merely warned.
- **FR-WFS-042 — Peer acceptance.** An eligible peer accepts an open or targeted `SwapRequest`,
  moving it to `PENDING_MANAGER_APPROVAL` (or directly to `APPROVED` if tenant policy
  `swap_requires_manager_approval=false` for same-role/same-location swaps — see §16 for the
  platform default).
- **FR-WFS-043 — Manager approval per tenant policy.** Tenant-configurable `SwapApprovalPolicy`
  determines whether manager approval is required, and at what scope: `NEVER`, `ALWAYS`,
  `CROSS_LOCATION_ONLY` (same-location peer swaps auto-approve; swaps involving a multi-location
  employee's other location require the receiving location manager's approval), or
  `POLICY_VIOLATION_ONLY` (auto-approve unless the swap would trip a labor-law or overtime
  threshold, in which case it escalates for explicit approval).
- **FR-WFS-044 — Approval updates both shifts atomically.** On final approval, both shifts'
  `ShiftAssignment` rows update in a single transaction: for a `SWAP`, each party takes the other's
  shift; for a `PICKUP`, the accepting peer becomes the sole assignee and the original assignee is
  removed. This emits `shift.reassigned` for both affected `Shift` rows, and republish-diff/
  notification behavior (FR-WFS-012/013) applies since these are `PUBLISHED` shifts.
- **FR-WFS-045 — Rejection/expiry.** A manager can reject a pending swap with a reason (both parties
  notified, no shift change). An unaccepted open `SwapRequest` expires after a tenant-configured
  window (default 48 hours before the shift start, or immediately at shift start, whichever is
  sooner) and reverts to the original assignee unchanged.
- **FR-WFS-046 — Swap creating a labor-law violation.** If an otherwise-accepted swap would place
  either party in violation of a configured rest-period or weekly-hours rule (§11), the swap cannot
  reach `APPROVED` automatically — it is forced into `PENDING_MANAGER_APPROVAL` regardless of policy
  (overriding an `ALWAYS`-auto-approve or `NEVER`-require setting for that one case) with the specific
  violation surfaced to the approver, who may still override with an audit-logged reason where tenant
  policy allows manager override of labor-law warnings (a hard-block tenant setting is also
  supported — see §11).

## 8. Coverage Gap Detection and Alerting

- **FR-WFS-050 — Coverage requirement source.** Required headcount per role/time-slot at a location
  comes from the `ShiftTemplate.headcount_required` values active for that slot, aggregated across
  overlapping templates when more than one applies.
- **FR-WFS-051 — Gap computation.** For each location and rolling time window (default: next 14
  days, aligned with Work Orchestration's materialization-window convention for consistency), the
  system computes `required_headcount - assigned_headcount` per `(location, role, time-slot)`. A
  positive gap is a coverage gap.
- **FR-WFS-052 — Gap alerting.** Gaps within a tenant-configurable proximity window to shift start
  (default 72 hours) generate a `coverage.gap_detected` event and surface a coverage-gap warning
  (§10) on the manager's schedule view and, optionally, an Exceptions-style notification, deliberately
  modeled after Work Orchestration's Exceptions pattern (FR-WORK-080) though this module owns its own
  gap entity rather than creating a `WorkItem`.
- **FR-WFS-053 — Gap resolution suggestions.** Where auto-fill (§6) can identify at least one
  eligible, available, policy-compliant candidate for a gap, the gap alert surfaces that candidate as
  a one-click "fill" suggestion; where none exists, the alert states "no eligible candidate found" so
  the manager knows to look outside the auto-fill pool (e.g., open-shift broadcast to a wider pool,
  overtime approval, or a vendor/temp channel — out of scope integration, flagged §14).

## 9. Eligibility and Multi-Location Employees

- **FR-WFS-060 — Eligibility model.** An employee is eligible for a `Shift` at a `ShiftTemplate`'s
  `role_id`/`location_id` if they hold a `UserScope` granting that role at that `OrganizationNode`
  (Identity/Org), consistent with the platform-wide scoping rule (conventions.md).
  Role-inheritance-from-parent-location (source-spec §6.1) applies identically here.
- **FR-WFS-061 — Multi-location employees.** An employee may hold `UserScope` rows at more than one
  location; this module's assignment, availability, and double-booking checks operate across **all**
  of an employee's eligible locations simultaneously (not per-location in isolation), so a
  cross-location double-booking is caught with the same rigor as a same-location one (FR-WFS-033).
- **FR-WFS-062 — Home location vs. secondary location.** An employee has exactly one `home_location_id`
  (drives default schedule visibility, default availability scope per FR-WFS-024) and zero or more
  secondary eligible locations; this distinction is a scheduling convenience only and does not affect
  authorization, which remains governed purely by `UserScope`.

## 10. Basic Labor Forecasting

- **FR-WFS-070 — Forecast input.** For MVP, labor demand forecasting uses simple historical signals
  per location: trailing N-week average of (a) transaction/traffic volume if fed from a POS/traffic
  integration (Integration Hub), or (b) historical scheduled-vs-worked hours if no external signal is
  connected, with a simple day-of-week and week-of-year seasonality adjustment (not a trained ML
  model — see §16 Resolved Decisions).
  Actual demand/traffic input arrives via the Integration Hub (module 12) as a configured connector;
  this module does not build integration-specific ingestion.
- **FR-WFS-071 — Forecast output.** A `LaborForecast` row per `(location, date, role)` gives a
  recommended headcount and estimated labor hours, used as an input suggestion when authoring or
  adjusting `ShiftTemplate.headcount_required` for an upcoming period — it does not automatically
  rewrite templates.
- **FR-WFS-072 — Forecast accuracy tracking.** Forecasted vs. actual-scheduled (and, once Time &
  Attendance data is available, vs. actual-worked) headcount is tracked per location/period to
  measure forecast usefulness, feeding the Analytics module's labor KPIs (§13).

## 11. Labor-Law / Rest-Period Policy Engine

- **FR-WFS-080 — Configurable policy, not hardcoded rules.** Minor labor-law and rest-period
  constraints (minimum rest between shifts, maximum consecutive days worked, maximum hours per day/
  week, minor/youth-employment hour restrictions, meal-break mandates) are modeled as a tenant- and
  jurisdiction-configurable `LaborPolicy` rule set (JSON-schema-validated rule rows), never as
  application code constants — jurisdictions vary too widely (by country, state/province, and
  sometimes municipality) to hardcode. See Low-Level Design §4.2 for the rule evaluation pseudocode.
- **FR-WFS-081 — Enforcement modes.** Each `LaborPolicy` rule has an `enforcement_mode ∈ {HARD_BLOCK,
  WARN_OVERRIDE}`. `HARD_BLOCK` rules cannot be bypassed by a manager through the UI (only a
  tenant-admin policy change can permit the schedule); `WARN_OVERRIDE` rules allow a manager to
  proceed with a required, audit-logged override reason.
- **FR-WFS-082 — Policy evaluation points.** The policy engine is evaluated at: manual assignment
  (§6), auto-fill candidate ranking (excludes `HARD_BLOCK` violators outright), swap acceptance/
  approval (§7), and publish time (a final sweep across the range being published, since availability
  or other assignments may have changed since individual assignments were made).
- **FR-WFS-083 — Jurisdiction resolution.** A `LaborPolicy` is scoped to an `OrganizationNode`
  subtree (e.g., a specific state/country's stores) with the same effective-dated inheritance model
  as `UserScope` — a location without an explicit override inherits its parent node's policy.

## 12. UX States

| State | Behavior |
|---|---|
| Empty schedule | A location/period with no `Shift` rows yet shows a "Start from a template" / "Build from scratch" empty state, distinct from a period that has shifts but none published yet (see next row). |
| Unpublished-draft banner | Any period containing `DRAFT` shifts shows a persistent banner ("N shifts not yet published — associates cannot see these") on the manager's schedule view until every shift in the visible range is `PUBLISHED` or explicitly left as an intentional draft placeholder. |
| Coverage-gap warning | A time-slot with a computed gap (§8) shows an inline amber/red indicator on the week-grid (amber = gap outside the alerting proximity window, red = within it); clicking shows required vs. assigned and the fill-suggestion affordance. |
| Swap-pending-approval | The requesting associate, the accepting peer, and the manager each see a distinct pending-state card: requester sees "Waiting for a peer to accept" or "Waiting for manager approval" depending on stage; manager sees the swap in an Approvals-style queue mirroring Work Orchestration's Approvals UX pattern (FR-WORK-081) with both shifts' before/after shown side by side. |
| Conflicting-availability warning | A shift assigned against a stated `UNAVAILABLE`/denied-availability window shows a distinct warning badge on the manager's grid (not a hard block, per FR-WFS-023) with the specific conflicting window named; dismissing requires an override reason, audit-logged. |
| Open shift (associate view) | Associates see an "Open Shifts" list of pickup-eligible shifts distinct from their own assigned shifts, filterable by location for multi-location employees. |
| Labor-law violation (hard block) | Assignment/swap action is disabled with an inline message naming the specific violated rule (e.g., "Minimum 10-hour rest period between shifts not met") rather than a generic validation error. |
| Labor-law violation (warn-override) | Action proceeds only after an explicit confirm dialog requiring a typed override reason; the resulting assignment/swap is visibly flagged on the grid so a later reviewer can see an override was used. |
| Offline (mobile, associate) | Per the platform Offline Design Rule (source-spec §9.2): the associate's own published shifts and their availability-setting/swap-request actions are available offline via the standard local-DB + sync pattern (queued swap requests sync on reconnect); manager schedule-building (drag-and-drop, publish) is a web-admin, online-only surface for MVP — see §14. |

## 13. Validation and Edge Cases

- **FR-WFS-090 — Double-booking prevention.** Two `Shift` rows for the same employee with overlapping
  `[start_at, end_at)` intervals (in UTC, after resolving each shift's local wall-clock time via its
  location's timezone) are rejected at assignment time with `409 DOUBLE_BOOKED`, including the
  cross-location case (FR-WFS-061). Adjacent-but-not-overlapping shifts (e.g., one ends exactly when
  another starts) are allowed but flagged as a rest-period policy check (§11), not a double-booking.
- **FR-WFS-091 — Shift spanning midnight.** A `Shift` may have `end_at` on the calendar day after
  `start_at`; all interval math (double-booking, rest-period gap computation, coverage-slot bucketing)
  uses absolute UTC instants derived from location-local wall-clock times, never a same-calendar-day
  assumption. A midnight-spanning shift is attributed to its `start_at` date for "which day's
  schedule is this on" display purposes, tenant-configurable to end-date attribution if a tenant's
  operational convention differs (e.g., overnight hospitality shifts commonly labeled by the night
  they start).
- **FR-WFS-092 — DST-crossing shifts.** A shift template expressed as a local wall-clock window
  (e.g., 22:00–06:00) computes its actual UTC duration correctly across a DST transition (a
  spring-forward night is one hour shorter in elapsed time, a fall-back night one hour longer) —
  duration for pay/labor-cost purposes is the true elapsed interval, not a fixed assumed duration,
  consistent with the DST-handling convention established in Work Orchestration (FR-WORK-091).
- **FR-WFS-093 — Employee availability changes after publish.** See FR-WFS-023; the shift is not
  auto-unassigned, only flagged. Edge case: if the availability change is an *approved* time-off
  request (FR-WFS-022) rather than a soft preference change, the flag severity is elevated (red, not
  amber) since a binding approved absence conflicting with a published shift is a materially higher
  risk of no-show than a soft preference conflict.
- **FR-WFS-094 — Swap creating a labor-law violation.** See FR-WFS-046.
- **FR-WFS-095 — Concurrent swap acceptance race.** If two eligible peers attempt to accept the same
  open `SwapRequest` concurrently, an optimistic-concurrency check on the `SwapRequest.version`
  column ensures exactly one wins; the loser receives `409 ALREADY_ACCEPTED` naming the accepting
  peer, mirroring Work Orchestration's `CLAIM`-mode assignment race pattern (FR-WORK-099).
- **FR-WFS-096 — Template headcount change after shifts generated.** Editing `headcount_required` on
  a `ShiftTemplate` does not retroactively change already-generated `Shift` rows (FR-WFS-004); it
  does immediately affect the coverage-gap computation (§8), which reads the template's current value
  for any not-yet-generated slot and the generated `Shift` count for already-generated slots — this
  can itself surface a *new* coverage gap against updated requirements without any new shift being
  deleted.
- **FR-WFS-097 — Employee loses eligibility after assignment.** If a `UserScope` granting an
  employee's role at a location is revoked (e.g., role change, transfer) while they hold
  `PUBLISHED` future shifts at that location, those shifts are flagged `NEEDS_ASSIGNMENT` (mirroring
  FR-WORK-094/101's fallback pattern) and surfaced to the manager rather than silently left assigned
  to a now-ineligible person; past/in-progress shifts are unaffected (historical-truth principle,
  source-spec P7).
- **FR-WFS-098 — Publish with unresolved coverage gaps.** Publishing is not blocked by an open
  coverage gap (business reality: sometimes you publish understaffed and keep working the gap) but
  requires the manager to acknowledge a confirmation listing the open gaps being published anyway,
  which is itself audit-logged.
- **FR-WFS-099 — Zero-length or negative-duration shift.** `end_at` must be strictly after `start_at`
  after timezone/midnight resolution (FR-WFS-091); rejected at authoring time with `400
  INVALID_SHIFT_DURATION`.

## 14. Non-Functional Requirements

- **NFR-WFS-01 — Schedule build responsiveness.** The manager week-grid (drag-and-drop) must reflect
  an assignment change optimistically in the UI within 100ms and confirm/rollback against the server
  within the platform's general API p95 < 300ms target (source-spec §23).
- **NFR-WFS-02 — Publish latency at scale.** Publishing a week's schedule for a single location
  (typically tens of shifts) completes, including the diff computation and notification fan-out
  trigger, within p95 ≤ 3 seconds; a multi-location bulk-publish (corporate-initiated) processes as a
  bulk job (§ LLD background jobs) rather than a single synchronous request.
- **NFR-WFS-03 — Coverage-gap detection freshness.** Coverage-gap recomputation reflects an
  assignment, availability, or template change within 60 seconds (near-real-time, event-driven
  recomputation, not a nightly batch) so a manager sees the effect of their own edit immediately.
- **NFR-WFS-04 — Scale.** Designed for the platform's general scale target (source-spec §23): up to
  100k locations, with a typical location generating on the order of 50–500 shift-instances/week —
  see High-Level Design §5 for the resulting shift-row volume and partitioning approach.
- **NFR-WFS-05 — Swap round-trip latency.** From peer acceptance to manager-approval-queue visibility
  (where approval is required) is p95 ≤ 5 seconds, consistent with Work Orchestration's approval
  step-advancement NFR (NFR-WORK-06), since swap approval reuses the same approval-notification
  latency expectation.

## 15. Dependencies on Other Modules

| Module | Direction | Nature |
|---|---|---|
| Identity & Organization | Consumes | `User`, `Role`, `UserScope` (role/location eligibility, §9), `OrganizationNode` (location, timezone); consumes `user.deactivated`, `user.role_changed` for eligibility-loss handling (FR-WFS-097) |
| Work Orchestration | Produces (consumed by) | Publishes `shift.published`/`shift.cancelled`/`shift.reassigned` events that Work Orchestration's `SHIFT_BASED` schedule mode subscribes to as the authoritative source of "shift instances" (FR-WORK-022) for materializing shift-anchored `WorkItem`s. This module never creates or reads `WorkItem` rows directly. |
| Time & Attendance (module 15) | Produces (consumed by) | Time & Attendance reads this module's `Shift`/`ShiftAssignment` rows as the "planned" side of its planned-vs-actual variance comparison; this module does not read Time & Attendance data |
| Integration Hub | Consumes | Optional POS/traffic-volume connector as an input signal to Labor Forecasting (§10); payroll export is downstream of Time & Attendance, not this module |
| Analytics | Produces | Shift/assignment/swap/coverage-gap lifecycle events feed labor-cost and coverage KPIs (labor cost %, coverage-gap rate, swap-approval rate, forecast accuracy) |
| Notification Hub | Produces (calls) | Publish notifications, swap request/acceptance/approval notifications, coverage-gap alerts routed through Notification Hub, never delivered directly |

## 16. Resolved Decisions

- **Labor-cost optimization for MVP is rule-based, not ML (FR-WFS-031).** The auto-fill ranking is
  role match → availability preference → rest-period compliance → lowest blended cost, exactly as
  specified in §6; no solver or trained model ships at MVP. Rationale: a multi-constraint ML optimizer
  requires historical assignment/outcome data volume this platform will not have at launch, and adds
  solver-infrastructure complexity and an unexplainable-ranking UX risk disproportionate to MVP value;
  rule-based ranking is fully explainable to a manager ("why was this person suggested") which matters
  more than marginal cost optimization at this stage. Revisit as a Phase 2/3 capability once the
  warehouse holds sufficient assignment/outcome history.
- **Default swap approval policy is `CROSS_LOCATION_ONLY` (§7, FR-WFS-043).** Same-location, same-role
  swaps auto-approve; cross-location swaps and any policy-violating swap always require explicit
  manager approval. This is the tenant's default at provisioning time and remains
  tenant-configurable to `NEVER`/`ALWAYS`/`POLICY_VIOLATION_ONLY` per the platform's
  progressive-complexity principle (P6); `CROSS_LOCATION_ONLY` is chosen as the default because it is
  the best balance of associate self-service speed (the common same-location case) against the one
  scenario (cross-location swap) where a receiving-location manager genuinely needs visibility before
  a stranger shows up on their roster.
- **Manager schedule-building is a web-admin, online-only surface for MVP.** The drag-and-drop
  schedule grid (build/publish) is not built for offline mobile use; associates' own view/
  availability/swap actions remain offline-capable per the standard mobile pattern (source-spec §9.2).
  This is a firm MVP scope line, not a placeholder: offline-editable schedule state is a materially
  harder conflict-resolution problem (many-to-many shift/employee constraint graph, concurrent manager
  edits) than most offline forms, and site managers overwhelmingly build schedules from a back-office
  computer or tablet with connectivity, not a phone in a dead zone. Reconsider only if field telemetry
  shows material manager usage attempting schedule edits offline.
- **`LaborForecast` (§10) is a scheduling-input suggestion only, never an auto-publish trigger.** A
  human always builds/confirms the schedule even when a forecast is available. This is a firm,
  permanent design constraint (not just an MVP stopgap), required by Policy-Before-AI (P5) and
  Human-in-the-Loop (source-spec §13.4): labor-cost/headcount decisions materially affect both
  business cost and employee hours, so no automated process may commit a published schedule without a
  manager's explicit action, regardless of forecast confidence.
- **Cross-tenant vendor/temp-staff shift fill is out of scope for this module and is not planned to
  move here.** A temp-agency worker filling an open shift is explicitly the responsibility of Field
  Service & Vendor Scheduling (module 25)'s `VENDOR` assignee-type pattern, applied at the `WorkItem`
  level once a vendor technician is dispatched; this module's `Shift`/`ShiftAssignment` model is
  scoped to `User` rows with an `UserScope` (i.e., the tenant's own employees) and will not gain a
  parallel vendor-assignee concept. Rationale: introducing a second assignee-type concept into the
  roster model (as opposed to the work-execution model, which already has one) would duplicate
  eligibility/scoping logic that Identity/Org already owns for employees, for a use case (temp-agency
  shift fill) that is niche relative to this module's core competitive scope; if demand emerges, it is
  a Field Service module extension, not a Workforce Scheduling one.
- **`ShiftAssignment` supports exactly one primary assignee per `Shift`, permanently for this data
  model** (no split-shift co-coverage within a single `Shift` row). A co-coverage need (two people
  covering the same slot) is modeled as two overlapping `ShiftTemplate`-derived shifts, each with
  independent `headcount_required` and its own `ShiftAssignment`, keeping the one-assignee-per-shift
  invariant simple and consistent with the double-booking/rest-period math in §13, which assumes a
  single assignee per shift row throughout.
