# Field Service & Third-Party Vendor Scheduling — Functional Specification

Module corresponds to a new epic (**EPIC-25**, not in the original Appendix E — added per
`new-modules-addendum.md` Module 25). Source: `new-modules-addendum.md` Module Map row 25 (new
entities `ServiceRequest`, `VendorTechnician`, `ServiceAppointment`, `VendorSLA`; the platform-wide
`WorkItem.assignee_type = VENDOR` extension this module introduces), `source-spec.md` §17.1
(`WorkItem` universal work abstraction), §7 (Work Orchestration scheduling/SLA machinery this module
reuses), §21 (tenant isolation — extended here with an external-actor isolation dimension). Read
alongside `docs/modules/16-asset-maintenance-cmms/functional-spec.md` §8 (FR-CMMS-052/053, the
vendor-dispatch handoff this module receives) and `docs/modules/02-work-orchestration/low-level-design.md`
§1.1 (`work_items.assignee_type` column, extended here from `USER, ROLE, LOCATION, QUEUE` to add
`VENDOR`).

## 1. Purpose and Scope

### 1.1 Purpose

Enterprise frontline operations routinely need work performed by people who are not employees of the
tenant: HVAC contractors, pest control services, elevator inspectors, appliance repair technicians,
janitorial contractors, and vendors dispatched by Asset Maintenance (Module 16) when no internal
technician is available or qualified. This module is the system of record for **who the tenant's
external vendors and their individual technicians are, where they can work, what service they
provide, how an appointment gets scheduled and confirmed with them, and how well they perform against
contracted SLAs** — while the actual unit of work (accept, travel, arrive, complete) rides on the
same `WorkItem` execution engine every other module uses, via a new `VENDOR` assignee type this
module introduces as an extension of `work_items.assignee_type` (not a parallel field, not a parallel
state machine).

This module closes a Tier-3 competitive gap against Operandio/MaintainX-class field-service and
vendor-management features: without it, external dispatch is an unmanaged email/phone exercise with
no SLA visibility, no scorecarding, and — critically — no safe way to give an external technician
system access without over-exposing tenant data.

### 1.2 In Scope

- **Vendor/technician directory**: `Vendor` (company-level record: name, service categories, contact
  info, contract/SLA-tier reference, active status) and `VendorTechnician` (individual technician
  employed by or contracted through a `Vendor`: name, contact info, certifications, the individual who
  actually shows up on-site).
- **Service categories & certifications**: a tenant-extensible taxonomy (`HVAC`, `PEST_CONTROL`,
  `ELEVATOR`, `PLUMBING`, `ELECTRICAL`, `APPLIANCE_REPAIR`, `JANITORIAL`, `GENERAL_CONTRACTOR`, etc.)
  attached to a `Vendor` (categories it offers) and optionally to a `VendorTechnician` (categories
  that specific person is certified/qualified for, a subset of the vendor's).
- **Coverage area**: the `OrganizationNode` subtree(s) a `Vendor` (and optionally a specific
  `VendorTechnician`) is contracted/willing to service, expressed the same way every other module
  expresses location scope.
- **Service request creation**: (a) derived from a `MaintenanceWorkOrder` (Module 16) that needs
  external dispatch — Module 16 hands off by creating a `ServiceRequest` referencing the originating
  `WorkItem`/`Asset`; (b) standalone, for work with no underlying `MaintenanceWorkOrder` at all (e.g.
  a scheduled pest-control visit, a periodic elevator inspection, an HVAC contractor's seasonal
  service call) — the `ServiceRequest` in this case is the origination point and this module creates
  the `WorkItem` itself.
- **Vendor assignment**: manual selection by an internal user, or auto-match by service category +
  coverage area + SLA-tier preference (§5).
- **External-facing appointment scheduling**: supports both directions — the vendor proposes one or
  more candidate time windows and the internal requester approves one, or the internal requester
  proposes a window and the vendor confirms/counter-proposes.
- **Limited-access vendor portal/account model**: a `VENDOR`-scoped identity distinct from a normal
  tenant employee, restricted to the appointments explicitly assigned to that technician (§7).
- **Appointment completion & sign-off**: an on-site contact (Store/Site Manager or delegate) confirms
  work was actually performed, captures evidence, and the appointment closes out.
- **Vendor SLA tracking and scorecarding**: response time, on-time arrival rate, first-time-fix rate,
  rolled up per vendor per period.
- **Fallback coordination for vendors without platform access**: email/SMS-based scheduling for
  vendors who will never get a portal account.

### 1.3 Out of Scope (owned by other modules, referenced here)

- **`WorkItem`/`ServiceAppointment` execution and state machine.** The underlying `WorkItem` (type
  `SERVICE_APPOINTMENT`) transitions through the canonical state machine (`DRAFT → SCHEDULED →
  AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW → APPROVED → VERIFIED`, with `BLOCKED`/
  `OVERDUE`/`ESCALATED`/`REJECTED` branches) exactly as defined in `conventions.md` and owned by
  **Work Orchestration (EPIC-03)**. This module never writes `work_items.status` directly; it calls
  Work Orchestration's public/internal APIs to create the `WorkItem` and to read its state, and it
  layers its own `ServiceRequest` lifecycle (§8) *around* that WorkItem for the vendor-facing
  concerns Work Orchestration has no model for (proposed windows, vendor confirmation, sign-off
  contact) — see §9 for the explicit non-duplication rule, mirroring the pattern Module 16 already
  established for `MaintenanceWorkOrder`.
- **Asset and parts data.** `Asset`, `AssetType`, `PartInventoryItem`, `PartUsage` are owned by
  **Asset & Equipment Maintenance (Module 16)**. This module receives an `asset_id` reference on a
  `ServiceRequest` created from a `MaintenanceWorkOrder` but never stores or mutates asset attributes,
  meter readings, or parts inventory. A vendor technician who uses a part during a visit records that
  usage against the originating `MaintenanceWorkOrder` in Module 16, not in this module.
- **Escalation ladders/SLA timers as a generic mechanism.** The Temporal-based escalation engine and
  `escalation_policies` table are owned by Work Orchestration; this module configures and reuses that
  mechanism (§7 FR-FSVC-070) rather than building a second SLA-timer engine for vendor response-time
  breaches.
- **General Issue/CAPA lifecycle.** A vendor performance failure that rises to the level of a managed
  exception (e.g., a vendor repeatedly missing SLA) is raised as an `Issue` via Issues & CAPA
  (Module 06); this module does not run its own root-cause workflow.
- **Restricted-scope identity/session mechanics.** The underlying principal type, session handling,
  RBAC role definition, and `UserScope` row semantics are owned by **Identity/Org**. This module
  defines *what* scope a vendor technician's account should be provisioned with and consumes
  Identity/Org's provisioning API; it does not implement its own authentication or session store.
- **General-purpose webhook/API integration infrastructure.** The vendor-facing webhook subscription
  mechanism and API key management reuse the **Integrations (EPIC-13)** Integration Hub's connector
  pattern; this module defines the vendor-specific event payloads and endpoints but does not build a
  second webhook delivery/retry system.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Store/Site Manager | Requests external service (standalone or as fallback from a Module 16 breakdown flow), reviews vendor-proposed appointment windows and approves one, acts as the on-site contact who confirms completion and captures sign-off evidence, escalates a vendor no-show. |
| Vendor Technician (external, limited-access user) | Logs into a scoped portal/mobile experience (or receives email/SMS if no account), sees only appointments explicitly assigned to them, accepts/proposes/confirms a time window, marks en route/arrived, records completion notes and photos, cannot browse any other tenant data. |
| Operations Admin | Manages the vendor/technician directory, defines service categories and coverage areas, configures vendor contracts and SLA tiers, sets up auto-match policy, provisions/deprovisions vendor technician accounts. |
| Regional Manager | Reviews vendor performance scorecards across a territory, compares vendors within a category, decides which vendors to renew/replace based on SLA compliance trends. |

## 3. Vendor & Technician Directory

- **FR-FSVC-001 — `Vendor` record.** Company-level: `name`, `legal_entity_name` (optional),
  `primary_contact` (name/phone/email), `service_category_codes[]`, `sla_tier` (`STANDARD`,
  `PREFERRED`, `PREMIUM` — tenant-extensible), `contract_reference` (free text/document link,
  contract *content* itself may live in a document store outside this module's scope), `status`
  (`ACTIVE`, `SUSPENDED`, `INACTIVE`), `has_platform_access` (boolean — drives whether this vendor's
  technicians get portal accounts or fall back to email/SMS coordination, FR-FSVC-045),
  `preferred` flag (surfaced first in manual-selection UI and weighted in auto-match, §5).
- **FR-FSVC-002 — `VendorTechnician` record.** Individual person: `vendor_id` FK, `name`, `phone`,
  `email`, `certification_tags[]` (subset of, or matching, the vendor's service categories — e.g. a
  vendor certified for both `HVAC` and `PLUMBING` may have one technician certified only for `HVAC`),
  `status` (`ACTIVE`, `INACTIVE`), and — when `Vendor.has_platform_access` is true — a
  `restricted_user_id` FK to an Identity/Org `User` record provisioned with the restricted external
  scope (§7). A `VendorTechnician` with `restricted_user_id = NULL` is a directory-only record used
  purely for assignment/reporting; all coordination with them happens over email/SMS (FR-FSVC-045).
- **FR-FSVC-003 — Service category taxonomy.** Tenant-scoped, extensible list of category codes
  (seeded with the common set in §1.2). A `Vendor` may hold multiple categories; a `ServiceRequest`
  specifies exactly one required category (the primary trade needed) plus optional secondary tags.
- **FR-FSVC-004 — Coverage area.** `VendorCoverageArea` rows: `vendor_id`, `organization_node_id`
  (subtree root), optional `vendor_technician_id` (when a specific technician's coverage is narrower
  than the vendor's overall contracted territory — e.g., a national HVAC vendor contracts
  enterprise-wide but only one technician actually covers the Pacific Northwest region). Coverage is
  evaluated as subtree membership against the requesting location, consistent with `OrganizationNode`
  scoping used everywhere else on the platform.
- **FR-FSVC-005 — Vendor directory search/list.** Filterable by `service_category`, `coverage`
  (location or subtree), `sla_tier`, `status`, `preferred`. Cursor pagination per platform convention.
- **FR-FSVC-006 — Vendor deactivation.** Setting `Vendor.status = INACTIVE` (or `SUSPENDED`) excludes
  it from auto-match candidate lists immediately and blocks new manual assignment (existing in-flight
  `ServiceAppointment`s are unaffected — they complete under the assignment already made); all of that
  vendor's `VendorTechnician.restricted_user_id` accounts are deprovisioned (session revoked, scope
  removed) via the same Identity/Org deactivation path used for any user (source-spec §24.1 "User
  deactivation" acceptance test applies identically to a vendor-scoped principal).

## 4. Service Request Creation

- **FR-FSVC-010 — From a `MaintenanceWorkOrder`.** When Module 16 determines a `MaintenanceWorkOrder`
  needs external dispatch (no eligible internal technician, FR-CMMS-052, or an Operations Admin
  explicitly chooses vendor dispatch), it calls this module's `create_service_request` API with
  `source_work_item_id`, `asset_id`, `service_category`, `location_id`, `urgency`, and any
  `breakdown_description`/evidence references. This module creates a `ServiceRequest` in state
  `REQUESTED` (§8) and does **not** create a second `WorkItem` — instead, once a vendor/appointment
  is confirmed, it transitions the *existing* `MaintenanceWorkOrder`'s assignment
  (`assignee_type = VENDOR`) via Work Orchestration's reassign API, per the addendum's rule that
  vendor dispatch of a `MaintenanceWorkOrder` extends that same WorkItem rather than spawning a
  parallel one.
- **FR-FSVC-011 — Standalone service request.** A Store/Site Manager (or any user with
  `field_service:create_request` scope) creates a `ServiceRequest` directly — no originating
  `MaintenanceWorkOrder` or `Asset` required — for work like pest control, janitorial contract visits,
  or elevator inspections. Required fields: `location_id`, `service_category`, `description`,
  `requested_window` (earliest/latest acceptable date range), `urgency` (`ROUTINE`, `URGENT`,
  `EMERGENCY`). In this path, this module itself creates the underlying `WorkItem`
  (`type_code = SERVICE_APPOINTMENT`) once a vendor is assigned and a window is confirmed (§8) —
  never before, since a `WorkItem` should not exist for work with no assignee or schedule yet.
- **FR-FSVC-012 — Recurring standalone requests.** A standalone `ServiceRequest` may reference a
  recurrence pattern (e.g., "pest control, every 60 days") using the same RFC5545-subset recurrence
  shape Work Orchestration's `schedules` table uses; this module runs its own Temporal recurrence
  workflow (mirroring the pattern Module 16 established for `MaintenanceSchedule`, per
  `high-level-design.md` §5) that periodically creates a new `ServiceRequest` occurrence, each of
  which flows through the normal vendor-match → appointment → completion cycle independently.

## 5. Vendor Assignment

- **FR-FSVC-020 — Manual selection.** An internal user with scope at the `ServiceRequest`'s location
  picks a `Vendor` (and optionally a specific `VendorTechnician`) from a filtered candidate list
  (filtered by service category and coverage, same predicate as auto-match, just not auto-applied).
- **FR-FSVC-021 — Auto-match.** When enabled (tenant or per-category policy), the system computes a
  ranked candidate list: (1) hard filter — `service_category` match AND coverage-area match for the
  request's `location_id` AND `Vendor.status = ACTIVE`; (2) rank by `sla_tier` (configurable
  preference order, default `PREMIUM` > `PREFERRED` > `STANDARD`), then by the vendor's rolling
  `VendorScorecard.on_time_arrival_rate` (descending), then by `preferred` flag, then by current open
  appointment load (ascending, a simple load-balancing tiebreaker). The top-ranked vendor is either
  auto-assigned (if policy = `AUTO_ASSIGN`) or presented as a recommendation for one-click confirm
  (if policy = `RECOMMEND`, the safer default). See LLD §4 for the concrete pseudocode.
- **FR-FSVC-022 — No eligible vendor found.** If auto-match returns zero candidates, the
  `ServiceRequest` moves to `NO_VENDOR_AVAILABLE` (§8) and surfaces in the requester's and Operations
  Admin's exceptions view; the Admin can widen the coverage-area/category filter manually or
  onboard/mark a new vendor as covering that area.
- **FR-FSVC-023 — Vendor company vs. individual technician assignment.** A `ServiceRequest`/
  `ServiceAppointment` is always assigned at the `Vendor` level (`work_items.assignee_id` references
  the `Vendor`); a specific `VendorTechnician` is attached once known (at proposal/confirmation time,
  or immediately if the vendor's dispatcher specifies one up front). This two-level model
  accommodates vendors who assign the specific person only close to the appointment date.
- **FR-FSVC-024 — Mid-appointment reassignment.** Either the vendor company reassigns which of its
  technicians will attend (common — the originally named tech calls in sick) or, less commonly, the
  internal requester reassigns to a different vendor entirely (the original vendor cancels/can't
  perform). Both are supported: a same-vendor technician swap is a lightweight update to
  `ServiceAppointment.assigned_technician_id` with no change to the underlying `WorkItem` assignment
  or state; a vendor-to-vendor reassignment is a full re-match/re-propose cycle through the
  `ServiceRequest` state machine (§8), because the SLA clock, coverage validation, and technician
  portal scope all need to be re-established for the new vendor.

## 6. External-Facing Appointment Scheduling

- **FR-FSVC-030 — Vendor-proposes flow.** After assignment, the vendor (via portal, or a coordinator
  relaying by phone/email for non-portal vendors) proposes one or more candidate time windows. The
  `ServiceRequest` enters `APPOINTMENT_PROPOSED`; the internal requester approves one window (which
  confirms the appointment) or rejects with a reason (which returns the request to await a new
  proposal, without re-running vendor match — same vendor, new window negotiation).
- **FR-FSVC-031 — Internal-proposes flow.** The internal requester (or the system, from
  `requested_window` on a standalone request) proposes a specific time; the vendor confirms or
  counter-proposes an alternative window through the same portal/email channel. Both flows converge
  on the same `CONFIRMED` state once a window both sides accept exists — the UI and data model treat
  "who proposed first" as provenance metadata, not a different downstream flow.
- **FR-FSVC-032 — Confirmation creates/updates the WorkItem.** The moment a window is `CONFIRMED`,
  this module (a) for a standalone request, creates the `WorkItem` (`type_code = SERVICE_APPOINTMENT`,
  `assignee_type = VENDOR`, `assignee_id = vendor_id`, `due_at`/`scheduled_at` = the confirmed window
  start, `location_id`) via Work Orchestration's `create_work_item` API; (b) for a
  `MaintenanceWorkOrder`-originated request, reassigns the existing `WorkItem` to
  `assignee_type = VENDOR` via the reassign API and sets its schedule metadata. Either way, this
  module never writes `work_items.status`/`assignee_*` columns directly.
- **FR-FSVC-033 — Reschedule.** Either party can propose a new window on an already-`CONFIRMED`
  appointment before work starts; this returns the `ServiceRequest` to `APPOINTMENT_PROPOSED` and
  updates the linked `WorkItem`'s `due_at` once re-confirmed, emitting a reschedule event distinct
  from a cancellation so scorecard metrics (§9) can distinguish "rescheduled once, arrived on time
  for the new window" from a true miss.
- **FR-FSVC-034 — Vendor en route / arrival.** The vendor technician (via portal/mobile) marks
  "en route" (optionally sharing an ETA) and "arrived on-site," each timestamped; these timestamps
  are the raw inputs to the on-time-arrival SLA metric (§9) and are visible to the on-site contact in
  real time (FR-FSVC-050 UX state).

## 7. Limited-Access Vendor Portal / Account Model

- **FR-FSVC-040 — `VENDOR` as a restricted principal type.** A `VendorTechnician` with platform access
  is provisioned as an Identity/Org `User` whose `UserScope` is **not** a normal role-at-location
  grant — it is a new restricted scope kind, `ASSIGNED_WORK_ITEM_ONLY`, that Identity/Org's
  authorization model must support as an extension (see `high-level-design.md` §7 for the precise
  enforcement mechanism). This is narrower than the platform's narrowest existing internal role: even
  a location-scoped internal `UserScope` grants visibility into everything at that location (all
  WorkItems, forms, other users); a vendor technician's scope grants visibility into **only the
  specific `WorkItem` row(s) currently or historically assigned to them**, nothing else at that
  location, and no `OrganizationNode` subtree access at all.
- **FR-FSVC-041 — Provisioning.** Creating a `VendorTechnician` with `Vendor.has_platform_access =
  true` triggers this module to call Identity/Org's user-provisioning API requesting the restricted
  scope kind, receiving back a `restricted_user_id`. The vendor technician receives an invitation
  (email/SMS) with a portal link and, on first login, sets up credentials (password or, where the
  tenant's vendor population supports it, a passwordless magic-link/OTP flow). Magic-link/OTP is the
  firm v1 authentication mechanism for portal-enabled vendors, given the low per-vendor account volume
  relative to the engineering cost of federation; full SSO/SAML federation with a vendor's own
  workforce IdP is deferred as a fast-follow (see §15 Resolved Decisions) rather than built now, since
  vendor technicians are not tenant employees and typically do not exist in the tenant's IdP at all.
- **FR-FSVC-042 — What the vendor portal shows.** A simplified, mobile-first view scoped entirely to
  the technician's own upcoming and past appointments: for each, the service address (street-level
  location detail, not the tenant's broader location hierarchy), the category/description, the
  proposed/confirmed window, the on-site contact's name and phone, and a completion-recording flow
  (notes, photos as `Evidence`, signature capture). No search, no directory browse, no visibility into
  any other WorkItem, user, asset, or tenant data of any kind.
- **FR-FSVC-043 — Scope enforcement is server-side, not UI-only.** Every API call the vendor portal
  makes is authorized by re-checking, on the server, that the requested `WorkItem`/`ServiceAppointment`
  id is currently or historically in the calling principal's assigned set — the portal UI simply never
  rendering a link to other data is treated as a UX nicety, never as the security boundary (§10 NFR).
- **FR-FSVC-044 — Account lifecycle tied to assignment activity, not indefinite.** A vendor
  technician's account remains provisioned as long as their `VendorTechnician` record is `ACTIVE`;
  deactivating the `VendorTechnician` (technician leaves the vendor company, or the vendor relationship
  ends) immediately deprovisions the account (session revocation + scope removal) via Identity/Org's
  standard deactivation path — historical `ServiceAppointment` records and evidence remain
  attributable to them (platform principle P7), but they can no longer authenticate.
- **FR-FSVC-045 — No-platform-access fallback.** For a `Vendor` with `has_platform_access = false` (a
  small pest-control contractor with no interest in a portal, for example), all coordination —
  proposal, confirmation, reschedule, completion confirmation — happens via outbound email/SMS
  generated by this module (using the Notification Hub) containing a **one-time, scoped, expiring
  link** to a minimal unauthenticated (token-secured) web view for that single appointment only (view
  details, confirm a window, mark complete with a photo upload). The token is single-appointment and
  time-bound (expires a configurable number of days after the appointment's scheduled date); it is
  never a general-purpose account and grants no ability to see any other appointment. This is the
  mechanism, not a person manually re-typing status into the system.

## 8. `ServiceRequest` Lifecycle

`ServiceRequest` is this module's own lifecycle, layered around (not replacing) the underlying
`WorkItem` state machine — see `architecture-diagram.md` for the full `stateDiagram-v2`.

- **FR-FSVC-050 — States.** `REQUESTED → VENDOR_MATCHED → APPOINTMENT_PROPOSED → CONFIRMED →
  IN_PROGRESS → COMPLETED → SIGNED_OFF`, with a `NO_VENDOR_AVAILABLE` branch from `REQUESTED` (no
  match found, §5) and an `ESCALATED` branch reachable from any pre-`SIGNED_OFF` state on SLA breach
  (§9) or explicit manual escalation (vendor no-show, vendor declines after matching).
- **FR-FSVC-051 — Relationship to `WorkItem` status.** `IN_PROGRESS`/`COMPLETED` on the
  `ServiceRequest` are driven by (not independently set from) the underlying `WorkItem`'s
  `IN_PROGRESS`/`SUBMITTED` transitions — this module's event consumer listens for `task.status_changed`
  events on the linked `WorkItem` and advances `ServiceRequest.status` accordingly, never the reverse.
  `SIGNED_OFF` is the one state this module drives independently, since sign-off is a
  vendor-scheduling-specific concept (an on-site contact's attestation) layered on top of the
  WorkItem's own `VERIFIED` status, which is set through the normal Work Orchestration review flow in
  parallel.

## 9. Appointment Completion & Sign-Off

- **FR-FSVC-060 — On-site confirmation required.** For any `ServiceAppointment`, a designated on-site
  contact (defaults to the requesting Store/Site Manager, reassignable per request) must confirm
  completion before the `ServiceRequest` reaches `SIGNED_OFF` — the vendor technician marking their
  own work "done" (which drives the underlying `WorkItem` to `SUBMITTED`) is necessary but not
  sufficient; this mirrors Work Orchestration's general `SUBMITTED → PENDING_REVIEW → APPROVED`
  pattern but is expressed here as a lightweight sign-off rather than a full multi-step approval
  route, since a single on-site attestation is the norm for vendor visits (a tenant needing a
  multi-level approval on vendor work can configure a Work Orchestration `approval_route_id` on the
  `SERVICE_APPOINTMENT` `WorkItem` type exactly as Module 16 does for cost-threshold work orders — no
  new mechanism required).
- **FR-FSVC-061 — Sign-off evidence.** The on-site contact captures a completion photo (of the
  finished work/area), an optional signature, and a rating of the visit (1–5, feeds
  `first_time_fix_rate`, §9); these are stored as `Evidence` records referenced from the sign-off
  event, not duplicated into this module's own tables.
- **FR-FSVC-062 — First-time-fix determination.** The sign-off flow asks a required yes/no: "Was the
  issue fully resolved on this visit?" A "no" answer keeps the originating problem open (for a
  `MaintenanceWorkOrder`-originated request, the `MaintenanceWorkOrder` itself is not auto-reopened —
  Module 16 owns that decision — but this module records `first_time_fix = false` against the vendor
  for scorecarding and, for a standalone request, offers one-click creation of a follow-up
  `ServiceRequest` referencing this one as `follow_up_of_request_id`).

## 10. Vendor SLA Tracking and Scorecarding

- **FR-FSVC-070 — `VendorSLA` (contracted targets).** Per `Vendor` (or per `Vendor` + `service_category`
  when a vendor's contract differentiates by trade): `response_time_target_minutes` (time from
  `ServiceRequest` assignment to vendor's first proposed window or acceptance), `on_time_arrival_
  window_minutes` (grace window around the confirmed appointment start counted as "on time"),
  `target_first_time_fix_rate`, `escalation_policy_id` (FK into Work Orchestration's
  `escalation_policies`, per the platform's "reuse the standard escalation mechanism, don't reinvent
  it" rule — a vendor missing its response-time target triggers the exact same Temporal-driven
  escalation ladder every other `WorkItem` uses, targeting this vendor's assigned `WorkItem`, with
  ladder steps configured to notify the internal requester, then an Operations Admin, then optionally
  auto-create an `Issue` via Issues & CAPA for a persistently underperforming vendor).
- **FR-FSVC-071 — Response-time SLA breach.** Measured from `ServiceRequest` entering
  `VENDOR_MATCHED` to entering `APPOINTMENT_PROPOSED` (vendor's first response). If the
  `VendorSLA.response_time_target_minutes` elapses first, the linked `WorkItem`'s
  `escalation_policy_id` (set from `VendorSLA.escalation_policy_id` at `WorkItem` creation) drives the
  standard `AVAILABLE → OVERDUE → ESCALATED` path — see LLD §3(c) sequence diagram. This module does
  not run a second timer; it configures the existing one.
- **FR-FSVC-072 — On-time-arrival metric.** Computed per completed appointment: `arrived_at` timestamp
  (FR-FSVC-034) vs. `confirmed_window_start + on_time_arrival_window_minutes`; a vendor with no
  `arrived_at` recorded (portal-less vendor with no arrival ping) is scored from the on-site contact's
  sign-off attestation ("did the technician arrive within the window?" — a required sign-off question
  when `arrived_at` telemetry is unavailable).
- **FR-FSVC-073 — `VendorScorecard` rollup.** Computed per `Vendor` per rolling period (default
  trailing 90 days, configurable) via a background job (LLD §7): `response_time_p50`/`p90`,
  `on_time_arrival_rate`, `first_time_fix_rate`, `total_appointments`, `escalation_count`,
  `average_completion_rating` (from FR-FSVC-061). Exposed via a dashboard (§11) and an API for
  Analytics/Control Tower consumption.
- **FR-FSVC-074 — Scorecard drives auto-match ranking.** As specified in FR-FSVC-021, the scorecard's
  `on_time_arrival_rate` directly feeds the ranking function — a vendor's own historical performance
  measurably affects future auto-match preference, closing the loop without any AI/agent involvement
  (this is a deterministic ranking rule, not an agent recommendation).

## 11. UX States

- **`service-request-pending-vendor-response`** — the request is `VENDOR_MATCHED` and awaiting the
  vendor's first proposed window; shown to the internal requester with an elapsed-time indicator
  against the SLA response target and a "nudge vendor" manual action.
- **`appointment-window-proposed-awaiting-confirmation`** — a window has been proposed (by either
  side) and awaits the other party's acceptance; shows the proposed window(s), who proposed, and
  accept/reject/counter-propose actions appropriate to the viewer (internal requester sees
  approve/reject; vendor portal or fallback link sees accept/counter-propose).
- **`vendor-en-route`** — appointment is `CONFIRMED`/day-of; shows the vendor's en-route status and
  ETA (when reported) to the on-site contact, with a "vendor hasn't updated status" indicator if the
  confirmed start time has passed with no en-route/arrival ping (feeds the no-show escalation path).
- **`on-site-confirmation-required`** — the vendor has marked work `SUBMITTED` but sign-off is
  pending; surfaces as an action item for the designated on-site contact, distinct from a generic
  pending-review item, with the completion photo/notes the vendor submitted shown alongside the
  sign-off form.
- **`vendor-scorecard` dashboard** — Regional Manager/Operations Admin view: sortable/filterable table
  of vendors (by category, coverage, sla_tier) with the FR-FSVC-073 metrics, trend sparklines period
  over period, and drill-down into individual appointment history per vendor.
- **Empty/loading/offline/permission-denied.** Empty vendor directory shows a "no vendors registered
  for this category/area" zero-state with a CTA (Operations Admin) to add one. The internal-facing
  screens (request creation, approval, sign-off) follow the platform's standard offline-capable
  mobile capture pattern (source-spec §19) for the on-site contact's sign-off flow specifically
  (photo/rating/signature queue and sync like any other evidence capture). The vendor portal itself is
  **not** an offline-first surface in v1 (external vendor devices are not part of the platform's
  managed offline sync fleet) — it requires connectivity, with a clear "you're offline, try again"
  state rather than a silent failure. A user without `field_service:read` scope at a location receives
  `403 SCOPE_DENIED`, not `404`, consistent with the platform's scope-leak-avoidance convention; a
  vendor technician's own scope violation (attempting to access an appointment not theirs) additionally
  returns `403 SCOPE_DENIED` and is logged as a security-relevant event (§10 NFR), not merely a normal
  authorization miss.

## 12. Validation and Edge Cases

- **12.1 — Vendor technician account access must be scoped only to explicitly assigned appointments.**
  This is treated as a hard security requirement (§13 NFR), not a best-effort UX convention. A vendor
  technician account must never be able to enumerate, search, or retrieve any `WorkItem`, `Asset`,
  `OrganizationNode`, or `User` record beyond the specific `ServiceAppointment`(s) currently or
  historically assigned to them — including via ID-guessing (IDOR), broad list endpoints, or any
  data returned incidentally (e.g., an appointment response must never include another appointment's
  detail even for the same location). See `high-level-design.md` §7 and `low-level-design.md` §3(b)
  for the precise enforcement mechanism and its verification sequence.
- **12.2 — SLA response-window miss triggers standard escalation, not a bespoke one.** A vendor
  missing `VendorSLA.response_time_target_minutes` follows the exact `escalation_policies` mechanism
  from Work Orchestration (FR-FSVC-071) — this module must never implement a second, parallel
  notification/escalation timer for this case, even though the *trigger condition* (vendor
  non-response) is specific to this module.
- **12.3 — Vendor company vs. individual technician reassignment mid-appointment.** See FR-FSVC-024.
  A same-vendor technician swap must not reset the `ServiceRequest`'s SLA clock or state (the vendor
  company already met its response/confirmation SLA; only the *person* changed) — the swap is logged
  as an audit event (`technician_reassigned`) with the outgoing/incoming `VendorTechnician` ids and
  reason, and the outgoing technician's portal access to that specific appointment is immediately
  revoked (their scope narrows the instant they're no longer assigned) while the incoming technician's
  scope is granted for that appointment id.
- **12.4 — Vendor has no platform access at all.** See FR-FSVC-045. The fallback flow must produce
  the same audit trail (who proposed what window, when confirmed, when completed) as the portal flow
  — email/SMS coordination is not permitted to be a lower-fidelity, unaudited shadow process; every
  inbound reply/click against the scoped token link is captured as a structured event, not free-text
  parsed loosely.
- **12.5 — Confirmed appointment window conflicts with vendor's other commitments.** This module does
  not attempt to model or validate a vendor's own capacity/calendar beyond the coverage-area and
  category checks — a vendor company is responsible for not over-committing itself; the platform's
  role is to make proposal/confirmation fast and auditable, not to run the vendor's dispatch
  operation for them. (A vendor with their own scheduling system may instead integrate via the
  Integrations webhook/API path, §14.)
- **12.6 — Reassignment after `CONFIRMED` but before `IN_PROGRESS` (vendor cancels late).** Returns
  the `ServiceRequest` to `VENDOR_MATCHED` (re-running auto-match, excluding the cancelling vendor for
  this occurrence) rather than `REQUESTED`, preserving the original request's urgency/category/
  description; the cancellation is recorded against the original vendor's scorecard as a negative
  signal (counts against `on_time_arrival_rate`'s denominator as a miss) distinct from a
  requester-initiated reschedule (FR-FSVC-033), which does not penalize the vendor.
- **12.7 — Duplicate service requests for the same asset/location.** Mirrors Module 16's
  FR-CMMS-022 pattern: a new standalone `ServiceRequest` against the same `location_id` +
  `service_category` within a short lookback window (default 4 hours) surfaces a non-blocking
  "possible duplicate" prompt rather than silently merging.

## 13. Non-Functional Requirements

- **NFR-FSVC-001 — Vendor-portal access-scoping correctness is a hard security requirement.** Unlike
  most authorization edge cases on the platform (which are correctness requirements handled with the
  same rigor as any other bug class), a scope leak in the vendor portal exposes internal tenant
  operational data to a non-employee external party — this is treated with the severity of a tenant
  isolation breach (source-spec §21.1), not an ordinary access-control bug. It requires: (a) every
  vendor-portal-facing endpoint enforced by a dedicated authorization check distinct from and
  layered on top of the normal `UserScope` evaluation path (§ HLD §7), never relying solely on the
  portal UI not rendering a link; (b) mandatory automated regression tests for horizontal
  privilege escalation, IDOR, and cross-vendor access specific to the `ASSIGNED_WORK_ITEM_ONLY` scope
  kind, run on every release touching this module or Identity/Org's scope evaluator (extending
  source-spec §6.1's existing regression-test requirement); (c) security review sign-off required
  before this module's portal endpoints ship to any tenant, and before any change to the
  `ASSIGNED_WORK_ITEM_ONLY` enforcement path ships at all, regardless of which module initiated the
  change.
- **NFR-FSVC-002 — API performance.** Directory and request/appointment endpoints meet the platform
  p95 < 300 ms target; the vendor-matching computation (FR-FSVC-021) is allowed a relaxed p95 < 500 ms
  given it involves a ranked multi-predicate query.
- **NFR-FSVC-003 — Fallback-channel reliability.** Email/SMS-based coordination (FR-FSVC-045) for
  non-portal vendors uses the platform's standard Notification Hub delivery guarantees (retries,
  dedupe) — a vendor coordination step must not silently fail to send.
- **NFR-FSVC-004 — Auditability.** Every `ServiceRequest` state transition, vendor assignment/
  reassignment, proposed/confirmed window, and sign-off is recorded in an append-only history
  sufficient to reconstruct the full negotiation and completion trail without relying on mutable
  current-state rows (platform Principle P7), which matters disproportionately here given the
  external-party, potentially-contractual nature of the record (SLA disputes, vendor contract
  renewal decisions).
- **NFR-FSVC-005 — Scale.** Designed for the platform's stated scale target; vendor/technician volume
  is small relative to `WorkItem`/`Asset` volume (thousands of vendors, tens of thousands of
  technicians per large tenant, at most), but `ServiceAppointment` volume rides on the platform's
  general `WorkItem` scale characteristics.

## 14. Dependencies

| Module | Dependency |
|---|---|
| Work Orchestration (EPIC-03) | Owns the `SERVICE_APPOINTMENT` `WorkItem` type's execution state machine, the new `VENDOR` `assignee_type` value this module introduces as an extension, and the escalation-policy mechanism this module reuses for SLA response-time breaches. This module creates/reads/reassigns `WorkItem` rows via its public/internal API and consumes `task.status_changed`/`task.overdue`/`task.escalated` events. |
| Asset & Equipment Maintenance (Module 16) | Hands off `MaintenanceWorkOrder`s needing external dispatch by calling this module's `create_service_request` API; this module never owns `Asset`/parts data and reads only the `asset_id` reference needed for context. |
| Identity/Org (EPIC-01/02) | Provisions the restricted `ASSIGNED_WORK_ITEM_ONLY`-scoped external-vendor principal type this module requires; owns the underlying authentication/session mechanics and the scope-evaluation engine this module's enforcement depends on. |
| Issues & CAPA (Module 06) | A persistently underperforming vendor (repeated SLA escalations) can be raised as a managed `Issue` via `create_issue`; this module does not run its own root-cause/CAPA workflow. |
| Evidence (EPIC-06) | Sign-off photos, signatures, and vendor-submitted completion photos are `Evidence` records referenced by ID. |
| Integrations (EPIC-13) | Provides the vendor-facing API/webhook option for vendors that run their own dispatch/scheduling systems and prefer machine-to-machine integration over the portal or email/SMS fallback; this module defines the vendor-specific payload/event contract, the Integration Hub owns delivery/retry/signing. |
| Notification Hub (part of source-spec §16.3 Core Technical Components) | Delivers vendor-facing email/SMS for the no-platform-access fallback flow (FR-FSVC-045) and portal-user push/email notifications for proposal/confirmation/reminder events. |
| Analytics & Control Tower (EPIC-12/15) | Consumes `VendorScorecard` and `ServiceRequest` lifecycle events/CDC feed for cross-tenant and cross-vendor performance reporting. |

## 15. Resolved Decisions

- **Vendor technicians are provisioned as genuine Identity/Org `User` rows carrying a new restricted
  `UserScope` kind (`ASSIGNED_WORK_ITEM_ONLY`), not a separate lightweight external-actor identity
  model.** A parallel identity system was considered and rejected: it would double the surface area
  that must be proven secure (source-spec §21.1's "prove object-level scope at every access boundary"
  would need two boundary implementations instead of one), double the session/audit/deactivation
  machinery, and cut against the addendum's guiding principle that a new module should feel like a
  natural extension of the platform, not a bolted-on product. The narrower semantics are achieved
  entirely through the new `ASSIGNED_WORK_ITEM_ONLY` scope kind's evaluation logic (HLD §7), not
  through a different identity substrate. This does require Identity/Org to accept a scope kind whose
  evaluation predicate is "row-level, dynamic, tied to a mutable assignment set" rather than the
  existing "subtree membership" predicate — new work for Identity/Org, not zero-cost, but bounded and
  reusable by any future module needing the same external-actor pattern (e.g., a future customer- or
  supplier-facing portal).
- **Vendor technicians never receive tenant SSO/SAML credentials in v1.** They are not employees and
  typically are not present in the tenant's IdP; magic-link/OTP is the firm v1 authentication
  mechanism for portal-enabled vendors (FR-FSVC-041). Full external-IdP federation with a vendor's own
  company IdP is deferred as a fast-follow, built only if a large tenant's vendor population
  demonstrates the need — building it speculatively now would add federation-protocol surface area for
  a population (external, low-volume, non-employee) where magic-link/OTP is already sufficient and
  simpler to secure.
- **`VendorSLA` supports per-service-category variance only in v1; per-location/per-region contract
  variance for a single vendor is not modeled.** This matches how most real vendor contracts are
  structured (by trade, not by sub-region) and keeps the SLA/escalation configuration surface small.
  A per-region override is added only if a pilot tenant's actual contract structure requires it, at
  which point it extends the existing `vendor_slas` table (LLD §1.1) with an additional scope column
  rather than a redesign.
- **The no-platform-access fallback (FR-FSVC-045) requires an Operations Admin to explicitly flip
  `Vendor.has_platform_access`; there is no vendor self-service upgrade path in v1.** Account
  provisioning for a new restricted principal type carries real security-review cost
  (NFR-FSVC-001); putting that decision behind an explicit admin action keeps it under deliberate
  tenant control rather than an automatic growth loop (e.g., "vendor clicked the link N times, offer
  them a portal account") that could provision accounts without the tenant's informed choice.
