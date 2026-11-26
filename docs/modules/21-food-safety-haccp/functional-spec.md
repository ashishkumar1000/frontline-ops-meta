# Food Safety & HACCP — Functional Specification

Corresponds to **EPIC-16-class new scope** (no dedicated epic in Appendix E; traced to
`EPIC-04` Form Platform, `EPIC-07` Issues & CAPA, and IoT Sensor Platform's epic for
implementation tickets — see §7 Dependencies for the per-requirement mapping). Tier-3
vertical-specific module for restaurant/QSR/food-retail tenants, per `new-modules-addendum.md`
Module Map row 21. Reference: `source-spec.md` (Appendix A — Golden End-to-End Scenario, a
temperature exception; Appendix B — Example Rule Specification, `TEMP-001`), `conventions.md`
(tech stack, multi-tenancy, API/event/state-machine conventions), `new-modules-addendum.md`
(Module 21 row: "largely a configuration layer on IoT Sensor Platform ... Forms & Workflow
Builder ... and Issues & CAPA"), `17-iot-sensor-platform/*` (sensor readings, alert rules,
breach-to-Issue pattern this module binds into), `03-forms-workflow-builder/*` (template/
submission contract used for manual CCP checks), `06-issues-capa/*` (Issue lifecycle reused for
CCP breaches).

## 1. Purpose and Scope

### 1.1 Purpose

Every food-safety regulatory framework (FDA Food Code, Codex Alimentarius, and their national/
local equivalents) is built on the same seven-principle HACCP methodology: analyze hazards,
identify critical control points, set critical limits, monitor those limits, define corrective
actions for a deviation, verify the system works, and keep records. The platform's own golden
end-to-end scenario (source-spec.md Appendix A: a 10.2°C temperature reading triggering a
mandatory corrective action, an Issue, and eventual verification) is *already* a HACCP monitoring
event in everything but name — it just is not yet organized under a HACCP plan, is not yet tied
to a specific Critical Control Point (CCP), and does not yet carry food-safety-specific corrective
action vocabulary ("discard product," "reheat to 165°F," "reject delivery") instead of generic
CAPA language.

This module's purpose is to formalize that pattern for regulated food-safety compliance: it lets a
QA/Food Safety Admin author a HACCP plan (process flow → hazard analysis → CCP list) as structured
metadata, bind each CCP to either a Forms-backed manual check or an IoT Sensor Platform automated
reading stream, and route every breach through the existing Issues & CAPA lifecycle with
food-safety-specific corrective-action recording bolted on. It does **not** build a new execution
engine, a new time-series store, or a new exception-management state machine — those already exist
in modules 17, 03, and 06 respectively, and this module is explicitly a **vertical configuration
and orchestration layer** over them, matching the addendum's Tier-3 guidance ("position it
explicitly as a vertical accelerator/template pack over existing platform primitives, not a new
architectural layer").

### 1.2 In Scope

- Authoring a `HACCPPlan` as structured metadata following the standard 7-principle methodology:
  process flow steps → hazard analysis per step → CCP identification → critical limits →
  monitoring procedures → corrective actions → verification procedures → record-keeping
  requirements (§3.1).
- Defining a `CriticalControlPoint` (CCP) with a critical limit, a monitoring frequency, and a
  monitoring *method* that is either **manual** (a Forms-backed `WorkItem`/`Submission`) or
  **automated** (an `IoT Sensor Platform` `SensorAlertRule` binding) — never a third,
  module-specific monitoring mechanism (§3.2).
- Generating and keeping in sync the Forms `TemplateVersion` used for a manual CCP check, and the
  IoT `SensorAlertRule` used for an automated CCP, from the CCP's own configuration, so a QA Admin
  edits one CCP definition rather than a form and a sensor rule separately (§3.3).
- Capturing a manual temperature/cook-log/receiving-inspection reading via a Forms-backed
  `WorkItem` that this module configures but does not render or store submissions for (§3.3).
- Consuming automated CCP readings from the IoT Sensor Platform's reading stream and alert-rule
  breach mechanism, with this module owning only the CCP-specific interpretation layer on top
  (§3.4).
- CCP breach handling: a food-safety-specific deviation procedure and corrective action (e.g.
  "discard product," "re-cook to temperature," "reject delivery," "recalibrate equipment") recorded
  as a `FoodSafetyCorrectiveAction`, which is a specialized, food-safety-worded front end over the
  same underlying `Issue` CAPA-step mechanism Issues & CAPA already provides — not a parallel issue
  lifecycle (§3.5).
- Verification and record review: a Store/Site Manager's shift-level or daily sign-off over a
  location's HACCP monitoring records (§3.6).
- Audit-ready record export: a regulator- or auditor-facing export of a location's HACCP plan
  version, CCP monitoring history, breaches, and corrective actions for a given date range (§3.7).
- Multi-unit HACCP plan template reuse: publishing one HACCP plan template across many locations
  with local critical-limit overrides where regulation or local practice allows (§3.1, §5.3).

### 1.3 Out of Scope

- **Sensor ingestion, alert-rule evaluation mechanics, sensor health/offline detection.** All owned
  by IoT Sensor Platform (Module 17). This module only *configures* a `SensorAlertRule` (via
  Module 17's authoring API) scoped to a CCP and *reads* Module 17's breach events; it never
  ingests a raw sensor reading itself or re-implements debounce/sustained-duration evaluation.
- **Form rendering, form logic engine, offline form capture/sync.** All owned by Forms & Workflow
  Builder (Module 03) and Mobile & Offline (Module 04). This module only authors the
  `TemplateVersion` content (field list, validation, risk-scoring rule) for a manual CCP check and
  reads the resulting `Submission`.
- **Issue lifecycle, SLA timers, root-cause analysis methods, generic CAPA step mechanics.** All
  owned by Issues & CAPA (Module 06). A CCP breach creates/uses an `Issue` through the same
  internal path IoT Sensor Platform and Form Platform already use; this module does not define a
  second exception state machine. `FoodSafetyCorrectiveAction` is additive vocabulary/UI over an
  `Issue`'s `CORRECTIVE_ACTION` CAPA step, not a replacement for it.
- **Evidence storage, media hashing/retention.** Owned by Evidence (Module 05); this module
  attaches evidence (e.g. a photo of discarded product) via Evidence's existing API.
- **General (non-food-safety) safety incidents, injury reporting, regulatory OSHA-style
  reporting.** Owned by EHS — Safety Incident & Risk Register (Module 20); a food-safety event is
  not routed to that module unless it also constitutes a workplace safety incident (rare, e.g. a
  burn during a "discard product" action — cross-linked, not merged).
- **General inventory/receiving workflows** (PO receipt, supplier scorecards) — owned by Inventory,
  Procurement & Supplier Management (Module 19); this module only cares about receiving as a CCP
  (e.g. cold-chain receiving temperature check), not the full receiving/PO workflow.
- **Jurisdiction-specific regulatory content authoring as a managed service** (i.e., this platform
  does not employ food-safety regulatory counsel to keep a jurisdiction's official limits current on
  the tenant's behalf) — see Resolved Decisions §8 for the firm decision to ship a tenant-editable
  starter template library rather than a fully-managed regulatory content subscription.

## 2. Personas

| Persona | Primary job | Primary product surface |
|---|---|---|
| Frontline Associate | Perform manual temperature checks and cook/cooling logs at their assigned CCPs during a shift; respond to a CCP breach with an immediate, mandatory corrective action before the check can be marked complete. | Mobile App |
| Store/Site Manager | Own the location's HACCP compliance day-to-day: see which CCPs are due/overdue, review and sign off a shift's monitoring records, act on breach-triggered Issues, decide on discard/hold/release for flagged product. | Mobile App + Manager Workspace |
| QA / Food Safety Admin | Author and publish `HACCPPlan`s (process flow → hazard analysis → CCP list); define each CCP's critical limit, monitoring method (manual/automated), and corrective action vocabulary; review compliance trends and audit-readiness across locations; manage multi-unit template reuse and local critical-limit overrides. | Web Admin Console — HACCP Plan Builder |
| Compliance Monitor Agent | Consume CCP breach patterns (via the same `RULE_TRIGGERED`/`FORM_TRIGGERED` Issue sources it already reads for any location) to detect repeated CCP failures and recommend escalation, exactly as it does for Appendix A's temperature-exception scenario, with no new agent tool required at MVP. | Agent Runtime |

## 3. Functional Requirements

### 3.1 HACCP Plan Authoring (7-Principle Structure)

- **FR-HACCP-001**: A QA/Food Safety Admin authors a `HACCPPlan` as a versioned document containing,
  per the standard 7-principle methodology: (1) a `process_flow` (ordered list of process steps,
  e.g. "Receiving → Cold Storage → Prep → Cooking → Hot Holding → Cooling → Reheating → Service"),
  (2) a `hazard_analysis` per process step (biological/chemical/physical hazard, likelihood,
  severity), (3) the derived `critical_control_points` list (§3.2), (4) `critical_limits` per CCP,
  (5) `monitoring_procedures` per CCP (§3.2), (6) `corrective_action_definitions` per CCP (§3.5),
  and (7) `verification_procedures` (§3.6) and `record_keeping_requirements` (§3.7). This mirrors
  the classic HACCP plan document structure so a QA Admin familiar with the paper/spreadsheet
  version of a HACCP plan recognizes the same shape here.
- **FR-HACCP-002**: A `HACCPPlan` follows the same versioning discipline as a `TemplateVersion`:
  `DRAFT -> REVIEW -> APPROVED -> PUBLISHED -> SUPERSEDED -> RETIRED` (reusing the platform's
  `TemplateVersion` state machine verbatim per conventions.md, since a HACCP plan version is,
  structurally, a specialized template version — see HLD §3 for the data-modeling decision). A
  published plan version is immutable; a completed CCP monitoring record always references the
  exact plan version in force when it was captured (platform principle P7 — Historical Truth).
- **FR-HACCP-003**: A `HACCPPlan` is authored once as a **template** at brand/enterprise level and
  **assigned** to one or more `OrganizationNode`s (locations); a location-assigned plan may carry
  **local critical-limit overrides** where the QA Admin has explicitly marked that CCP's limit as
  override-eligible (e.g. a state/local health code allows 41°F or 45°F cold-holding depending on
  jurisdiction) — see Edge Case §5.3 for override governance.
- **FR-HACCP-004**: Before publishing a plan version, the QA Admin sees an Impact Preview (reusing
  Forms & Workflow Builder's Impact Preview mechanism, §6 of that module's spec) listing affected
  locations, active CCP monitoring `WorkItem` schedules, and any manual-check `TemplateVersion`s
  that will be superseded.
- **FR-HACCP-005**: A `HACCPPlan` can be cloned from an existing plan (own or a starter library
  plan, see §8) as the starting point for a new plan, preserving process-flow/hazard-analysis
  structure while letting the QA Admin adjust CCPs, limits, and monitoring methods per concept or
  menu.

### 3.2 Critical Control Point (CCP) Definition

- **FR-HACCP-010**: A QA/Food Safety Admin defines a `CriticalControlPoint` with: `process_step`
  (reference into the plan's `process_flow`), `hazard_type` (`BIOLOGICAL`, `CHEMICAL`, `PHYSICAL`),
  `critical_limit` (structured: `metric` e.g. `TEMPERATURE_F`/`TEMPERATURE_C`/`TIME_MINUTES`/
  `PH`/`WATER_ACTIVITY`, `operator`, `value`, optional `value_max` for a range), `monitoring_
  frequency` (e.g. every 2 hours, once per delivery, continuous), and `monitoring_method`
  (`MANUAL` or `AUTOMATED`, §3.2.1–3.2.2). This is a direct generalization of Appendix B's
  `TEMP-001` rule shape and IoT Sensor Platform's `SensorAlertRule` condition shape, applied at the
  CCP-definition layer.
- **FR-HACCP-011**: A CCP's `monitoring_method` determines which binding this module manages:
  1. **MANUAL** — this module generates/updates a Forms `TemplateVersion` (via Module 03's
     authoring API) with one field per critical-limit metric, mandatory-field logic tied to the
     critical limit (`require_field` when out of range, mirroring Appendix B), and a `WorkItem`
     schedule matching `monitoring_frequency` (via Work Orchestration's scheduling primitives,
     `WorkItemType = TASK` with a food-safety category tag).
  2. **AUTOMATED** — this module generates/updates a `SensorAlertRule` (via Module 17's
     authoring API) scoped to a bound `Sensor`/`sensor_tag` at the CCP's location, with
     `condition_type` chosen from Module 17's supported set (`THRESHOLD` or `SUSTAINED_DURATION`
     are the two recommended for CCP use; `RATE_OF_CHANGE` is available but rare for a CCP), and
     `actions` including `create_issue: true`, `require_field: corrective_action`, and
     `attach_as_evidence: true`.
- **FR-HACCP-012**: A CCP may be defined with monitoring_method `MANUAL` even where an automated
  sensor would be technically feasible (e.g. a small independent-franchise tenant without sensor
  hardware); the module does not require IoT Sensor Platform adoption as a precondition for using
  Food Safety & HACCP.
- **FR-HACCP-013**: A CCP's critical limit, once referenced by a `PUBLISHED` `HACCPPlan` version,
  cannot be edited in place; a change creates a new plan version (FR-HACCP-002) so historical
  monitoring records remain interpretable against the limit that was actually in force (platform
  principle P7).
- **FR-HACCP-014**: A CCP can be marked `visual_distinction = MANUAL` or `AUTOMATED` for UI purposes
  independent of the underlying binding mechanics, so a manager's CCP list visually distinguishes
  "an associate must go check this" from "a sensor is watching this continuously" at a glance (see
  UX States §4).

### 3.3 Manual Temperature/Cook-Log Capture (Forms-Backed)

- **FR-HACCP-020**: A manual CCP check renders as a standard Forms-backed `WorkItem` (`WorkItemType
  = TASK`, category `FOOD_SAFETY_CCP_CHECK`) on the Frontline Associate's Today list, scheduled per
  the CCP's `monitoring_frequency` using Work Orchestration's existing scheduling primitives
  (calendar-based recurrence, shift-based, or event-triggered — e.g. "on delivery receipt").
- **FR-HACCP-021**: The generated `TemplateVersion` for a manual CCP check includes: the metric
  input field (number/temperature, matching the CCP's `critical_limit.metric`), a calculated
  in-range/out-of-range indicator (Forms' calculated-field logic evaluating against the CCP's
  `critical_limit` at fill time), and a conditionally-required `corrective_action` field-group that
  becomes mandatory the moment the calculated indicator evaluates out-of-range (Forms' conditional
  mandatory-condition logic, §8.2 of source-spec, reused verbatim).
- **FR-HACCP-022**: The Associate cannot submit (complete) a manual CCP check `WorkItem` while the
  calculated indicator is out-of-range and the `corrective_action` field-group is incomplete — this
  is enforced by Forms' own cross-field validation, not a bespoke validation layer in this module
  (Edge Case §5.2).
- **FR-HACCP-023**: A manual CCP check `Submission` is linked back to its originating
  `CriticalControlPoint` and `HACCPPlan` version via a `ccp_id`/`haccp_plan_version_id` reference
  carried in the `TemplateVersion`'s metadata, so compliance-history queries (§3.6, §3.7) can join
  submissions to CCPs without this module owning a duplicate submission-content table (see HLD §3
  data-ownership decision).
- **FR-HACCP-024**: Multiple readings for the same CCP within one shift (e.g. an hourly walk-in
  cooler check) each produce their own `Submission`; this module does not aggregate them into a
  single mutable record — the shift-level view (§3.6) is a read-time rollup over the individual
  submissions.

### 3.4 Automated CCP Monitoring (IoT-Backed)

- **FR-HACCP-030**: An automated CCP's readings flow entirely through IoT Sensor Platform's
  ingestion and rule-evaluation pipeline (Module 17 §3.2–3.4); this module's only run-time
  involvement is (a) having authored the `SensorAlertRule` at CCP-definition time (FR-HACCP-011.2)
  and (b) tagging the resulting `Issue`/`SensorAlertEvent` with the CCP reference so compliance
  history and audit export (§3.6, §3.7) can attribute a sensor-driven breach to a specific CCP.
- **FR-HACCP-031**: An automated CCP's current-status tile (Manager Workspace, mobile) reads
  directly from IoT Sensor Platform's latest-reading cache and trend API (Module 17 §2.6) rather
  than this module maintaining a parallel "current temperature" cache.
- **FR-HACCP-032**: When a CCP is reconfigured from `MANUAL` to `AUTOMATED` (or vice versa) — e.g. a
  location installs a sensor for a previously manually-monitored cooler — this module closes the
  old binding (Forms schedule or sensor rule) and opens the new one as of the plan version that
  introduces the change; historical records under the old method remain queryable and correctly
  attributed to their original monitoring method (Edge Case §5.5).

### 3.5 CCP Breach Handling (Food-Safety-Specific Corrective Action)

- **FR-HACCP-040**: A CCP breach — whether from a manual check's out-of-range submission or an
  automated sensor rule firing — results in the same underlying `Issue` creation path already used
  platform-wide (`POST /internal/issues` for the automated/rule path per Module 17's contract; the
  Form Platform's existing rule-triggered path for the manual path), with `source_ref` additionally
  carrying `ccp_id` and `haccp_plan_version_id`. This module introduces **no second issue-creation
  mechanism**.
- **FR-HACCP-041**: On breach, this module presents the Associate/Manager with a **food-safety
  corrective-action picker** scoped to the specific CCP's `hazard_type` and process step, using
  vocabulary a generic CAPA screen does not: `DISCARD_PRODUCT`, `REJECT_DELIVERY`,
  `RECOOK_TO_TEMPERATURE`, `RECOOL_AND_RETEST`, `RECALIBRATE_EQUIPMENT`, `HOLD_FOR_MANAGER_REVIEW`,
  `OTHER` (free text, requires a note). The QA Admin configures which options are valid per CCP at
  plan-authoring time (FR-HACCP-001's `corrective_action_definitions`).
- **FR-HACCP-042**: Selecting a food-safety corrective action records a `FoodSafetyCorrectiveAction`
  row and simultaneously satisfies the underlying `Issue`'s `CORRECTIVE_ACTION` CAPA step
  (Issues & CAPA §10.3) — it is a specialized, food-safety-worded **front end** over the same CAPA
  step, not a parallel record disconnected from the Issue lifecycle. The Issue's CAPA step content
  is populated from the `FoodSafetyCorrectiveAction`'s structured fields (action type, quantity/
  product discarded, evidence).
- **FR-HACCP-043**: `DISCARD_PRODUCT` and `REJECT_DELIVERY` corrective actions require at least one
  photo `Evidence` attachment (of the discarded product/rejected delivery) before the action can be
  recorded as complete — mirroring the platform's evidence-first principle (P4) for the corrective
  actions with the highest food-cost and liability stakes.
- **FR-HACCP-044**: A CCP breach's corrective action **must be recorded before the originating
  `WorkItem`/check can be marked complete** (manual path) or before the Issue can leave
  `CORRECTIVE_ACTION` status (automated path) — the check cannot be dismissed or skipped past a
  failed CCP (Edge Case §5.2, UX §4 "CCP-breach-blocking-flow").
- **FR-HACCP-045**: A resolved CCP breach still requires the standard Issues & CAPA verification
  step (a manager or a scheduled re-check confirms the corrective action was effective) before the
  Issue reaches `RESOLVED`/`CLOSED` — this module does not shortcut Issues & CAPA's verification
  requirement for food-safety breaches; if anything, tenants commonly configure food-safety
  categories to *require* verification even where their policy allows auto-resolution for lower-risk
  categories (Issues & CAPA policy configuration, referenced not owned here).

### 3.6 Verification & Record Review

- **FR-HACCP-050**: A Store/Site Manager can open an **end-of-shift (or end-of-day) HACCP log
  review checklist**: a read-time rollup of every CCP monitoring record (manual submissions +
  automated sensor summaries) captured in the shift/day window at their location, grouped by CCP,
  showing in-range/out-of-range/inconclusive status and linked breach Issues.
- **FR-HACCP-051**: The Manager's review produces a `haccp_shift_review` sign-off record
  (`reviewed_by`, `reviewed_at`, `period_start`/`period_end`, `location_id`, `outstanding_
  breach_count` at time of sign-off) — this is itself a lightweight `WorkItem` (`WorkItemType =
  APPROVAL`, category `HACCP_SHIFT_REVIEW`) so it inherits Work Orchestration's existing
  assignment/overdue/escalation handling rather than this module building a parallel approval
  mechanism.
- **FR-HACCP-052**: A Manager cannot sign off a shift review while any CCP breach in the reviewed
  window has an unrecorded or unverified corrective action — the sign-off screen surfaces those
  breaches inline and blocks submission until each is either resolved or explicitly acknowledged as
  still open with a reason (matching the platform's "verified completion, not merely reported
  completion" principle).

### 3.7 Audit-Ready Record Export

- **FR-HACCP-060**: A QA Admin or auditor-facing role can export, for a location and date range: the
  `HACCPPlan` version(s) in force, every CCP monitoring record (manual submissions + automated
  sensor summaries with breach windows), every `FoodSafetyCorrectiveAction`, every shift-review
  sign-off, and evidence references — as a paginated JSON/CSV export and a print-ready PDF summary,
  matching the format regulators/auditors expect from a paper HACCP log binder.
- **FR-HACCP-061**: The export is reproducible against historical plan versions: a record captured
  under `HACCPPlan` v3 renders against v3's critical limits and CCP definitions even after v5 is
  published, per platform principle P7 (Historical Truth) and mirroring Module 03's template-version
  immutability guarantee for submissions.
- **FR-HACCP-062**: Export generation is an asynchronous job (large date ranges / many locations)
  with a completion notification and a signed, time-limited download URL, consistent with the
  platform's large-export pattern used elsewhere (e.g. Analytics exports).

## 4. UX States

| State | Surface | Behavior |
|---|---|---|
| Temp-check-due reminder | Frontline mobile, push + Today list | A manual CCP check appears in "Next" with a due countdown; overdue crosses into the platform's standard `OVERDUE -> ESCALATED` WorkItem states, escalating to the Store Manager per the CCP's configured SLA. |
| CCP-breach-blocking-flow | Frontline mobile, in-line on the check form | The moment the entered value is out-of-range, the screen locks into a blocking sub-flow: corrective-action picker (FR-HACCP-041) appears inline, "Submit" is disabled until an action (and required evidence, FR-HACCP-043) is recorded; there is no "skip"/"dismiss" affordance — this is a hard gate, not a warning banner. |
| Manual-vs-automated visual distinction | Manager Workspace CCP list, mobile CCP list | A manual CCP shows a clipboard/hand icon with "Next check due {time}"; an automated CCP shows a sensor/wave icon with a live current-value tile (reading from IoT Sensor Platform's latest-reading cache) and "Continuously monitored." A sensor-offline automated CCP additionally shows the fallback banner below. |
| Sensor-offline fallback banner | Manager Workspace, mobile, on an automated CCP whose bound sensor is `OFFLINE`/`DEGRADED` | "Sensor offline since {time} — manual check required" banner with a one-tap action that spawns an ad-hoc manual CCP check `WorkItem` for that CCP (Edge Case §5.1); the CCP's tile switches from the live-value display to this banner for the duration of the outage. |
| End-of-shift log review checklist | Manager Workspace | Grouped-by-CCP list with in-range/out-of-range/inconclusive counts, a "Sign off shift" action gated per FR-HACCP-052, and drill-through to any individual submission or breach Issue. |
| Empty state | HACCP Plan Builder (QA Admin) | "No HACCP plan yet" with a choice to start from a blank plan or clone a starter template from the library (§8), rather than a bare empty screen. |
| Loading | Compliance history / audit export | Skeleton list while historical records resolve; large exports show a background-job progress state, not a blocking spinner (FR-HACCP-062). |
| Permission-denied | HACCP Plan Builder, audit export | Standard `403 SCOPE_DENIED` — a Frontline Associate never sees the Builder; a Regional Manager without export scope at a location sees the compliance dashboard but not the raw export action. |
| Offline (mobile) | Manual CCP check | Fully offline-capable like any Forms-backed WorkItem (Mobile & Offline module); the breach-blocking flow (above) works identically offline — the corrective-action selection is captured locally and syncs with the submission, never deferred to "when back online." |

## 5. Validation and Edge Cases

- **5.1 Sensor-based CCP goes offline.** Per IoT Sensor Platform's health-state machine, an
  automated CCP's bound sensor transitioning to `OFFLINE`/`DEGRADED` does not mean "in compliance"
  — it means "not being watched." This module treats a sensor-offline automated CCP as requiring a
  **fallback to manual capture**: the CCP's tile shows the fallback banner (UX §4), and this module
  auto-spawns an ad-hoc manual CCP-check `WorkItem` (reusing the CCP's own `TemplateVersion`
  generation logic, FR-HACCP-011.1, generated on demand rather than pre-scheduled) at the CCP's
  configured `monitoring_frequency` for the duration of the outage, so a required-by-regulation
  check cadence is never silently skipped just because the sensor stopped reporting. This mirrors
  IoT Sensor Platform's own `RULE_INCONCLUSIVE` handling (Module 17 §5.1) at the CCP layer: absence
  of a reading is never treated as evidence of compliance.
- **5.2 Cannot skip past a failed CCP.** A breach's corrective action must be recorded before (a) a
  manual check's `WorkItem` can transition `IN_PROGRESS -> SUBMITTED` (enforced by Forms'
  conditional-mandatory validation, FR-HACCP-022) and (b) an automated CCP's resulting `Issue` can
  leave `CORRECTIVE_ACTION` status (enforced by Issues & CAPA's own CAPA-step-completion gate,
  unchanged — this module supplies the food-safety-worded action, Issues & CAPA enforces the gate).
  There is no "acknowledge and dismiss" path that bypasses this for either monitoring method.
- **5.3 Multi-unit plan template reuse with local critical-limit overrides.** A CCP's critical limit
  is override-eligible only if the QA Admin explicitly flagged it as such at plan-authoring time
  (FR-HACCP-003); an override at a location can only *tighten*, never loosen, the brand-level limit
  unless the plan is explicitly marked `jurisdiction_override_allowed = true` for that CCP (default
  false) — this prevents a location from quietly relaxing a food-safety limit below the corporate
  standard while still allowing a location in a jurisdiction with a stricter local code to tighten
  it. Every override is recorded with a `reason` and an approving QA Admin, and is visible on the
  audit export (§3.7) as an explicit deviation from the brand-level plan.
- **5.4 CCP check completed but for the wrong process step / mislabeled asset.** If a manual check's
  bound location/asset context doesn't match the CCP's configured process step (e.g. an associate
  logs a walk-in cooler reading against the wrong cooler's CCP because two CCPs share a similar
  name), the submission is still accepted (never silently dropped) but flagged
  `ccp_context_mismatch_suspected` for QA Admin review, mirroring IoT's `clock_skew_suspected`
  non-blocking flag pattern — this module does not attempt automated correction of an ambiguous
  match.
- **5.5 Monitoring method changed mid-plan-version (manual → automated or vice versa).** Per
  FR-HACCP-032, the change takes effect only via a new plan version (never edits a `PUBLISHED`
  version in place); historical records retain the monitoring method that was actually active when
  captured, and the audit export (§3.7) shows the transition point explicitly rather than presenting
  a homogenized history.
- **5.6 A CCP breach occurs during an already-open Issue for the same CCP.** Follows IoT Sensor
  Platform's existing-open-breach guard (Module 17 §5.5(c)) for the automated path: a repeat
  automated breach on an already-open Issue for the same `(ccp_id, sensor rule)` attaches new
  evidence rather than opening a duplicate Issue. For the manual path, a second out-of-range
  submission for a CCP with an already-open Issue is linked to the existing Issue (via
  `source_ref.ccp_id` match) rather than opening a duplicate, following the same idempotent-issue
  convention (Issues & CAPA §2.1 unique-index pattern), extended here with `ccp_id` as part of the
  dedupe key for the manual/form-triggered path.
- **5.7 Regulatory audit export requested for a plan version that has since been retired.**
  Supported without restriction — a `RETIRED` plan version's structure and the records captured
  under it remain fully queryable and exportable (they are never deleted on retirement, only marked
  no-longer-assignable to new locations), consistent with platform principle P7.

## 6. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-HACCP-001 | Breach-to-corrective-action-gate latency | The blocking corrective-action UI must appear within the same interaction (no server round-trip required) for the manual path — the out-of-range calculation is a client-side/offline-capable Forms calculated field (Module 03), not a server call, so the gate works identically online and offline. |
| NFR-HACCP-002 | Plan/CCP configuration propagation | A published `HACCPPlan` version's CCP-derived `TemplateVersion`/`SensorAlertRule` changes must propagate to affected locations' mobile devices within the platform's standard template-sync window (Module 04's offline sync cadence) — no bespoke faster-path sync mechanism is introduced for this module. |
| NFR-HACCP-003 | Audit export completeness | An audit export for a given location/date range must be provably complete (no silently-omitted breach or corrective action) — verified by a reconciliation check comparing the export's record count against the underlying Issue/Submission/SensorAlertEvent counts for the same window before the export is marked `READY`. |
| NFR-HACCP-004 | Multi-tenant/multi-brand isolation | Standard platform multi-tenancy rule (conventions.md) — no cross-tenant HACCP plan, CCP, or corrective-action visibility under any query path, consistent with the shared `tenant_id` + RLS backstop used throughout. |
| NFR-HACCP-005 | Regulatory retention | HACCP records (monitoring records, corrective actions, shift reviews) default to a **minimum 2-year retention** (a common baseline referenced by FDA Food Code-aligned recordkeeping expectations; tenant-configurable upward, never below the platform's default, per source-spec §21.4's data lifecycle and the EHS module's precedent for regulatory-retention records that outlive normal operational retention windows). |

## 7. Dependencies

- **IoT Sensor Platform (Module 17)** — authoritative source for automated CCP readings, alert-rule
  evaluation, breach detection, sensor health state, and trend/history queries; this module
  authors `SensorAlertRule`s scoped to a CCP via Module 17's existing authoring API and reads its
  breach/health events (FR-HACCP-011.2, §3.4, Edge Case §5.1). Traces to `EPIC-16`-class scope.
- **Forms & Workflow Builder (Module 03)** — authoritative source for manual CCP check templates,
  submissions, conditional-mandatory validation, and the Impact Preview mechanism this module reuses
  for HACCP plan publication (FR-HACCP-004, FR-HACCP-011.1, §3.3). Traces to `EPIC-04`.
- **Issues & CAPA (Module 06)** — authoritative source for the Issue lifecycle, SLA timers,
  CAPA-step gating, and repeat-failure signal the Compliance Monitor Agent reads; every CCP breach
  becomes/joins an `Issue` through Module 06's existing creation path, with `FoodSafetyCorrectiveAction`
  as a specialized front end over its `CORRECTIVE_ACTION` CAPA step (§3.5). Traces to `EPIC-07`.
- **Evidence (Module 05)** — photo/document evidence for discarded product, rejected deliveries, and
  any manual check's evidence-bearing fields (FR-HACCP-043).
- **Work Orchestration (Module 02)** — scheduling/assignment/SLA/escalation for manual CCP check
  `WorkItem`s and shift-review `WorkItem`s (FR-HACCP-020, FR-HACCP-051); this module introduces no
  parallel scheduling mechanism.
- **Mobile & Offline (Module 04)** — offline-capable rendering and sync for manual CCP checks,
  including the breach-blocking flow, which must work identically with zero connectivity
  (NFR-HACCP-001).
- **Identity & Organization (Module 01)** — `OrganizationNode`/`UserScope` govern which QA Admin can
  author plans for which brand/location, and which Manager/Associate sees which CCPs at their
  location.
- **Analytics & Control Tower (Module 11)** — CCP compliance-rate rollups, repeat-breach trend
  reporting, and the audit-export reconciliation feed for enterprise-level food-safety dashboards.
- **Agentic Operations (Module 10)** — the Compliance Monitor Agent consumes CCP-tagged Issues
  identically to any other Issue source; no new agent tool required at MVP (mirrors Module 17's
  same conclusion).

## 8. Resolved Decisions

- **Ship a jurisdiction-specific default HACCP template library (e.g. FDA Food Code-aligned) as a
  starter (FR-HACCP-005).** Firm decision: **yes**, as **tenant-editable starting templates**, never
  hardcoded/immutable regulatory content. Ship a small library of common QSR/restaurant HACCP plan
  templates ("Cold Holding," "Hot Holding," "Cooking," "Cooling," "Receiving") pre-populated with
  commonly-cited FDA Food Code critical limits (e.g. cold holding ≤ 41°F, hot holding ≥ 135°F, cooking
  minimums by product), clearly labeled in-product as a **starting point requiring the tenant's own
  QA/food-safety authority to review and approve for their specific jurisdiction and menu** before
  publishing. Rationale: the platform is not the regulatory authority of record and must not
  represent these as certified/compliant out of the box; this approach meaningfully accelerates a new
  tenant's time-to-first-published-plan (consistent with Module 03's template-cloning precedent)
  without the platform taking on unbounded liability for keeping jurisdiction-specific legal limits
  current.
- **`HACCPPlan` scope (FR-HACCP-003).** Firm decision: a `HACCPPlan` is scoped to a brand/concept and
  assigned to locations running that concept, never authored per-location from scratch. Rationale:
  matches the platform's general template/location-assignment pattern (Module 03) and this module's
  explicit framing as a configuration/template-pack layer, not a bespoke per-site authoring tool.
- **`FoodSafetyCorrectiveAction` is additive, not a second source of truth (FR-HACCP-042).** Firm
  decision: it is always additive UI/vocabulary over an `Issue`'s `CORRECTIVE_ACTION` CAPA step;
  Issues & CAPA remains the single lifecycle owner (HLD §1's non-ownership statement is authoritative
  and unchanged by this decision).
- **Predictive Food Safety Agent (proactive breach-risk flagging).** Firm decision: **out of scope
  for MVP.** No agent may proactively flag a CCP at elevated breach risk (e.g. via Module 17's
  anomaly-score field, FR-IOT-051) in this version. Rationale: Module 17's anomaly scoring needs
  real-world tuning across food-safety tenants before it is reliable enough to drive a food-safety-
  relevant prediction; this is deferred to a Phase 3 extension once that tuning data exists, and any
  such agent would in any case enter at L1 Recommend only per source-spec P5 (Policy before AI).
- **Jurisdiction-specific audit export template (FR-HACCP-060).** Firm decision: MVP ships **one**
  generic, comprehensive PDF/CSV/JSON export format, not a per-jurisdiction template system.
  Rationale: auditor/inspector interactions generally accept "show me your records" in a comprehensive
  generic format rather than requiring a specific proprietary form; building per-jurisdiction export
  templates before any tenant has asked for one would be speculative scope. A jurisdiction-specific
  export template system is explicitly deferred until a paying tenant's actual regulator requires one.
