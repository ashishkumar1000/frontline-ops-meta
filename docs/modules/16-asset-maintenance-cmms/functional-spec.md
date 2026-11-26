# Asset & Equipment Maintenance (CMMS) — Functional Specification

Module corresponds to **EPIC-16 (Asset & Equipment Maintenance)** (new epic; not in the original
Appendix E — added per `new-modules-addendum.md` Module 16). Source: `new-modules-addendum.md`
Module Map row 16, `source-spec.md` §17.1 (`Asset` listed as a canonical entity, currently a
placeholder), §7.1 (`Asset Inspection` already an existing `WorkItem` type — distinct from this
module's `MAINTENANCE_WORK_ORDER`), §5 (`IoT/Assets` Phase-2/3 capability row), Appendix A (Golden
Scenario references a "Maintenance action created" step this module now formalizes). Stays fully
consistent with `conventions.md` (canonical `WorkItem` state machine reused verbatim, Temporal for
scheduling, Postgres/Kafka/Redis stack, `/api/v1/` conventions) and with `new-modules-addendum.md`
(this module promotes `Asset` from placeholder to a full first-class entity).

## 1. Purpose and Scope

### 1.1 Purpose

This module closes the platform's largest Tier-1 competitive gap against MaintainX-class CMMS
products: a proper asset registry, preventive maintenance (PM) program authoring, parts inventory,
and technician/vendor assignment for equipment and machinery at every location. It is the system of
record for **what equipment exists, where, in what condition, and what maintenance program governs
it**, and it is the origination point for `MaintenanceWorkOrder` — a new `WorkItem` type (per
`new-modules-addendum.md`: `MAINTENANCE_WORK_ORDER`) that reuses Work Orchestration's canonical
state machine (`DRAFT → SCHEDULED → AVAILABLE → IN_PROGRESS → SUBMITTED → PENDING_REVIEW → APPROVED
→ VERIFIED`, with `BLOCKED`/`OVERDUE`/`ESCALATED`/`REJECTED` branches) verbatim. This module
operationalizes the "Maintenance action created → Follow-up inspection scheduled" step of the
Golden End-to-End Scenario (source-spec Appendix A) as a first-class, schedulable capability rather
than an ad hoc manual task, and gives Regional/Executive Control Tower views a real "preventive
intervention rate" signal (source-spec §5 PRD Capability Map row "IoT/Assets").

### 1.2 In Scope

- **Asset registry**: `Asset` promoted from a placeholder entity (source-spec §17.1) to a full
  entity — asset type taxonomy, per-asset attributes, install date, warranty, location assignment,
  parent-child asset relationships (e.g., a compressor unit that is a child component of a walk-in
  cooler asset), asset operational status lifecycle (`IN_SERVICE`/`OUT_OF_SERVICE`/
  `UNDER_MAINTENANCE`/`RETIRED`).
- **PM schedule authoring**: calendar-based recurrence (reusing Work Orchestration's `schedules`
  recurrence patterns) and meter-based recurrence (e.g., every 500 operating hours, every 10,000
  cycles) — a scheduling mode Work Orchestration's existing `schedule_mode` enum does not cover and
  which this module introduces and owns.
- **Automatic `MaintenanceWorkOrder` creation** from due PM schedules (via Work Orchestration's
  `create_work_item` API) and from ad-hoc breakdown reports (Store/Site Manager-initiated).
- **Parts inventory**: per-location stock levels, reorder points, parts usage recorded against a
  work order, low-stock alerting, and the hook into Inventory/Procurement (Module 19) for purchase
  requisition when stock falls below reorder point.
- **Technician assignment**: internal technician pool (skills/certifications), skill/certification
  matching against a `MaintenanceWorkOrder`'s required skill tags.
- **Downtime tracking**: asset `OUT_OF_SERVICE` state transitions and duration, feeding downtime
  KPIs.
- **Meter reading capture**, modeled as `Evidence` (per `new-modules-addendum.md`: "Evidence for
  before/after photos, meter readings") — not a parallel telemetry store.
- Asset detail view, maintenance history timeline, low-stock-parts alert, overdue-PM badge,
  parts-needed blocking state UX.
- `MaintenanceWorkOrder` failure-triggered `Issue` creation for recurring failures (via Issues &
  CAPA integration).

### 1.3 Out of Scope (owned by other modules, referenced here)

- **Work-order execution and state**: the actual `IN_PROGRESS → SUBMITTED → PENDING_REVIEW →
  APPROVED → VERIFIED` execution flow, assignment/reassignment mechanics, escalation ladder
  timers, and multi-level approval routing (used here for cost-threshold work orders, e.g. a repair
  quote above a tenant-configured dollar amount requiring manager approval before parts are
  ordered/labor is dispatched) are **owned by Work Orchestration (EPIC-03)**. This module is a
  producer and a consumer of `WorkItem` rows of type `MAINTENANCE_WORK_ORDER` — it never writes
  `work_items.status` directly and never invents a second state machine for work-order execution.
  See §9 for the explicit non-duplication rule.
- **Evidence capture, storage, hashing, retention** — Evidence module (EPIC-06). Meter readings and
  before/after photos are `Evidence` records referenced by ID; this module never stores media or
  raw sensor values as first-class rows of its own (a meter reading's *value* is stored on this
  module's `meter_readings` table for scheduling-calculation purposes, but the *photo/proof* of that
  reading, when captured, is an `Evidence` record).
- **Issue lifecycle, root cause analysis, CAPA steps** — Issues & CAPA (EPIC-07/Module 06). A
  recurring asset failure creates an `Issue` via `create_issue`; this module does not run its own
  root-cause or corrective-action workflow.
- **ERP parts/vendor master-data sync** — Integrations & Extensibility (EPIC-13). This module owns
  the local `part_inventory_items`/`vendors` tables and their operational fields (stock level,
  reorder point) but treats ERP as the source of truth for catalog/pricing/vendor master data where
  a tenant has that integration enabled, consuming it via the Integration Hub's connector pattern.
- **External vendor dispatch, vendor-facing portal, vendor SLA tracking** — Field Service & Third-
  Party Vendor Scheduling (Module 25). This module supports assigning a `MaintenanceWorkOrder` to
  either an internal `Technician` or, via the `assignee_type=VENDOR` extension
  (`new-modules-addendum.md`), a vendor — but the vendor-facing appointment scheduling, vendor SLA
  enforcement, and vendor portal UX belong to Module 25.
- **Purchase requisition and purchase-order lifecycle** — Inventory, Procurement & Supplier
  Management (Module 19). This module raises a low-stock signal and can auto-create a purchase
  requisition *record* via that module's API, but does not own PO approval routing or supplier
  scorecards.
- **Sensor telemetry ingestion at scale** (predictive maintenance from continuous sensor streams) —
  IoT Sensor Platform (Module 17). A `Sensor` bound to an `Asset` can feed this module's meter-based
  PM calculation via a lightweight "latest meter value" read, but this module does not ingest or
  store high-volume time-series readings itself.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Store/Site Manager | Reports a breakdown (ad-hoc `MaintenanceWorkOrder`), views asset detail/maintenance history for equipment at their location, sees overdue-PM and low-stock-parts badges, approves cost-threshold work orders at their scope. |
| Maintenance Technician (internal) | Views assigned `MaintenanceWorkOrder`s (executed via the standard `WorkItem` mobile flow), records parts usage, captures meter readings and before/after photos, marks work blocked when parts are unavailable. |
| Operations Admin | Configures asset type taxonomy, authors PM schedules (calendar and meter-based), defines parts catalog/reorder points per location, configures cost-approval thresholds, manages the technician pool and skill/certification taxonomy. |
| Regional Manager | Views asset health, downtime, and PM-compliance rollups across a territory in the Control Tower; drills into overdue-PM and repeat-failure patterns. |
| AI Agent (Resolution Agent, Compliance Monitor) | Detects a recurring-failure pattern on an `Asset` (via `get_recent_failures`-class tooling scoped to asset history) and proposes a `MaintenanceWorkOrder` or an `Issue`, per the same L1-default autonomy contract as elsewhere in the platform. |

## 3. Asset Registry

- **FR-CMMS-001 — Asset type taxonomy.** Operations Admins define `AssetType` records (e.g.
  "Walk-in Cooler," "HVAC Rooftop Unit," "Compressor," "POS Terminal," "Delivery Van") with a
  tenant-scoped code, display name, default maintenance interval hints (informational, not binding
  — actual schedules are authored per-asset or per-type in §4), a JSON-Schema-validated
  `custom_attributes_schema` (so different asset types can carry different structured fields, e.g.
  refrigerant type for a cooler vs. VIN for a vehicle), and a default `required_skill_tags[]` used
  for technician matching (§7).
- **FR-CMMS-002 — Asset record.** Required fields: `name`, `asset_type_id`, `location_id`
  (`OrganizationNode` ref), `serial_number` (optional but unique per tenant when present),
  `manufacturer`, `model`, `install_date`, `warranty_expires_at` (nullable), `purchase_cost`
  (nullable), `custom_attributes` (JSONB, validated against the asset type's schema),
  `operational_status` (§6), `criticality` (`LOW`/`MEDIUM`/`HIGH`/`CRITICAL` — drives escalation and
  Control Tower prioritization). An asset is always scoped to exactly one current `location_id`, with
  location moves tracked as history (§9.2).
- **FR-CMMS-003 — Parent-child asset relationships.** An `Asset` may reference a `parent_asset_id`
  (e.g., a compressor is a child component of a walk-in cooler). Children inherit the parent's
  `location_id` by construction (a child cannot be at a different location than its parent — enforced
  at write time) but have independent `operational_status`, PM schedules, and maintenance history. A
  parent asset's maintenance history timeline (§8) optionally rolls up child-asset events
  (UI toggle "include components"), since a technician replacing a compressor is operationally a
  cooler-maintenance event even though the work order targets the child asset.
- **FR-CMMS-004 — Multi-level hierarchy depth.** No hard-coded depth limit on `parent_asset_id`
  chains, consistent with the platform's arbitrary-depth-graph principle for `OrganizationNode`, but
  the UI collapses to a practical 3-level tree view (asset → major component → sub-component) by
  default with an "expand further" affordance for deeper chains — deep chains beyond 3 levels are
  expected to be rare and are not optimized for in the default UX.
- **FR-CMMS-005 — Asset decommission/retire.** Setting `operational_status = RETIRED` (§6) is a
  one-way terminal transition (retiring is not reversible via the standard UI; un-retiring requires
  an Operations Admin override with an audit reason, since a retired asset may already have been
  excluded from PM scheduling and Control Tower rollups). Retiring an asset with active PM schedules
  auto-pauses those schedules (`MaintenanceSchedule.status = PAUSED`, reason `ASSET_RETIRED`) rather
  than deleting them, so historical schedule configuration remains inspectable.
- **FR-CMMS-006 — Asset search and list.** Filterable by `asset_type`, `location`/location-subtree,
  `operational_status`, `criticality`, overdue-PM flag, and free-text (name/serial/model). Cursor
  pagination per platform convention.

## 4. Preventive Maintenance Schedule Authoring

- **FR-CMMS-010 — `MaintenanceSchedule` entity.** Each schedule targets either a single `Asset` or
  an `AssetType` + location-scope combination (e.g., "all HVAC Rooftop Units at Region West"), and
  has a `schedule_basis` of `CALENDAR` or `METER`.
- **FR-CMMS-011 — Calendar-based PM.** Reuses the recurrence expression shape from Work
  Orchestration's `schedules.rrule` (RFC5545 subset) — e.g., "every 90 days," "first Monday of every
  quarter." Calendar-based `MaintenanceSchedule` rows delegate materialization timing to the same
  Temporal recurrence pattern Work Orchestration uses (see HLD §5 technology justification); this
  module does not re-implement RRULE expansion.
- **FR-CMMS-012 — Meter-based PM.** Defined as `meter_type` (`OPERATING_HOURS`, `CYCLES`,
  `MILEAGE`, `UNITS_PRODUCED` — tenant-extensible enum) + `interval_value` (e.g., 500 hours) +
  `last_service_meter_value` (the meter reading at which the schedule was last satisfied). Due-date
  calculation requires an estimated future meter value, since the system cannot know exact future
  usage — see LLD §4.1 pseudocode for the due-date estimation algorithm, which uses a rolling
  average usage rate derived from historical `meter_readings`.
- **FR-CMMS-013 — Hybrid schedule (calendar OR meter, whichever comes first).** A schedule may
  specify both a calendar interval and a meter interval with `trigger_mode = EARLIEST` (default) or
  `LATEST`; the PM Scheduler evaluates both and materializes a `MaintenanceWorkOrder` per the
  configured combination rule. This matches common OEM maintenance contracts (e.g., "every 6 months
  or 5,000 miles, whichever comes first").
- **FR-CMMS-014 — Schedule template linkage.** A schedule may reference a `TemplateVersion`
  (Forms & Workflow Builder) defining the PM checklist a technician completes (e.g., a 12-point HVAC
  inspection form) — the same mechanism Work Orchestration uses to attach a form to any `WorkItem`.
  This module does not build a separate checklist authoring surface.
- **FR-CMMS-015 — Schedule lifecycle.** `ACTIVE`, `PAUSED`, `ARCHIVED` — mirrors Work Orchestration's
  `schedules.status` enum for consistency. Pausing a schedule stops future materialization but does
  not affect in-flight `MaintenanceWorkOrder`s.
- **FR-CMMS-016 — Cost/approval threshold.** A schedule (or an ad-hoc work order at creation) may
  carry an `estimated_cost`; when `estimated_cost` (or, at completion, `actual_cost` from summed
  `part_usages` + labor) exceeds a tenant-configured `cost_approval_threshold`, the
  `MaintenanceWorkOrder` is created with an `approval_route_id` set, reusing Work Orchestration's
  approval-routing infrastructure (§9) rather than a bespoke cost-approval mechanism.

## 5. Automatic Work Order Creation

- **FR-CMMS-020 — PM-driven creation.** The PM Scheduler (a Temporal workflow per `MaintenanceSchedule`,
  mirroring Work Orchestration's per-schedule recurrence workflow pattern) determines the next due
  occurrence and calls Work Orchestration's `POST /api/v1/work-items` (internal service-to-service,
  §HLD External Interfaces) with `type_code = MAINTENANCE_WORK_ORDER`, `origin = SCHEDULE`, the
  linked `asset_id`, `template_version_id` (if configured), and `due_at` computed per §4.
- **FR-CMMS-021 — Ad-hoc breakdown report creation.** A Store/Site Manager (or any user with
  `asset:report_breakdown` scope) creates a `MaintenanceWorkOrder` directly against an `Asset` with
  `origin = AD_HOC`, a required `breakdown_description`, optional severity/urgency flag, and optional
  initial `Evidence` (photo of the fault). Creating a breakdown report with `severity = CRITICAL` (or
  tenant policy marking the asset type as safety-critical) auto-sets the asset's
  `operational_status → OUT_OF_SERVICE` (§6) atomically with work-order creation.
- **FR-CMMS-022 — Duplicate breakdown suppression.** Before creating an ad-hoc work order, the system
  checks for an existing open `MaintenanceWorkOrder` against the same `asset_id` with
  `origin = AD_HOC` and status not in a terminal set (`VERIFIED, CANCELLED, REJECTED`) reported within
  a configurable lookback window (default 4 hours). A match surfaces as a non-blocking
  "possible duplicate" prompt ("A breakdown was already reported for this asset 20 minutes ago by J.
  Rao — view it instead?") rather than silently merging or silently creating a second work order; the
  reporting user may proceed anyway (e.g., a genuinely distinct new fault), which creates a second
  work order flagged `possible_duplicate_of`.
- **FR-CMMS-023 — Failure-triggered Issue creation.** When an `Asset` accumulates more than a
  tenant-configured number of `MaintenanceWorkOrder`s of `origin = AD_HOC` within a rolling window
  (default: 3 within 30 days), this module calls Issues & CAPA's `create_issue` with
  `category = EQUIPMENT`, `source = SYSTEM_PATTERN` (an addition to Issue's source set consistent
  with `AGENT`/`RULE_TRIGGERED` sources already defined by Module 06), and a reference to the
  contributing work orders, so a recurring-failure pattern becomes a managed exception with root
  cause analysis rather than an endless stream of independent repairs.
- **FR-CMMS-024 — Work order carries asset context, not a copy of it.** `work_items.asset_id`
  (already present in Work Orchestration's schema, source-spec §17.1) is the FK; this module never
  denormalizes asset attributes into `work_items.type_config` beyond a small display cache (asset
  name, location name) needed for offline-list rendering, refreshed on every asset update via an
  event-driven sync (§HLD External Interfaces).

## 6. Downtime Tracking (Asset Operational Status)

- **FR-CMMS-030 — Operational status enum.** `IN_SERVICE`, `OUT_OF_SERVICE`, `UNDER_MAINTENANCE`,
  `RETIRED`. See `architecture-diagram.md` for the full state diagram; this is a status owned and
  transitioned by this module, distinct from and never conflated with the `WorkItem` state machine of
  any `MaintenanceWorkOrder` acting on the asset.
  Transitions: `IN_SERVICE → OUT_OF_SERVICE` (breakdown reported or manually flagged),
  `OUT_OF_SERVICE → UNDER_MAINTENANCE` (a technician starts work — i.e., the linked
  `MaintenanceWorkOrder` transitions to `IN_PROGRESS`), `UNDER_MAINTENANCE → IN_SERVICE` (work
  verified complete), `IN_SERVICE → UNDER_MAINTENANCE` directly for scheduled PM that does not take
  the asset fully down first (e.g., routine filter change performed while the asset stays nominally
  available is instead modeled as `IN_SERVICE` throughout with no downtime — operational status only
  reflects genuine unavailability), and `* → RETIRED` (terminal, FR-CMMS-005).
- **FR-CMMS-031 — Downtime duration.** Every `OUT_OF_SERVICE`/`UNDER_MAINTENANCE` span is recorded as
  an `asset_status_events` row with `started_at`/`ended_at`; downtime duration = sum of spans where
  status ∈ `{OUT_OF_SERVICE, UNDER_MAINTENANCE}`. This feeds the "preventive intervention rate" and
  downtime KPIs surfaced in the Regional Manager Control Tower view.
- **FR-CMMS-032 — Auto-transition on work order completion.** When the driving `MaintenanceWorkOrder`
  reaches `VERIFIED`, this module's event consumer (on `task.completed`, per HLD §External
  Interfaces) transitions the asset back to `IN_SERVICE` automatically. A manual override is
  available (e.g., a manager marks the asset back in service before formal verification completes,
  or keeps it down after verification pending a second, unrelated fault) with an audit reason
  required.
- **FR-CMMS-033 — Multiple concurrent work orders on one asset.** An asset stays `OUT_OF_SERVICE`/
  `UNDER_MAINTENANCE` as long as at least one non-terminal `MaintenanceWorkOrder` targets it; the
  status only returns to `IN_SERVICE` when the last such work order reaches `VERIFIED` (or is
  cancelled), computed via a live count rather than a naive "any completion flips it back" rule
  (edge case §11.1).

## 7. Parts Inventory

- **FR-CMMS-040 — `PartInventoryItem`.** Per-location stock record: `part_number`, `description`,
  `unit_of_measure`, `quantity_on_hand`, `reorder_point`, `reorder_quantity`, `unit_cost`,
  `location_id`, `preferred_vendor_id` (nullable FK to `Vendor`). A part catalog entry (the
  part_number/description/unit_cost master data) may be tenant-wide with per-location stock rows, or
  ERP-synced (§1.3) — this module treats the ERP as authoritative for catalog fields when integrated,
  syncing read-only, and always authoritative for `quantity_on_hand` itself (operational state, not
  master data).
- **FR-CMMS-041 — Parts usage recording.** A technician executing a `MaintenanceWorkOrder` records
  `PartUsage` rows (`part_inventory_item_id`, `quantity_used`, `work_item_id`) against the work order;
  each recorded usage atomically decrements `quantity_on_hand` at that location (concurrency handling
  in LLD §6).
  Recording usage in excess of `quantity_on_hand` is a hard block by default (`INSUFFICIENT_STOCK`
  error), with a tenant-configurable override permission (`inventory:override_stockout`) for cases
  where physical stock is known to exceed the recorded count (reconciliation lag).
- **FR-CMMS-042 — Low-stock alert.** When `quantity_on_hand ≤ reorder_point` after a decrement, this
  module raises a low-stock alert (surfaced as a badge on the parts inventory view and, per tenant
  configuration, a notification to the Operations Admin/location manager) and — when the tenant has
  Inventory/Procurement (Module 19) enabled — automatically creates a purchase requisition record via
  that module's API (LLD §3(c)).
- **FR-CMMS-043 — Parts-needed blocking state.** A `MaintenanceWorkOrder` whose technician has flagged
  "parts needed" (a structured sub-state carried in `work_items.type_config`, not a new `WorkItem`
  top-level status — see §9) surfaces as a distinct, visually blocked card in both the technician's
  work list and the manager's exceptions view, referencing the specific `PartInventoryItem`(s) awaited
  and their current stock/on-order status. The underlying `WorkItem.status` remains `BLOCKED` (an
  existing canonical status), consistent with reusing the state machine rather than inventing a
  parallel one.

## 8. Technician Assignment

- **FR-CMMS-050 — `Technician` record.** An internal technician is a `User` with a `Technician`
  extension row: `skill_tags[]`, `certifications[]` (each with `certification_type`, `issued_at`,
  `expires_at`), `home_location_id`, `service_area_node_ids[]` (locations/subtrees they can be
  dispatched to), `availability_status` (`AVAILABLE`, `ON_JOB`, `OFF_SHIFT`).
- **FR-CMMS-051 — Skill/certification matching.** At `MaintenanceWorkOrder` creation (from either a
  PM schedule's `AssetType.required_skill_tags` or an ad-hoc report's asset type), the system computes
  an eligible-technician candidate list filtered by matching `skill_tags` and non-expired required
  `certifications`, then intersected with scope eligibility (`service_area_node_ids` covering the
  asset's location) — reusing the same `UserScope`/eligibility-check pattern Work Orchestration uses
  for role-based assignment, extended with the skill/certification predicate this module introduces.
  Assignment itself (writing `work_items.assignee_id`) is performed via Work Orchestration's existing
  reassign/assign API — this module only supplies the filtered candidate list and, optionally, an
  auto-assign-to-best-match policy.
- **FR-CMMS-052 — Expired certification handling.** A technician with an expired required
  certification is excluded from the eligible list (not merely deprioritized); if zero eligible
  internal technicians remain, the UI offers vendor dispatch (`assignee_type = VENDOR`, Module 25
  handoff) as the fallback path.
- **FR-CMMS-053 — Vendor assignment.** Assigning a `MaintenanceWorkOrder` to a `Vendor` sets
  `work_items.assignee_type = VENDOR` (the platform-wide extension introduced by Module 25) and
  `assignee_id` to a `VendorTechnician`/`Vendor` reference; this module records the `Vendor` entity
  (`name`, `service_categories[]`, `contact_info`, `preferred` flag) but the actual vendor-facing
  appointment/dispatch workflow is owned by Module 25 once assignment occurs.

## 9. Non-Duplication of the WorkItem State Machine

`MaintenanceWorkOrder` is a `WorkItem` type (`type_code = MAINTENANCE_WORK_ORDER`), not a parallel
entity with its own status column. This module:

- Never writes `work_items.status` outside calling Work Orchestration's status-transition API
  (`PATCH /api/v1/work-items/{id}/status`) — including for the "parts needed" sub-state (§7), which
  is expressed as `BLOCKED` status + a `type_config.block_reason = PARTS_NEEDED` payload field, not a
  new top-level status value.
- Reuses Work Orchestration's existing Approval Routing for cost-threshold work orders (see
  FR-CMMS-016), not a second approval mechanism.
- Never re-implements escalation/SLA timers for `MaintenanceWorkOrder` — an overdue PM work order
  follows the exact same `AVAILABLE/IN_PROGRESS → OVERDUE → ESCALATED` path and
  `escalation_policies` mechanism as any other `WorkItem` type.
- Owns only the module-specific supporting entities that feed into or read from that `WorkItem`:
  `Asset`, `AssetType`, `MaintenanceSchedule`, `PartInventoryItem`, `PartUsage`, `Vendor`,
  `Technician`.

## 10. UX States

- **FR-CMMS-060 — Asset detail view.** Header (name, type, location, criticality badge,
  operational-status badge), attributes panel (custom_attributes per type schema, warranty
  countdown), **maintenance history timeline** — a reverse-chronological feed merging: completed
  `MaintenanceWorkOrder`s (with outcome, technician, cost, linked evidence thumbnails), asset
  status-change events (downtime spans), meter-reading captures, and location-move events (§11.2).
  Optional "include child asset events" toggle (FR-CMMS-003).
- **FR-CMMS-061 — Overdue-PM badge.** An asset with at least one `MaintenanceSchedule`-derived
  `MaintenanceWorkOrder` past `due_at` without reaching `VERIFIED` shows a red "PM overdue" badge on
  its list row and detail header; the badge shows the count when more than one schedule is overdue.
- **FR-CMMS-062 — Low-stock-parts alert.** A location's parts inventory list shows an amber badge per
  item at/below `reorder_point`, and a summary "3 parts low" chip on the location's overview card in
  the manager workspace.
- **FR-CMMS-063 — Work-order-in-progress-with-parts-needed blocking state.** As described in
  FR-CMMS-043: a distinct card treatment (amber/blocked, wrench-with-parts icon) distinguishing this
  from a generically `BLOCKED` work order (e.g., blocked on a dependency) so managers can immediately
  tell "blocked because we're waiting on a part" from other block reasons.
- **FR-CMMS-064 — Empty/loading/offline/permission-denied states.** Empty asset registry shows a
  distinct "no assets registered at this location" zero-state with a CTA to add one (Operations
  Admin) vs. "no assets match filter." Technician mobile flows (parts usage recording, meter reading
  capture) queue offline via the standard sync protocol (source-spec §19), idempotent on
  `client_uuid`, consistent with every other offline-capable capture flow on the platform. A user
  without `asset:read` scope at a location receives `403 SCOPE_DENIED`, not a 404, per the platform's
  scope-leak-avoidance convention (see Module 06 FR-CAPA-085 for the identical rule applied here).

## 11. Validation and Edge Cases

- **11.1 — Meter-based PM schedule with a meter-reading gap or estimation need.** See LLD §4.1. When
  no fresh meter reading exists within a configurable staleness window (default 7 days) ahead of a
  projected due date, the scheduler falls back to the rolling-average usage-rate estimate rather than
  blocking materialization; the resulting `MaintenanceWorkOrder`'s `due_at` is flagged
  `estimated = true` in `type_config`, shown to the technician/manager as "estimated due date — no
  recent meter reading," and is recomputed (not silently left stale) the moment a fresh reading
  arrives, per LLD pseudocode.
- **11.2 — Asset moved between locations mid-maintenance-cycle.** Changing `Asset.location_id` while
  an active (non-terminal) `MaintenanceWorkOrder` exists against it does not retroactively change
  that work order's `location_id` (the work order stays attributed to the location it was created at,
  per the platform's historical-attribution principle, source-spec P7/§24.1 "Reorganization"
  acceptance test) — it does, however, immediately affect eligible-technician computation (§8) for
  any *new* work order created after the move, and the asset detail view's maintenance-history
  timeline records the move as a first-class event so a manager reviewing history is never confused
  about why an old work order references a different location than the asset's current one. A
  location move is recorded in `asset_location_history` (append-only), never overwriting prior rows.
- **11.3 — Parts stockout blocking work order completion.** A technician cannot transition a
  `MaintenanceWorkOrder` to `SUBMITTED` while it carries an unresolved `PARTS_NEEDED` block reason
  (§7/§9) — the status-transition API call is rejected with `422 PARTS_PENDING` (this module's
  validation hook registered with Work Orchestration's type-specific transition validators, the same
  extension point other `WorkItem` types use for type-specific completion rules). The technician must
  either record sufficient `PartUsage` (clearing the block) or an Operations Admin/manager must
  explicitly override with a documented reason (e.g., temporary field fix without the part, formal
  part backorder accepted) before completion is allowed.
- **11.4 — Duplicate breakdown reports for the same asset.** See FR-CMMS-022. Never silently merged;
  a genuine duplicate is linked (`possible_duplicate_of` → explicit user confirmation → the duplicate
  work order is cancelled with reason `DUPLICATE`, mirroring Module 06's `duplicate_of_issue_id`
  pattern) rather than deleted, preserving the reporter's audit trail.
- **11.5 — PM schedule targets a retired or since-deleted asset.** A `MaintenanceSchedule` whose
  target `Asset` is retired auto-pauses (FR-CMMS-005); one targeting an `AssetType` + location-scope
  where zero assets currently match simply materializes nothing for that occurrence (not an error) —
  it resumes materializing automatically the moment a matching asset is registered or un-retired at
  that scope.
- **11.6 — Cost-threshold approval and stock decrement interaction.** A `MaintenanceWorkOrder`
  requiring cost approval (§4 FR-CMMS-016) can still have parts usage *recorded* (and thus stock
  decremented) before approval completes, if the technician has already begun work under an
  emergency-repair exception; the approval, in this case, is retroactive/attestation-style rather than
  gating. Tenant policy configures whether cost approval is gating (default: gating for scheduled PM,
  non-gating/attestation for `CRITICAL`-severity ad-hoc breakdowns, so a safety-critical repair is
  never blocked on a manager's approval latency).

## 12. Non-Functional Requirements

- **NFR-CMMS-001 — PM materialization accuracy.** A due `MaintenanceWorkOrder` is materialized within
  the same accuracy bound as Work Orchestration's own recurrence engine (materialization window
  reconciled at least daily, per HLD §5) since it reuses that identical Temporal pattern.
- **NFR-CMMS-002 — API performance.** Asset and inventory read/write endpoints meet the platform p95
  < 300 ms target (source-spec §23) under normal load; the maintenance-history timeline endpoint
  (which merges multiple event sources) is allowed a relaxed p95 < 500 ms.
- **NFR-CMMS-003 — Stock-decrement correctness under concurrency.** Two technicians recording parts
  usage against the same `PartInventoryItem` concurrently must never both succeed in decrementing
  below zero, nor lose an update (LLD §6 concurrency design) — this is a correctness requirement, not
  merely a performance one, since incorrect stock counts cascade into false low-stock alerts and
  false stockout blocks.
- **NFR-CMMS-004 — Scale.** Designed for the platform's stated scale target (source-spec §23):
  assumed asset density of up to several hundred assets per location across 100k locations (tens of
  millions of `Asset` rows), with `MaintenanceWorkOrder` volume a subset of the platform's overall
  millions-of-work-items-per-day target.
- **NFR-CMMS-005 — Auditability.** Every asset status transition, location move, parts usage, and PM
  schedule change is recorded in an append-only history table sufficient to reconstruct full asset
  history without relying on mutable current-state rows, consistent with Product Principle P7.

## 13. Dependencies

| Module | Dependency |
|---|---|
| Work Orchestration (EPIC-03) | Owns the `MAINTENANCE_WORK_ORDER` `WorkItem` type's execution state machine, assignment, escalation ladder, and approval routing (for cost-threshold work orders); this module creates/reads `WorkItem` rows via its public/internal API and consumes `task.completed`/`task.overdue` events. |
| Evidence (EPIC-06) | Before/after photos and meter-reading proof captures are `Evidence` records referenced by ID from `MaintenanceWorkOrder`s and `meter_readings`. |
| Issues & CAPA (Module 06) | Recurring-failure patterns on an `Asset` (FR-CMMS-023) create an `Issue` via `create_issue`; this module does not run its own root-cause/CAPA workflow. |
| Integrations (EPIC-13) | ERP parts-catalog and vendor master-data sync via the Integration Hub's connector pattern; this module remains authoritative for operational stock levels regardless of ERP integration state. |
| Field Service & Vendor Scheduling (Module 25) | A `MaintenanceWorkOrder` assigned `assignee_type = VENDOR` hands off vendor-facing dispatch, appointment scheduling, and vendor SLA tracking to Module 25; this module only records the `Vendor` reference and the assignment event. |
| Inventory, Procurement & Supplier Management (Module 19) | Low-stock alerts (FR-CMMS-042) auto-create a purchase requisition via that module's API when enabled; PO approval routing and supplier scorecards remain Module 19's responsibility. |
| IoT Sensor Platform (Module 17) | A `Sensor` bound to an `Asset` can supply a "latest meter value" feed into meter-based PM calculation; this module does not ingest raw high-volume sensor time-series. |
| Identity/Org (EPIC-01/02) | `UserScope`/`OrganizationNode` resolve technician service-area eligibility, location scoping for assets, and approval-route eligible approvers for cost-threshold work orders. |
| Analytics & Control Tower (EPIC-12/15) | Consumes asset downtime, PM-compliance, and repeat-failure events/CDC feed for the "preventive intervention rate" and related KPIs. |

## 14. Resolved Decisions

- **Meter readings are captured manually for v1** (technician enters a value, optionally with a photo
  as `Evidence`); automated meter feeds from bound `Sensor`s (Module 17) are an additive input once
  that module ships, not a v1 dependency. This module does not block on Module 17's delivery timeline.
- **`AssetType.custom_attributes_schema` reuses the same JSON-Schema validation approach as Issues &
  CAPA's custom root-cause-method schemas** (Module 06 FR-CAPA-040), rather than a third
  schema-validation mechanism on the platform — one validated-JSON-Schema pattern platform-wide keeps
  tooling (schema editors, validators) shared across modules.
- **`PartInventoryItem` catalog fields (description, unit_cost) are tenant-global by default, with
  per-location stock rows only, for v1.** Per-location catalog overrides (e.g., a regional vendor
  substitution changing the part number used at one location) are not built in v1. Rationale: a
  tenant-global catalog is the common case and avoids the reconciliation complexity of divergent
  per-location part definitions; this is revisited only if a pilot tenant's ERP integration
  demonstrably requires per-location catalog overrides, rather than building for a hypothetical need
  ahead of real integration data.
- **The failure-triggered `Issue` threshold (FR-CMMS-023: "3 breakdowns in 30 days") is a single
  tenant-level default for v1, not configurable per `AssetType`/`criticality`.** A single, simple
  threshold is easier for an Operations Admin to reason about at launch than a matrix of
  per-asset-type thresholds with no usage data yet to calibrate them; per-`AssetType` override is a
  fast-follow once Analytics shows which asset types actually need a different threshold, avoiding
  building configurability the data hasn't yet justified.
- **`MaintenanceWorkOrder` cost-approval gating for `CRITICAL`-severity ad-hoc breakdowns (§11.6) is
  tenant-configurable from day one, defaulting to non-gating for `CRITICAL`.** Rationale: safety/
  regulatory posture varies significantly by industry vertical on this platform (retail vs.
  manufacturing floor vs. healthcare-adjacent site ops), so a single hard platform default would be
  wrong for a meaningful share of tenants; non-gating is the safer default because it never delays a
  safety-critical repair behind a manager's approval latency, and a tenant with a stricter compliance
  posture opts into gating explicitly.
