# Time & Attendance — Functional Specification

Module corresponds to a new Tier-1 competitive-gap-closure epic (not in the original v2.0
baseline). Shortcode: **TA**. Source: `_context/source-spec.md` (platform principles P1/P2/P7/P9,
§9 Frontline Mobile and Offline, §17 Domain Model, §18 API/Event conventions, §19 Offline Sync and
Conflict Resolution — especially §19.4 "client clock cannot be trusted"), `_context/conventions.md`
(tech stack, multi-tenancy, state-machine conventions), and `_context/new-modules-addendum.md`
(Module Map row 15). This module closes a Tier-1 gap versus Connecteam, Wooqer, and Operandio-class
competitors, all of which bundle geofenced time clock, break compliance, and timesheet approval
directly into the frontline app the associate already uses for task execution.

## 1. Purpose and Scope

### 1.1 Purpose

The Time & Attendance module lets a Frontline Associate clock in and out of a shift from the same
mobile app they use for daily work, with a geofence check and optional anti-buddy-punching capture,
lets a Store/Site Manager review and correct exceptions and approve the resulting timesheet, and
produces an accurate, auditable timesheet ready for export to a payroll system. It reuses the
platform's offline-first capture model (Module 04) so a clock event is never lost to a network gap,
and it reuses Workforce Scheduling (Module 14) as the source of the planned shift a punch is
compared against.

### 1.2 In Scope

- Clock-in / clock-out capture (mobile, geofenced), including break start/end punches.
- Geofence zone configuration and radius-based validation per location.
- Optional photo capture and/or device biometric confirmation at clock-in/out for anti-buddy-punching.
- Break policy configuration (mandatory/optional breaks, minimum duration, paid/unpaid, reminder
  timing) and break-taken enforcement/nudging.
- Timesheet generation: aggregating raw `TimeClockEvent` rows into daily and weekly totals
  (regular, overtime, break-deducted, unpaid) per employee per pay period.
- Exception detection and correction workflows: missed punch, early clock-in, late clock-out,
  missed break, punch without a matching scheduled shift.
- Manager approval workflow for timesheets and for individual correction requests, prior to
  payroll export.
- Overtime calculation using a per-tenant configurable rule set (daily threshold, weekly threshold,
  consecutive-day premium, rounding rules) — not hardcoded to one jurisdiction's labor law.
- Multi-location clock-in: an employee clocking in at a location other than their assigned home
  location, where policy permits it.
- Producing the `timesheet.approved` event and the exportable timesheet payload consumed by the
  Integration Hub's payroll connector (Module 12).

### 1.3 Out of Scope (owned by other modules, referenced here)

- **Running payroll, calculating pay, tax withholding, or issuing pay statements** — this module
  produces an approved `Timesheet`/`TimesheetLine` set; the payroll engine of record (Workday,
  ADP, SAP-class systems) consumes it via the Integration Hub (Module 12) outbound connector. Time
  & Attendance never computes gross/net pay.
- Shift planning, shift templates, availability, shift swaps, labor forecasting — **Workforce
  Scheduling (Module 14)**. This module reads `Shift`/`ShiftAssignment` for planned-vs-actual
  comparison only; it does not create or modify shifts.
- Offline local database, sync protocol, camera/capture UX chrome, background sync scheduling —
  **Mobile & Offline (Module 04)**. Clock events are captured as an offline-first evidence-like
  submission using the same local queue and sync contract; this module defines the domain payload,
  not the sync transport.
- Employee identity, roles, org hierarchy, home-location assignment — **Identity & Organization
  (Module 01)**. This module reads `User`/`OrganizationNode`/`UserScope`, it does not manage them.
- General WorkItem scheduling/approval routing infrastructure — **Work Orchestration (Module 02)**.
  A missed-punch correction request reuses the platform's approval-routing pattern but is modeled
  as its own `CorrectionRequest` entity (see Module Map rationale: a lightweight approval, not a
  full WorkItem), because it must carry pay-affecting before/after punch values that a generic
  WorkItem schema does not model.
- Labor law jurisdiction rules engine as a legal-compliance product (e.g. auto-detecting
  California meal-break penalties) — **out of scope for MVP**; the module ships a configurable
  rule primitive (§3.6) that a tenant's HR/Payroll admin (or system integrator) configures to match
  their jurisdiction. See Resolved Decisions §9.

## 2. Personas

| Persona | Primary use of this module |
|---|---|
| Frontline Associate | Clocks in/out and starts/ends breaks from the mobile app, dozens of times/week; submits missed-punch correction requests; views their own timesheet. |
| Store/Site Manager | Reviews daily/weekly exceptions (missed punch, late clock-out, geofence violation) for their location's team; approves or rejects correction requests; approves the timesheet before it locks for payroll export. |
| Payroll/HR Admin | Configures break policy, geofence zones, overtime rule, rounding rule, and pay-period calendar per tenant (and optionally per location group); reviews/exports approved timesheets; does not typically clock in/out. |
| Regional/District Manager | Read-only visibility into exception rates and approval SLAs across their territory (via Analytics/Control Tower, Module 11), not primary approver in the default flow. |

## 3. Functional Requirements

### 3.1 Clock-In / Clock-Out

- **FR-TA-001** — The mobile app exposes a persistent "Clock In" / "Clock Out" primary action from
  the Today screen (Module 04 §3.1) whenever the associate has an eligible open shift window
  (default: from `shift.starts_at - grace_before_min` to `shift.ends_at + grace_after_min`,
  tenant-configurable, default 15 min each side) or, where policy allows unscheduled punches
  (§3.7), at any time.
- **FR-TA-002** — On tap, the app captures device GPS coordinates and evaluates them against the
  `GeofenceZone`(s) configured for the target location. If the device is within the configured
  radius, the clock event proceeds; if outside, the app enters the **outside-geofence-blocked**
  state (§4) unless the tenant policy for that location/role is `geofence_mode: ADVISORY`, in which
  case the punch is allowed but flagged `geofence_violation: true` for manager review.
- **FR-TA-003** — Geofence radius is configured per `GeofenceZone` (default 150m, tenant/location
  configurable down to 25m or up to 1000m); a location may have multiple zones (e.g. a large
  distribution center with several building entrances) and a punch passes if it falls inside any
  associated zone.
- **FR-TA-004** — GPS unavailable (no fix within an 8-second timeout, indoors/underground) does not
  silently block the punch: the app falls back to the last-known-good fix if captured within the
  last 5 minutes and flags `location_source: STALE_FIX`; with no usable fix at all, the tenant
  policy decides (`require_gps: true` blocks the punch with an explanation and a manual-override
  path for the manager; `require_gps: false`, the default, allows the punch with
  `location_source: UNAVAILABLE` flagged for review).
- **FR-TA-005** — **Anti-buddy-punching capture**, per-tenant configurable and applied per
  role/location: `NONE` (default), `PHOTO` (front-camera selfie captured at clock-in/out, stored as
  `Evidence`, reviewable by manager, not biometrically matched by default), or `BIOMETRIC`
  (on-device biometric confirmation — Face ID/fingerprint — matched against the enrolled device
  owner, not a server-side face-match; see Resolved Decisions §9 for why this is decided as
  device-local-only).
- **FR-TA-006** — A clock-in requires no prior open clock-in for the same employee (duplicate
  clock-in prevention, §5); attempting to clock in while already clocked in surfaces the current
  open punch and offers "Clock Out Now" instead of creating a second open event.
- **FR-TA-007** — Clock-out requires a prior open clock-in for that employee; a clock-out with no
  open clock-in (e.g. app state lost) is handled as a **missed-punch correction** (§3.4) rather than
  silently rejected.
- **FR-TA-008** — Each clock event stores both `client_captured_at` (device clock, display/ordering
  hint only) and `server_received_at` (server-assigned, authoritative) per source-spec §19.4; all
  payroll-affecting duration math uses `server_received_at` reconciled per the offline rules in §5.
- **FR-TA-009** — Clock events are versioned against the employee's `template_version`-style
  configuration snapshot is not required (clock events are not form submissions), but each event
  does snapshot the `geofence_zone_id`, `break_policy_id`, and `overtime_rule_version` in effect at
  capture time, so later policy changes never retroactively alter an already-recorded punch's
  evaluation context.
- **FR-TA-010** — Every clock-in/out emits a canonical-style domain event (`time.clocked_in`,
  `time.clocked_out`) onto the Event Backbone per the platform envelope (source-spec §18.4),
  consumed by Workforce Scheduling (actual-vs-planned) and Analytics.

### 3.2 Multi-Location Clock-In

- **FR-TA-011** — An employee may clock in at a `GeofenceZone` belonging to a location other than
  their `home_location_id`, when: (a) they hold an active `ShiftAssignment` at that location for the
  current shift window (covering/borrowed shifts), or (b) the tenant policy flag
  `allow_cross_location_punch: true` is set for their role, in which case the punch is accepted and
  flagged `cross_location: true` with the worked location recorded distinctly from the home location
  on the `TimeClockEvent` and on the resulting `TimesheetLine` (for cost-center/labor-allocation
  reporting).
- **FR-TA-012** — Where neither condition in FR-TA-011 holds, a cross-location punch attempt is
  blocked with an explanatory state and a manager-override path (manager can approve the punch
  after the fact via the correction workflow, §3.4).
- **FR-TA-013** — Timesheet aggregation (§3.3) always attributes worked hours to the location
  actually punched (`worked_location_id`), while retaining `home_location_id` for the employee's
  default reporting rollup; a payroll export can be sliced by either dimension.

### 3.3 Break Policy Configuration & Enforcement

- **FR-TA-014** — Payroll/HR Admin configures one or more `BreakPolicy` records per tenant
  (optionally scoped to a location group or role), each defining: break type (`MEAL` / `REST`),
  paid or unpaid, minimum duration, maximum duration (optional), whether mandatory, the shift-length
  threshold that triggers the requirement (e.g. "one 30-min unpaid meal break required for shifts
  ≥ 6 hours"), and how many breaks of that type a shift of a given length requires.
- **FR-TA-015** — When a shift in progress crosses a configured trigger threshold without a break
  having been started, the app surfaces a **break-reminder-nudge** (§4): a local (offline-capable)
  notification plus an in-app banner; the reminder is device-scheduled from the cached policy so it
  fires with zero connectivity, consistent with Module 04's local-notification pattern.
- **FR-TA-016** — Break start/end are captured as their own `TimeClockEvent` rows
  (`event_type: BREAK_START` / `BREAK_END`) nested under the open clock-in; a break-end without a
  matching open break-start (or vice versa) is a missed-punch correction case, same as §3.4.
- **FR-TA-017** — **Break-taken confirmation**: at clock-out, if a mandatory break was required for
  the shift length but no qualifying break (meeting minimum duration) was recorded, the app shows a
  blocking confirmation step ("No qualifying meal break was recorded for this shift — confirm this
  is correct, or add one now") before allowing clock-out to complete; the confirmation (or the
  added break) is recorded on the `TimeClockEvent`/`TimesheetLine` as `break_compliance_flag` for
  manager and (where applicable) regulatory visibility. This module does not adjudicate legal
  penalty pay for a missed meal break — it records the fact so the tenant's payroll system or a
  jurisdiction-specific rule extension can apply it.
- **FR-TA-018** — A break that runs longer than `max_duration` (when configured) does not block
  clock-in-from-break; it is flagged `break_overrun: true` on the event for manager review and
  timesheet aggregation deducts only up to the paid-break allowance where the policy defines one
  (unpaid breaks are deducted in full regardless of overrun, up to a configurable cap to prevent an
  extreme outlier value from corrupting a daily total — default cap 4 hours per break, configurable).

### 3.4 Exception Handling

- **FR-TA-019** — The system detects and labels the following exception types on ingestion/
  aggregation, each stored as a first-class flag on the relevant `TimeClockEvent`/`TimesheetLine`:
  `MISSED_PUNCH` (an expected clock-out or break-end never arrived, e.g. a shift's clock-in has no
  matching clock-out by end-of-day + grace window), `EARLY_CLOCK_IN` (before `shift.starts_at -
  grace_before_min`), `LATE_CLOCK_OUT` (after `shift.ends_at + grace_after_min`),
  `UNSCHEDULED_PUNCH` (a clock-in with no matching `ShiftAssignment` for that employee/location/
  window at all, distinct from early/late against a shift that does exist), `GEOFENCE_VIOLATION`
  (§3.1), `BREAK_COMPLIANCE` (§3.3).
- **FR-TA-020** — **Missed-punch correction workflow**: an employee (or, on their behalf, their
  manager) submits a `CorrectionRequest` proposing a specific corrected timestamp and a reason code
  (`FORGOT_TO_CLOCK_OUT`, `DEVICE_ISSUE`, `SYSTEM_ERROR`, `OTHER` + free text) for a missing or
  incorrect punch; the request is queued `PENDING_APPROVAL` against the Store/Site Manager for that
  location (default approver; escalation-eligible per §3.5's approver-hierarchy pattern), who can
  **Approve** (creates/corrects the underlying `TimeClockEvent` with `source: MANAGER_CORRECTION`
  and a full before/after audit trail), **Reject** (with a reason, returned to the employee), or
  **Modify-and-Approve** (approve a different timestamp than requested, also fully audited).
- **FR-TA-021** — A `TimeClockEvent` created via correction is flagged `is_corrected: true` and
  retains a link to its originating `CorrectionRequest`; the original (missing/incorrect) state is
  never silently overwritten — the audit trail preserves both the gap/error and the correction
  (source-spec P7 historical truth).
- **FR-TA-022** — `EARLY_CLOCK_IN`/`LATE_CLOCK_OUT`/`UNSCHEDULED_PUNCH`/`GEOFENCE_VIOLATION` flags do
  not, by default, block the punch or require a correction request — they are informational flags
  surfaced to the manager during timesheet review; a tenant policy can elevate any of them to
  `require_manager_approval: true`, which routes the specific `TimesheetLine` into a "needs review"
  state before the parent timesheet can be approved (§3.5).
- **FR-TA-023** — A pending `CorrectionRequest` prevents the affected day's `TimesheetLine` from
  being included in a timesheet approval submission until it resolves (approved/rejected); the
  weekly timesheet as a whole can still be approved for all non-blocked lines with the blocked
  line(s) carried forward, so one outstanding correction never holds an entire pay period's payroll
  export hostage for the rest of the team (firm platform default, not tenant-configurable — see §9).

### 3.5 Timesheet Generation & Approval

- **FR-TA-024** — A nightly background job (LLD §7) aggregates the prior day's `TimeClockEvent` rows
  per employee into a `TimesheetLine` (one row per employee per work-date): raw worked minutes,
  break-deducted minutes, regular minutes, overtime minutes (per §3.6), flags carried from §3.4, and
  `worked_location_id`. Lines roll up into a `Timesheet` per employee per pay period (weekly or
  bi-weekly, tenant-configured pay-period calendar).
- **FR-TA-025** — A `Timesheet` follows the lifecycle `DRAFT/OPEN → PENDING_APPROVAL → APPROVED →
  EXPORTED`, with a `CORRECTION_REQUESTED` side-branch back to `OPEN` for the affected line(s) (full
  state diagram in `architecture-diagram.md`). The timesheet becomes `PENDING_APPROVAL`
  automatically at pay-period close (configurable cutoff time) or manually when a manager opens it
  for early review.
- **FR-TA-026** — The Store/Site Manager reviews a `PENDING_APPROVAL` timesheet per employee: daily
  and weekly totals, all flags, planned-vs-actual comparison (from Workforce Scheduling), and any
  resolved corrections; approval requires explicit action per employee (bulk-approve-all is
  available for lines with zero unresolved flags, but a line carrying any exception flag configured
  `require_manager_approval: true` cannot be included in a bulk action and must be reviewed
  individually).
- **FR-TA-027** — Once `APPROVED`, a `Timesheet`'s `TimesheetLine` values are immutable
  (source-spec P7); any subsequent correction requires a formal **reopen** action (manager or
  Payroll/HR Admin role only, fully audited) that moves the timesheet back to `OPEN` for the
  affected line(s) and requires re-approval before it can be re-exported — an already-`EXPORTED`
  timesheet reopening additionally emits a `timesheet.correction_after_export` event so the
  Integration Hub/payroll system is notified a prior export is now stale.
- **FR-TA-028** — `APPROVED` timesheets are picked up by the payroll export batch job (LLD §7),
  transition to `EXPORTED`, and emit `timesheet.approved` and `timesheet.exported` events consumed
  by the Integration Hub's payroll connector (Module 12); this module has no knowledge of the
  destination payroll system's own data model beyond the exported field contract (LLD §2).

### 3.6 Overtime Calculation

- **FR-TA-029** — Overtime is computed from a per-tenant, versioned `OvertimeRule` configuration
  (not hardcoded to one jurisdiction), with the following configurable primitives: `daily_threshold_
  minutes` (e.g. 480 = 8h/day, null = not applied), `daily_multiplier` (e.g. 1.5), `weekly_threshold_
  minutes` (e.g. 2400 = 40h/week, null = not applied), `weekly_multiplier`, `double_time_daily_
  threshold_minutes` (optional, e.g. 720 = 12h/day) with its own multiplier, `consecutive_day_
  premium` (optional flag: the 7th consecutive worked day in a defined week gets a premium
  multiplier regardless of hours, a California-style rule expressed as configuration, not code),
  and `rounding_rule` (`NONE`, `NEAREST_5MIN`, `NEAREST_15MIN`, tenant-configurable grace/rounding
  window applied at punch time before aggregation, symmetric by default to avoid systematically
  favoring either party; platform default at provisioning is `NONE` — see §9 for rationale).
- **FR-TA-030** — Where both a daily and weekly threshold are configured, the daily calculation
  runs first per day, and weekly overtime is computed on the *remaining* regular hours across the
  week (no double-counting of the same minutes as both daily and weekly overtime) — see LLD §5
  pseudocode for the exact algorithm and worked example.
- **FR-TA-031** — `OvertimeRule` is versioned like other tenant policy configuration; a
  `TimesheetLine` snapshots the `overtime_rule_version_id` used at aggregation time so a later rule
  change never silently recomputes historical pay periods (source-spec P7, P10).
- **FR-TA-032** — `OvertimeRule` scope is overridable per `OrganizationNode` subtree, not
  tenant-global-only, shipped in MVP (state/country-level labor law differences within one enterprise
  tenant are supported from day one): an `OvertimeRule` may be scoped to any `OrganizationNode` with
  the most-specific applicable rule winning (same effective-dated resolution pattern as `UserScope`),
  since a single multi-state retail tenant is a first-order expected customer shape, not an edge case
  (see §9 for the full rationale).

### 3.7 Unscheduled / Off-Schedule Clock-In

- **FR-TA-033** — Where a tenant/role permits punches with no matching `ShiftAssignment`
  (`allow_unscheduled_punch: true`, default `true` for hourly frontline roles so a legitimately
  called-in-early or extra-shift worker is never blocked from clocking in), the punch is accepted
  and flagged `UNSCHEDULED_PUNCH` (§3.4) for manager review rather than rejected; where the policy
  is `false`, the app blocks the clock-in with an explanation and a "Request manager override" path.

## 4. UX States

| State | Trigger | Behavior |
|---|---|---|
| **Outside-geofence-blocked** | Device GPS outside all configured `GeofenceZone` radii for the target location, `geofence_mode: STRICT` | Full-screen non-blocking-of-app (but blocking-of-punch) state: "You're outside the clock-in area for [Location]. Move closer or contact your manager." Shows distance-to-nearest-zone-edge when available; offers "Request manager override" which raises a `CorrectionRequest`-style pre-punch exception, not a fabricated punch. |
| **Break-reminder-nudge** | Shift duration crosses a configured mandatory-break trigger threshold with no break started | Local (offline-capable) notification + persistent in-app banner: "Time for your break — tap to start." Dismissible without penalty; re-fires once more before shift-end grace window if still not acted on. |
| **Timesheet-pending-approval** | Timesheet auto-transitions at pay-period close, or manager opens early | Manager sees a per-employee review list with flag badges (missed punch, geofence, unscheduled, break compliance); Associate sees a read-only "Awaiting manager approval" banner on their own timesheet view. |
| **Correction-request-pending** | Employee or manager submits a `CorrectionRequest` | Employee sees "Correction submitted — awaiting manager review" on the affected day; Manager sees it in their Exceptions queue (Module 04 Manager tab, Module 02 approval surfaces) with before/after values and reason code. |
| **Clock-in success (confirmation)** | Successful punch, online or queued offline | Brief confirmation ("Clocked in at 9:02 AM — [Location]") with the effective (server-reconciled, once synced) time; while offline, shows "Clocked in (will confirm once synced)" using the local device time as a provisional display value only. |
| **Duplicate clock-in blocked** | Attempted clock-in while an open clock-in already exists for the employee | Non-blocking modal shows the existing open punch's start time/location and offers "Clock Out Now" instead of silently rejecting or creating a duplicate. |
| **Anti-buddy-punch capture (photo)** | `PHOTO` mode configured | Camera-first capture screen (reuses Module 04 capture pattern) presented immediately after the punch is registered locally, not gating the punch itself — a punch is never blocked/lost because the photo step was skipped or failed; a skipped/failed photo is flagged `punch_photo_missing: true` for manager review. |
| **Break-taken confirmation (blocking)** | Clock-out attempted with a mandatory break required but not recorded | Blocking modal per FR-TA-017: "Confirm no meal break was taken" / "Add a break now" before clock-out completes. |
| **Offline clock event (queued)** | No connectivity at punch time | Identical capture UX to online (per Module 04 offline-is-normal principle); shows the standard offline banner/queued-count pattern from Module 04 §4, not a Time & Attendance-specific offline state. |
| **Cross-location punch flagged** | Punch accepted under FR-TA-011 policy path | Confirmation shows the worked location distinctly ("Clocked in at Downtown Store — your home location is Uptown Store") so the associate is never confused about which site they're logged against. |

## 5. Validation & Edge Cases

| Edge case | Required behavior |
|---|---|
| Offline clock-in, later synced | Client records `client_captured_at`; server assigns `server_received_at` on sync and this is the authoritative timestamp for all duration/overtime math, per source-spec §19.4 ("client clock cannot be trusted"). The associate-facing confirmation shown offline is provisional and, if the reconciled server time differs materially (beyond a configurable tolerance, default 2 minutes, accounting for normal sync latency) from the device time, the discrepancy is flagged `clock_skew_flagged: true` for manager visibility rather than silently accepted or silently corrected. |
| Clock-in without a matching scheduled shift | Accepted or blocked per FR-TA-033 tenant policy; when accepted, flagged `UNSCHEDULED_PUNCH` and excluded from planned-vs-actual variance metrics (there is no plan to compare against) but still included in worked-hours/payroll aggregation. |
| Duplicate clock-in prevention | Server enforces at most one open (`clock_out_at IS NULL`) `TimeClockEvent` per `(tenant_id, employee_id)`; a duplicate clock-in request (including a retried offline sync of the same local event, de-duplicated by `client_uuid` per the standard sync idempotency pattern) never creates a second open punch — see LLD §6 concurrency. |
| Employee clocks in, device goes offline before clock-out | The open clock-in syncs (or was already synced) independently of the clock-out; if the device never reconnects before an operational cutoff (default: end of the scheduled shift + 12h grace, tenant-configurable), the shift is surfaced to the manager as `MISSED_PUNCH` (missing clock-out) for correction rather than left open indefinitely blocking the next clock-in — see LLD §7 nightly job. |
| Cross-midnight shift | A shift/clock-in that spans midnight is attributed to a single logical work-date (the date the shift *started*, tenant-configurable to "date shift ends" for split cases) for daily-total and daily-overtime-threshold purposes; the underlying `TimeClockEvent` timestamps are unaffected (real UTC/local timestamps), only the aggregation-bucket assignment follows this rule. Weekly totals are unaffected by the bucketing choice since they sum the same underlying minutes regardless of which day they're attributed to. |
| Two devices/sessions punch for the same employee (shared device edge case) | The server enforces the single-open-punch invariant per employee regardless of originating device; a second device attempting to clock in the same already-open employee is treated identically to FR-TA-006 (duplicate-clock-in blocked), not as a new device's independent state. |
| Manager approves a timesheet line, then a late correction request arrives for the same line | Blocked: a correction request against an already-`APPROVED` (or `EXPORTED`) line requires the reopen workflow (FR-TA-027), never a silent edit; the correction UI clearly states the line must be reopened, with the reopen actor and reason captured in the audit trail. |
| Geofence zone deleted/deactivated while an employee is clocked in under it | The active clock-in is unaffected (it already snapshotted `geofence_zone_id`, FR-TA-009); the employee's next clock-out at that location falls back to the tenant's `geofence_mode` default handling as if no zone existed (fails open to `ADVISORY` unless the location has a replacement zone configured), and this is flagged for admin visibility so a deleted zone doesn't silently create clock-out friction. |
| Employee terminated/deactivated mid pay-period with an open clock-in | Deactivation (Identity/Org module) does not retroactively delete attendance history (source-spec P7); an open clock-in at deactivation time is auto-closed at the deactivation timestamp with a system-generated flag `auto_closed_on_deactivation: true`, routed to the manager for confirmation as part of that employee's final timesheet. |
| Break started but shift's clock-out never happens (nested missed punch) | Both the break-end and the shift clock-out are independently detected as `MISSED_PUNCH` candidates by the nightly job; the correction UI presents them as one combined correction request for that day rather than two separate ones, to avoid a manager approving a corrected clock-out while a phantom open break remains. |

## 6. Non-Functional Requirements

| Requirement | Target | Measurement |
|---|---|---|
| Clock-event local capture latency (tap to locally-persisted, pre-sync) | p95 ≤ 500ms, indistinguishable online vs. offline, consistent with Module 04's atomic-write pattern | Device telemetry |
| Clock-event capture success rate (never silently lost) | 100% of confirmed local captures eventually sync — release-gating acceptance test, same bar as Module 04's offline-submission test | QA acceptance suite (airplane-mode clock in/out, kill/restart, reconnect → exactly one server `TimeClockEvent`) |
| Sync reliability for clock events | Same SLA as platform-wide sync (source-spec §23): ≥95% of standard reconnect syncs < 30s; ≥99% eventual sync success within 24h | Sync telemetry |
| Geofence evaluation latency | p95 ≤ 300ms from GPS fix availability to allow/block decision (client-side, no network round-trip required for the geofence check itself — zones are cached) | Device telemetry |
| Timesheet nightly aggregation completeness | 100% of prior-day clock events for active employees aggregated before the configured manager-review-availability time each morning | Job telemetry / SLA alert |
| Timesheet approval-to-export latency | p95 ≤ 5 min from manager approval action to `timesheet.approved` event availability to Integration Hub | Event lag telemetry |
| Shift-change peak burst handling | Sustain a burst of clock events at shift-change times (e.g. a 500-location retail tenant with synchronized 9:00am shift starts) without degrading p95 API latency beyond the platform-wide 300ms target (source-spec §23) | Load test |

## 7. Dependencies

| Dependency | Direction | Nature |
|---|---|---|
| Workforce Scheduling (Module 14) | Inbound (consumes) | Reads `Shift`/`ShiftAssignment` to determine the eligible clock-in window, detect `EARLY_CLOCK_IN`/`LATE_CLOCK_OUT`/`UNSCHEDULED_PUNCH`, and produce planned-vs-actual variance on the timesheet review screen. Time & Attendance never writes to Workforce Scheduling's entities. |
| Mobile & Offline (Module 04) | Inbound (consumes) | Clock-in/out and break punches are captured using the same local queue, `client_uuid` idempotency, offline-first UX patterns, and sync contract as any other offline submission/evidence capture; this module supplies the domain payload shape only. |
| Identity & Organization (Module 01) | Inbound (consumes) | Employee record (`User`), `home_location_id`, role, and `UserScope` for authorization of who can clock in where and who can approve whose timesheet. |
| Evidence (Module 05) | Outbound (produces) | Anti-buddy-punching photo capture (§3.1 `PHOTO` mode) is stored as a standard `Evidence` object linked to the `TimeClockEvent`. |
| Integrations & Extensibility (Module 12) | Outbound (produces) | `timesheet.approved`/`timesheet.exported` events and the exportable timesheet payload are consumed by a payroll outbound connector (Workday/ADP/SAP-class); this module owns no payroll-system-specific mapping logic itself. |
| Work Orchestration (Module 02) | Referenced (pattern reuse) | `CorrectionRequest` approval reuses the platform's general approval-routing UX conventions (approve/reject/modify) without being a `WorkItem` subtype. |
| Analytics & Control Tower (Module 11) | Outbound (produces) | Exception rates, on-time-punch rates, and approval SLA metrics feed the manager/regional dashboards. |

## 8. Traceability

| Requirement range | Related Epic |
|---|---|
| FR-TA-001–013 (clock-in/out, geofence, multi-location) | Extension of EPIC-05 (Mobile Offline) capture pattern + new attendance capability |
| FR-TA-014–018 (break policy) | New capability, policy-configuration pattern shared with EPIC-04 (Form Platform) template-versioning style |
| FR-TA-019–023 (exceptions/corrections) | Pattern-aligned with EPIC-07 (Issues & CAPA) exception lifecycle, implemented as its own lightweight lifecycle |
| FR-TA-024–028 (timesheet generation/approval) | New capability; approval UX aligned with EPIC-03 (Work Core) approval routing conventions |
| FR-TA-029–032 (overtime) | New capability |
| FR-TA-033 (unscheduled punch) | Depends on EPIC-01 (Tenant & Org Graph)/Workforce Scheduling for role/shift context |

## 9. Resolved Decisions

- **Biometric clock-in (`BIOMETRIC` mode, FR-TA-005) is optional per tenant, device-dependent, and
  never mandatory.** It is implemented as an on-device OS-level biometric gate (confirms "this is the
  enrolled device owner unlocking the app to punch," using Face ID/Android BiometricPrompt) rather
  than a server-side biometric face-match against a stored template. Rationale: (a) device
  fragmentation — not every frontline device has biometric hardware, and BYOD fleets vary widely;
  (b) storing and matching biometric templates server-side introduces significant regulatory exposure
  (BIPA-class state biometric privacy laws) disproportionate to the anti-buddy-punching value versus
  the simpler `PHOTO` mode; (c) it always degrades gracefully to PIN/passcode (consistent with Module
  04 FR-MOB-041's biometric-app-unlock pattern) so a device without biometric hardware, or an employee
  who declines enrollment, is never blocked from clocking in.
- **Pending correction requests block only the affected `TimesheetLine`/day, not the entire employee
  timesheet or the entire location's payroll batch (FR-TA-023).** This is the platform default and is
  not tenant-configurable to an all-or-nothing gate: a single slow-moving exception must never delay
  the rest of a team's pay, which matters more than the marginal auditability benefit of an
  all-or-nothing per-employee gate — auditability is already fully preserved at the line level via
  `blocked_by_correction_request_id` and the timesheet-level `PENDING_APPROVAL` state, which makes any
  outstanding exception visible to Payroll/HR without needing to withhold the whole timesheet.
- **`OvertimeRule` and `BreakPolicy` scope to `OrganizationNode` subtree ships in MVP, not Phase 2.**
  The effective-dated, node-scoped resolution model (§3.6 FR-TA-032, already reflected in the LLD §1.2
  `organization_node_id` nullable column and most-specific-wins resolution) is built from day one.
  Rationale: retrofitting scoped policy resolution after a single-tenant-global launch is a materially
  larger migration (requiring backfill of every historical aggregation's rule attribution) than
  building it in from the start, and multi-state/multi-country retail and QSR tenants are a primary
  ICP for this platform, not an edge case to defer.
- **First-party jurisdiction-specific compliance rule packs (e.g. California meal/rest break premium
  pay, EU working-time-directive rest requirements) are deferred to Phase 2.** MVP ships only the
  generic, fully configurable `BreakPolicy`/`OvertimeRule` primitives, configured by the tenant's
  Payroll/HR Admin or systems integrator to match their jurisdiction. Rationale: a first-party rule
  pack is a packaging/content investment, not a platform capability gap — the generic primitives
  already support every jurisdiction's rules structurally (FR-TA-014/029) — so no tenant is blocked at
  launch, and packaging is revisited once pilot-tenant jurisdictions are known and can inform which
  packs are worth pre-building.
- **`CorrectionRequest` default approver is the Store/Site Manager of the `worked_location_id`** (not
  `home_location_id`) for a cross-location punch, since that manager has direct visibility into the
  actual shift worked. Escalation on SLA breach follows the same ladder pattern as Issues & CAPA
  (Module 06), with the SLA fixed at 24 hours from request creation (`sla_due_at = created_at + 24h`,
  matching the correction-request SLA escalation job in LLD §8) — short enough that a payroll-blocking
  exception is resolved within the same pay cycle in the overwhelming majority of cases, long enough
  to cover a manager's typical day-off/rest-day gap.
- **Default overtime rounding rule (`OvertimeRule.rounding_rule`, FR-TA-029) is `NONE` (exact-time
  crediting, no rounding) at tenant provisioning time.** Rounding is available (`NEAREST_5MIN`,
  `NEAREST_15MIN`) and commonly enabled by tenants for payroll-processing simplicity, but defaulting
  to no rounding is the safer platform posture: rounding rules carry jurisdiction-specific wage-and-
  hour legal exposure (several US states restrict or prohibit rounding that systematically favors the
  employer, even when nominally symmetric), so the platform does not enable rounding by default —
  a tenant's Payroll/HR Admin opts in explicitly, with the symmetric-by-default behavior (already
  specified in FR-TA-029) applying once they do.
