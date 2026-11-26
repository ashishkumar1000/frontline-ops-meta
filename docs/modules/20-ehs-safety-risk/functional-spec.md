# EHS — Safety Incident & Risk Register — Functional Specification

Module corresponds to a new Tier-2 competitive-gap-closure module (`new-modules-addendum.md` row
20, `20-ehs-safety-risk`). Not present in the original v2.0 baseline (`source-spec.md`); extends it
consistently with `conventions.md` (tech stack, entity-naming style, API/event conventions,
diagram conventions, multi-tenancy rule). Source-spec touchpoints: §12.1 lists "PPE/safety" as a
**Visual AI** use case (a *detection* input, not this module's system of record); §7.1 lists
"Incident / Observation" as an existing **WorkItem type** (the schedulable/assignable *work* of
filing a report or performing an observation walk, reusing the canonical WorkItem state machine);
§10 (Evidence, Issues and CAPA) defines the `Issue` entity and CAPA machinery this module wraps
rather than replaces. Shortcode: `EHS`.

## 1. Purpose and Scope

### 1.1 Purpose

Generic operational exceptions (Issues & CAPA, EPIC-07) are designed for brand-standard,
food-safety-adjacent, and equipment deviations where the "exception" is the whole story: detect,
correct, verify, close. A formal workplace safety incident — an injury, illness, near-miss, or
property-damage event — carries a second, independent obligation the generic Issue model does not
model at all: a **regulatory classification and reporting/retention obligation** that exists
whether or not any corrective action is ever taken. A recordable injury must be logged within a
jurisdiction-defined window (e.g. an OSHA-style "record within 7 calendar days of the employer
learning of the case"), must be retained for a legally mandated period (commonly 5+ years,
independent of the tenant's platform-wide default retention policy), and must survive
reclassification, audit, and even the underlying corrective Issue being long since closed. This
module — the **EHS Service** — is a thin regulatory/classification and risk-management layer that
sits beside Issues & CAPA: it owns *what happened, how it is legally classified, what hazard/risk
it maps to, and when it must be reported and for how long it must be kept*, while delegating all
corrective-action execution machinery (root cause analysis, corrective/preventive action tasks,
effectiveness verification, reopening) to the existing Issues & CAPA state machine and CAPA steps.

It operationalizes the parts of the Product Promise (source-spec §2.2) that generic Issues &
CAPA under-serves for regulated safety events: "every deviation can become a managed exception"
still holds, but a safety incident *additionally* must never lose its regulatory record even after
the operational exception is fully closed — because auditors, regulators, and insurers query the
incident record years after the CAPA record's retention would otherwise have lapsed.

### 1.2 In Scope

- **Incident reporting**: capture and classification of workplace safety incidents — injury,
  illness, near-miss, and property-damage categories — including a configurable, per-jurisdiction
  regulatory severity/recordability taxonomy (OSHA-recordable-style, but not hardcoded to one
  country's rules).
- **Witness statement capture** attached to an incident record.
- **Root-cause linkage**: a `SafetyIncident` may spawn an `Issue` (via Issues & CAPA's
  `create_issue` tool/API) for corrective-action execution; this module tracks that linkage and
  listens for the Issue's resolution to close its own feedback loop, but does not re-implement root
  cause analysis, corrective/preventive action steps, or effectiveness verification.
- **Risk assessment authoring**: likelihood × severity risk matrix, standard risk-scoring
  methodology, residual-risk-after-controls computation.
- **Hazard register**: identified hazards, controls in place, residual risk, owner, review cadence,
  and asset/location linkage.
- **Safety observation program**: positive and negative behavioral/conditional observations
  (BBS-style), near-miss trending, feeding the `SAFETY_OBSERVATION` WorkItem type.
- **Regulatory reporting deadline tracking**: configurable per-jurisdiction rules for "recordable
  incidents must be logged/reported within X days," deadline countdown, approaching-deadline
  alerts, and missed-deadline flagging.
- **Regulatory retention**: a longer, legally mandated retention period for `SafetyIncident` and its
  attached records, distinct from and overriding the platform's default `Evidence`/record retention
  policy (Data Lifecycle, source-spec §21.4) for this record class.
- **Safety KPI source data**: incident/hazard/observation facts feeding TRIR/LTIR-style metrics
  computed in Analytics.
- **EHS-specific UX**: incident-report-in-progress with witness-add flow, reporting-deadline
  badges, risk-matrix visual picker, hazard-register-review-due surfacing, anonymous near-miss
  reporting.

### 1.3 Out of Scope (owned by other modules, referenced here)

- **Corrective-action execution machinery** — immediate correction, containment, root cause
  analysis (5 Whys/Fishbone/Pareto/FMEA/Custom), corrective/preventive action steps, effectiveness
  verification, reopening on failed verification, SLA/escalation ladders on the corrective-action
  side — all owned by **Issues & CAPA (EPIC-07)**. This module calls `create_issue` when a
  `SafetyIncident` requires corrective action and consumes `issue.resolved`/`issue.escalated`; it
  never duplicates CAPA steps or the `Issue` state machine. See functional-spec.md of module
  `06-issues-capa` §6–§9 for the machinery this module delegates to.
- **Generic operational exceptions** with no regulatory reporting/retention obligation (a wrong
  planogram, a late delivery) remain plain `Issue`s and never touch this module.
- **Evidence capture, storage, hashing, retention mechanics** — **Evidence module (EPIC-06)**. This
  module references `Evidence` records by ID for incident photos, scene documentation, and witness
  statement attachments, and additionally stamps a regulatory retention override on the specific
  Evidence records linked to a `SafetyIncident` (see §9, NFR-EHS-002), but does not implement
  storage/hashing itself.
- **Visual AI PPE-violation detection** (source-spec §12.1) — owned by **Visual AI (EPIC-10)**.
  Visual AI produces a structured detection result (`compliance_score`, `violations[]` per §12.2
  result contract); this module only consumes a PPE-violation detection as one possible *trigger*
  for a `SafetyIncident` or `SafetyObservation` record, and does not own the vision pipeline,
  confidence thresholds, or human-review workflow for the underlying image.
- **Training/microlearning content and completion tracking** — **Learning module (EPIC-09)**. This
  module creates a `LearningAssignment` reference when a hazard control or corrective action is
  training-linked, and consumes `training.completed` for context, but does not own course content.
- **Generic WorkItem scheduling/assignment mechanics** for the `Incident / Observation` and
  `SAFETY_OBSERVATION` WorkItem types (source-spec §7.1; addendum's new `SAFETY_OBSERVATION`
  WorkItem type) — owned by **Work Orchestration (EPIC-03)**. This module is the *functional owner*
  of what happens when that WorkItem type is submitted (classification, risk scoring, regulatory
  logic), the same "functional owner of an existing WorkItem type" pattern the addendum uses for
  Employee Engagement's `Survey` WorkItem.
- **TRIR/LTIR-style enterprise/regional rollups and safety dashboards** — **Analytics & Control
  Tower (EPIC-12/15)**, consuming this module's events and CDC feed.
- **Agent reasoning/policy/autonomy enforcement** for any future Safety Agent — **Agent Runtime
  (EPIC-11)**. This module would expose typed tools (e.g. `create_safety_incident`,
  `get_hazard_register`) but is not in scope for this baseline to define agent policy.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Frontline Associate | Reports an injury, illness, near-miss, or property-damage incident (including anonymously for near-misses); may be asked to give a witness statement on someone else's incident; submits positive/negative safety observations. |
| Store/Site Manager | Receives and triages incoming incident reports at their location; completes initial classification fields the associate could not (e.g. confirms shift, equipment involved); decides whether an incident needs to spawn an Issue for corrective action; manages local hazard register entries and observation follow-up. |
| EHS/Safety Officer | Owns regulatory classification and recordability determination; manages the risk assessment and hazard register program end-to-end (authoring, review cadence, residual-risk sign-off); owns regulatory reporting deadline compliance and submits regulatory reports; is the persona with authority to reclassify an incident's regulatory category after the fact; configures the jurisdiction taxonomy. |
| Executive | Views enterprise/regional safety KPIs (TRIR/LTIR-style, near-miss ratio, hazard-register health) in the Control Tower; is alerted on any missed regulatory reporting deadline as a governance-severity event. |

## 3. Incident Reporting

- **FR-EHS-001 — Incident classification taxonomy.** Every `SafetyIncident` is classified along two
  independent axes at creation: `incident_type` ∈ `INJURY`, `ILLNESS`, `NEAR_MISS`,
  `PROPERTY_DAMAGE`, and a **jurisdiction-specific regulatory category** resolved from a
  tenant-configured, per-jurisdiction `RegulatoryTaxonomy` table (not a hardcoded OSHA enum — see
  §13 Resolved Decisions). For a jurisdiction configured with an OSHA-style taxonomy, regulatory
  category values include `RECORDABLE`, `FIRST_AID_ONLY` (not recordable), `RESTRICTED_WORK`,
  `LOST_TIME`, `FATALITY`, `NOT_APPLICABLE` (near-miss/property-damage default). The taxonomy is
  resolved from the incident's `location_id` → `OrganizationNode` → configured jurisdiction code,
  not from tenant-level default alone, so a multi-country tenant applies the correct rule per site.
- **FR-EHS-002 — Reporter-facing report flow.** Any authenticated user with `safety:report` scope
  (default: all frontline roles) can start an incident report from the mobile app or web. Minimum
  fields to save a draft: `incident_type`, `location_id`, `occurred_at` (defaults to now,
  editable), free-text `description`. The remainder of the regulatory classification fields
  (body part, injury nature, days away, restricted/transferred duty, equipment involved) are
  presented as a guided multi-step form but are not required to submit an initial report —
  submission creates the record and starts the regulatory reporting clock (FR-EHS-060) even if
  classification is incomplete, since the reporting deadline is measured from when the employer
  *learns of* the case, not from when classification finishes.
- **FR-EHS-003 — Severity taxonomy, configurable per jurisdiction.** In addition to regulatory
  category, every incident receives an internal `severity` (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`,
  same enum values as `Issue.severity` for cross-module consistency) computed from a
  tenant-configured mapping table keyed by (`jurisdiction_code`, `incident_type`,
  `regulatory_category`) → `severity`, with a documented default mapping (e.g. `FATALITY` →
  `CRITICAL`, `LOST_TIME`/`RESTRICTED_WORK` → `HIGH`, `RECORDABLE` first-aid-adjacent → `MEDIUM`,
  `NEAR_MISS` → `LOW` unless a `near_miss_potential_severity` field is set `HIGH`/`CRITICAL` by the
  reporter, in which case the near-miss is escalated for trending purposes per FR-EHS-041).
- **FR-EHS-004 — Anonymous near-miss reporting.** Tenant policy may enable
  `anonymous_near_miss_reporting = true` per location or org-wide. When enabled, a `NEAR_MISS`
  report may be submitted with `reporter_user_id = NULL` and `is_anonymous = true`; the client
  device still transmits a `device_id`/session for abuse-rate-limiting purposes but that value is
  never persisted against the incident record nor exposed to any manager/EHS Officer UI or export.
  Anonymous reports cannot be used as the sole source for a `RECORDABLE`/injury classification
  (regulatory categories other than `NEAR_MISS` require an identified reporter or a manager/EHS
  Officer co-filing), enforced as a hard validation rule (§10).
- **FR-EHS-005 — Witness statement capture.** From an incident record, any user (including a
  non-witness manager facilitating capture on someone's behalf, e.g. offline) can add one or more
  `IncidentWitnessStatement` records: `witness_name` (free text, since a witness may not be a
  platform user), `witness_user_id` (nullable FK when the witness is an identified platform user),
  `statement_text`, optional recorded/uploaded statement `Evidence` reference (audio/video/signed
  document), `captured_by`, `captured_at`. Witness statements are append-only once submitted
  (edits create a new versioned statement referencing the prior one, never mutate in place) — same
  audit-defensibility rationale as Issues & CAPA's `issue_events` (P7, Historical truth).
- **FR-EHS-006 — Equipment/asset linkage.** An incident may reference an `Asset` (CMMS module,
  EPIC-16-equivalent) by ID when equipment was involved; the reference is by ID only, this module
  does not own asset lifecycle state (see §10 edge case: decommissioned asset reference).
- **FR-EHS-007 — PPE-violation-triggered incident/observation.** A Visual AI detection result
  (§12.2 result contract) with a `violations[]` entry of a tenant-configured PPE-related `type`
  (e.g. `missing_ppe`) above the tenant's action-confidence threshold can, per tenant policy, either
  (a) auto-create a `SafetyObservation` of `observation_type = NEGATIVE` with
  `source = VISUAL_AI`, `ai_result_ref` FK for traceability, requiring human confirmation before it
  counts toward trending (mirrors the Visual AI human-review-gate pattern, §12.2 "not automatically
  treated as irreversible compliance facts"), or (b) surface a **suggested** incident/near-miss
  draft for a manager to confirm and complete — it never auto-creates a confirmed `SafetyIncident`
  without a human submitting/confirming it, since a `SafetyIncident` carries regulatory
  consequences an unreviewed AI detection must not trigger unilaterally.

## 4. Root-Cause Linkage to Issues & CAPA

- **FR-EHS-010 — Spawning an Issue.** From an incident record (any regulatory category except a
  low-severity near-miss the EHS Officer marks "trend only, no CAPA needed"), a Store/Site Manager
  or EHS Officer with `safety:spawn_issue` scope can spawn a corrective-action `Issue` via the
  internal Issues & CAPA API, passing `category = SAFETY`, a seed `severity` copied from
  `SafetyIncident.severity`, and `source = SAFETY_INCIDENT` (a new `Issue.source` enum value this
  module introduces, parallel to `MANUAL`/`RULE_TRIGGERED`/`WORK_ESCALATION`/`AGENT` in
  `06-issues-capa`, carrying `source_ref = { safety_incident_id }`). Exactly one `Issue` may be
  spawned per `SafetyIncident` at a time (idempotency keyed on `safety_incident_id`); the created
  `issue_id` is stored back on `SafetyIncident.linked_issue_id`.
- **FR-EHS-011 — CAPA execution is not re-implemented here.** Once spawned, all root cause analysis,
  containment, corrective/preventive action, and effectiveness verification for that Issue proceed
  entirely inside Issues & CAPA per its own functional spec (`06-issues-capa/functional-spec.md`
  §6–§9). This module does not present RCA method pickers or CAPA step forms; the incident record's
  UI instead deep-links to the linked Issue's CAPA workspace.
- **FR-EHS-012 — Closing the loop.** This module subscribes to `issue.resolved` and (via CDC or a
  direct `issue_events` read on `CLOSED`) `Issue` closure for any Issue with `source =
  SAFETY_INCIDENT`. On `issue.resolved`, `SafetyIncident.corrective_action_status` transitions to
  `RESOLVED`; on the Issue reaching `CLOSED`, it transitions to `CLOSED`, which is one of the two
  gating conditions (together with regulatory reporting completion, FR-EHS-062) for the
  `SafetyIncident` lifecycle's own `CORRECTIVE_ACTION_LINKED -> CLOSED` transition (see
  architecture-diagram.md state diagram). If the linked Issue is `REOPENED` (e.g. a failed
  effectiveness verification), `SafetyIncident.corrective_action_status` reverts to
  `IN_PROGRESS` and, if the `SafetyIncident` had already closed, it is moved to its own `REOPENED`
  state (FR-EHS-013).
- **FR-EHS-013 — Reopening on regulatory audit finding.** Independent of the linked Issue's state, a
  `CLOSED` `SafetyIncident` can be reopened by an EHS Officer when a regulatory audit or inspection
  finds the classification, reporting, or corrective record deficient (`reopen_reason` ∈
  `AUDIT_FINDING`, `RECLASSIFICATION_REQUIRED`, `LINKED_ISSUE_REOPENED`, `OTHER`). This is a
  regulatory-record-integrity reopening, distinct from and in addition to the linked Issue's own
  reopening path, and is the reason `SafetyIncident` needs its own lifecycle rather than inheriting
  `Issue`'s directly (see architecture-diagram.md narration).

## 5. Risk Assessment Authoring

- **FR-EHS-020 — Risk matrix scoring.** A `RiskAssessment` scores one identified risk as
  `likelihood` × `severity`, each on a tenant-configurable N×N scale (default 5×5: likelihood
  `RARE`/`UNLIKELY`/`POSSIBLE`/`LIKELY`/`ALMOST_CERTAIN` = 1–5, severity
  `NEGLIGIBLE`/`MINOR`/`MODERATE`/`MAJOR`/`CATASTROPHIC` = 1–5), producing a `raw_risk_score =
  likelihood_value * severity_value` (range 1–25) and a `risk_band` (`LOW`/`MEDIUM`/`HIGH`/
  `CRITICAL`) from a tenant-configurable band-cutoff table (default: 1–4 `LOW`, 5–9 `MEDIUM`, 10–15
  `HIGH`, 16–25 `CRITICAL`). See LLD §Pseudocode for the scoring algorithm including tenant-scale
  overrides (e.g. a tenant using a 4×4 or 3×3 matrix instead of 5×5).
- **FR-EHS-021 — Controls and residual risk.** A `RiskAssessment` records `controls_in_place[]`
  (free text + control `type` ∈ `ELIMINATION`, `SUBSTITUTION`, `ENGINEERING`,
  `ADMINISTRATIVE`, `PPE` — the standard hierarchy-of-controls taxonomy, ordered most-to-least
  effective) and a `residual_likelihood`/`residual_severity` re-score assuming those controls are in
  place, producing `residual_risk_score`/`residual_risk_band` using the same matrix. A residual
  score higher than or equal to the raw score triggers a validation warning ("controls should
  reduce risk; confirm this is intentional") rather than a hard block, since a tenant may
  legitimately record a control that does not reduce a specific dimension.
- **FR-EHS-022 — Assessment linkage.** A `RiskAssessment` may reference a `location_id`, an
  `Asset` (by ID), a `HazardRegisterEntry` (when the assessment formalizes a previously logged
  hazard, §6), a `Template`/`TemplateVersion` (when authored via a Forms-platform-backed structured
  assessment template rather than the standalone EHS risk-matrix picker UI), and/or a
  `SafetyIncident` (when the assessment is a post-incident reassessment of a hazard that caused it).
- **FR-EHS-023 — Assessment review cycle.** Every `RiskAssessment` has a `next_review_due_at`
  (default: risk-band-dependent — `CRITICAL`/`HIGH` review annually, `MEDIUM` every 2 years, `LOW`
  every 3 years, all tenant-configurable) and an `assessor_user_id`/`approved_by_user_id` pair
  (segregation of duties for `HIGH`/`CRITICAL` bands, mirroring Issues & CAPA's verifier-≠-owner
  rule for high-severity effectiveness verification).

## 6. Hazard Register

- **FR-EHS-030 — Hazard register entry.** A `HazardRegisterEntry` records an identified hazard
  independent of any specific incident: `hazard_description`, `hazard_category` (tenant taxonomy,
  e.g. `CHEMICAL`, `ERGONOMIC`, `ELECTRICAL`, `MECHANICAL`, `SLIP_TRIP_FALL`, `FIRE`,
  `ENVIRONMENTAL`, `OTHER`), `location_id` and/or `asset_id`, `identified_by`, `identified_at`,
  current linked `RiskAssessment` (`current_risk_assessment_id`), `status` (`ACTIVE`,
  `CONTROLLED`, `RETIRED`), and `review_cadence_days` (defaults from the linked assessment's
  risk-band-driven cadence, FR-EHS-023, but independently overridable).
  A hazard register entry can exist without ever producing an incident (proactive identification)
  or can be created *from* an incident (reactive — the incident's post-mortem reveals a
  previously-unregistered hazard), tracked via `source_safety_incident_id` (nullable).
- **FR-EHS-031 — Review-due surfacing.** When `next_review_due_at` (derived from the linked
  `RiskAssessment.next_review_due_at`, or the entry's own override) is within a tenant-configured
  lookahead window (default 30 days), the entry surfaces a "review due" badge in the hazard register
  UI and a background sweep (LLD §Background Jobs) generates a review reminder notification to the
  entry's `owner_user_id` (defaults to `identified_by`, reassignable).
- **FR-EHS-032 — Retiring a hazard.** An entry moves to `RETIRED` only with a recorded
  `retirement_reason` and `retired_by`; a retired entry is not deleted (regulatory defensibility —
  a regulator or plaintiff's discovery request may ask "did you know about this hazard and when did
  you retire the mitigation"), and its historical `RiskAssessment` versions remain queryable.
- **FR-EHS-033 — Decommissioned-asset reference.** See §10 edge case; a hazard register entry
  referencing a since-decommissioned `Asset` is not auto-retired (the physical hazard at that
  location may still be relevant even if the specific asset instance is gone) but is flagged
  `asset_status_stale = true` for EHS Officer review, surfaced in the review-due sweep regardless of
  the normal cadence timer.

## 7. Safety Observation Program

- **FR-EHS-040 — Observation submission.** A `SafetyObservation` records a positive (safe behavior
  reinforcement) or negative (at-risk behavior/condition) observation via the `SAFETY_OBSERVATION`
  WorkItem type (this module is the functional owner of that WorkItem type's submission payload,
  per the addendum's ownership pattern). Fields: `observation_type` (`POSITIVE`, `NEGATIVE`),
  `category` (tenant taxonomy aligned with hazard categories, §6), `location_id`,
  `observed_user_id` (nullable — an observation may target a general condition, not a person),
  `description`, optional `Evidence` (photo), and for `NEGATIVE` observations an optional
  `immediate_action_taken` free-text field (distinct from a full CAPA immediate-correction step —
  this is a lightweight field, not a `CapaStep`).
- **FR-EHS-041 — Near-miss trending.** `SafetyObservation` records of category/severity aligned with
  near-miss trending and `SafetyIncident` records of `incident_type = NEAR_MISS` feed a shared
  trending computation (Analytics-owned rollup, this module supplies the fact rows): near-miss
  frequency by location/category/time-window, and a **leading-indicator ratio**
  (positive-observation-count : negative-observation-count : near-miss-count : recordable-incident
  -count) exposed to the Executive Control Tower alongside lagging TRIR/LTIR metrics.
- **FR-EHS-042 — Observation-to-hazard escalation.** A manager or EHS Officer reviewing a `NEGATIVE`
  observation can promote it to a `HazardRegisterEntry` (`source_observation_id` set) when it
  represents a systemic condition rather than a one-off behavior, or spawn a `SafetyIncident`
  directly if the observed condition already caused (undetected until now) harm.

## 8. Regulatory Reporting Deadlines and Retention

- **FR-EHS-050 — Deadline rule configuration.** Each tenant configures one or more
  `RegulatoryReportingRule` rows keyed by `jurisdiction_code`, each specifying: which
  `regulatory_category` values it applies to, the deadline window (`deadline_days`, measured from a
  configurable anchor — default `incident.occurred_at`, tenant-overridable to
  `employer_notified_at` to match jurisdictions whose clock starts at employer knowledge rather than
  event time), and the required `RegulatoryReport` type/form (e.g. an OSHA 300/301-style log entry,
  modeled generically as a `report_type` string plus a `report_payload` JSONB the tenant's
  jurisdiction pack defines the shape of — this module does not hardcode any single country's form
  layout).
- **FR-EHS-051 — Deadline computation and tracking.** On `SafetyIncident` creation (or
  reclassification, FR-EHS-052), the system resolves the applicable `RegulatoryReportingRule`(s) for
  the incident's jurisdiction + regulatory category and computes `reporting_deadline_at` per rule,
  creating a `RegulatoryReportDeadline` row per applicable rule. See LLD §Pseudocode for the
  resolution algorithm (an incident can be subject to more than one rule, e.g. an internal-company
  deadline and a statutory deadline, tracked as separate deadline rows with independent statuses).
- **FR-EHS-052 — Reclassification changes the regulatory category.** An EHS Officer may reclassify
  an incident's `regulatory_category` after initial submission (e.g. a case initially logged
  `FIRST_AID_ONLY` is later reclassified `RECORDABLE` when the injury proves more serious than first
  understood). Reclassification: (a) creates a new `RegulatoryClassificationHistory` row (append-
  only, previous classification preserved, never overwritten) with `reclassified_by`,
  `reclassified_at`, `reason`; (b) recomputes applicable `RegulatoryReportingRule`s and
  `RegulatoryReportDeadline` rows — a newly-applicable deadline's anchor remains the *original*
  `occurred_at`/`employer_notified_at`, not the reclassification date, since jurisdictions
  typically measure from the original event/knowledge date regardless of when classification was
  corrected (a deadline computed this way may already be in the past at the moment of
  reclassification — see §10 edge case, this is expected and surfaced, not suppressed).
- **FR-EHS-053 — Approaching-deadline alerting.** Each `RegulatoryReportDeadline` surfaces an
  escalating badge/notification ladder as it approaches: informational at a tenant-configured
  lead time (default 50% of window elapsed), warning (default 80% elapsed), critical (default 95%
  elapsed and unreported), and a distinct **missed-deadline** state once `deadline_at` passes with
  `status != SUBMITTED`. Because a missed regulatory deadline is a compliance/legal-exposure event,
  not an ordinary SLA miss, this tracking is explicitly a fail-loud path (NFR-EHS-001): it never
  silently degrades to "escalation didn't fire" the way a best-effort operational SLA might.
- **FR-EHS-054 — Regulatory report submission.** An EHS Officer files the `RegulatoryReport`
  (`report_type`, `report_payload`, optionally an exported/generated document `Evidence`
  reference) against one or more `RegulatoryReportDeadline` rows, setting `status = SUBMITTED`,
  `submitted_by`, `submitted_at`, and (for jurisdictions requiring it) an external
  `regulator_confirmation_ref`. A report submitted after `deadline_at` is still accepted (the
  platform never blocks a late filing) but the deadline row retains `late = true` permanently for
  audit purposes (§10 edge case).
- **FR-EHS-060 — Regulatory retention override.** `SafetyIncident` (and its `IncidentWitnessStatement`,
  `RegulatoryClassificationHistory`, `RegulatoryReport`, and linked `Evidence` records) are subject
  to a per-jurisdiction `retention_years` value from the same `RegulatoryTaxonomy`/jurisdiction
  configuration (default recommendation 5 years absent tenant/jurisdiction input, commonly longer
  for exposure-record categories), which **overrides** the tenant's platform-wide default retention
  policy (source-spec §21.4 Data Lifecycle) for these specific records whenever the regulatory value
  is longer. The retention override is a floor, never a ceiling — a tenant cannot configure a
  shorter EHS retention than its jurisdiction's statutory minimum through this module's settings
  (hard-validated against the jurisdiction pack's minimum).
- **FR-EHS-061 — Retention immutability.** Once a `SafetyIncident` record (and the linked records in
  FR-EHS-060) is within its regulatory retention window, no user role — including tenant admin —
  can delete or force-purge it through the standard UI/API; only a documented legal-hold-release /
  data-lifecycle administrative process (out of this module's UI scope, coordinated with Security &
  Compliance, EPIC-14) can do so, and only after the retention window lapses. This is enforced at
  the data-access layer (LLD §Concurrency/Immutability), not merely as an application-layer
  convention, given the legal-defensibility requirement (NFR-EHS-002).
- **FR-EHS-062 — Regulatory-closure gate.** A `SafetyIncident` cannot reach its terminal `CLOSED`
  lifecycle state (architecture-diagram.md state diagram) until every applicable
  `RegulatoryReportDeadline` for it is either `SUBMITTED` or explicitly marked
  `NOT_APPLICABLE`/`WAIVED` by an EHS Officer with a recorded reason — closing the corrective-action
  loop (FR-EHS-012) alone is not sufficient, since the regulatory obligation is independent of
  whether corrective action was needed at all (e.g. a `NEAR_MISS` may need no Issue but may still
  need to be logged in a voluntary internal near-miss report by policy).

## 9. UX States

- **FR-EHS-080 — Incident-report-in-progress with witness-add flow.** The mobile report screen shows
  a persistent draft state (`DRAFT` — see architecture-diagram.md; distinct from and prior to the
  `REPORTED` lifecycle state, which is entered on submit) with a progress indicator across the
  guided classification steps; a "Add witness" action is available from the moment the base report
  is submitted (post-`REPORTED`) without requiring the full classification to be complete first, so
  witness memory is captured while fresh. Each added witness shows inline (name, capture status:
  `RECORDED_TEXT`/`RECORDED_AUDIO`/`PENDING_SIGNATURE`).
- **FR-EHS-081 — Reporting-deadline-approaching badge.** Anywhere a `SafetyIncident` or its
  `RegulatoryReportDeadline` rows appear (incident detail, EHS Officer's deadline dashboard, Control
  Tower safety widget), a badge reflects the most urgent applicable deadline state per FR-EHS-053:
  neutral (>50% remaining), amber "due in N days" (warning threshold), red "URGENT — due in N
  hours/overdue" (critical/missed). The missed-deadline state additionally pins the incident to the
  top of the EHS Officer's queue regardless of other sort criteria.
- **FR-EHS-082 — Risk-matrix visual picker.** Risk assessment authoring presents an N×N grid
  (likelihood rows × severity columns) with each cell colored by its resulting `risk_band` (a
  standard green/yellow/orange/red heat-map convention); selecting a cell sets both `likelihood`
  and `severity` in one action, and the residual-risk re-score (FR-EHS-021) is shown as a second,
  smaller grid with the "before" cell ghosted for comparison.
- **FR-EHS-083 — Hazard-register-review-due state.** The hazard register list defaults to sorting by
  review urgency (overdue first, then soonest-due), with a distinct visual treatment for
  `asset_status_stale = true` entries (FR-EHS-033) so a stale-asset-reference review is
  distinguishable from an ordinary time-based review-due entry.
- **FR-EHS-084 — Anonymous reporting UX.** When `anonymous_near_miss_reporting` is enabled for a
  location, the incident-type picker shows a visible "Report anonymously" toggle only for
  `NEAR_MISS`; selecting it removes reporter-identifying fields from the rest of the flow and shows
  a persistent notice that anonymity is preserved end-to-end (no reporter field is ever rendered to
  any downstream viewer, per FR-EHS-004).
- **FR-EHS-085 — Empty/loading/offline/permission-denied states.** An incident report started
  offline (mobile) queues via the standard sync protocol (source-spec §19), idempotent on
  `client_uuid`, and is clearly labeled "queued — will submit when online" without blocking the
  associate from adding witness statements locally in the meantime. A user without `safety:read`
  scope at a location sees a scope-denied state (`403 SCOPE_DENIED`), never a 404, consistent with
  Issues & CAPA's FR-CAPA-085 pattern. The hazard register and deadline dashboard show a distinct
  "no hazards/deadlines match filter" zero-state versus "no hazards/deadlines exist at this scope."

## 10. Validation and Edge Cases

- **10.1 — Incident reported after the regulatory deadline window.** A `SafetyIncident` may be
  reported (associate files it) after its jurisdiction's reporting deadline has already elapsed
  relative to `occurred_at` (e.g. a delayed disclosure). The system still computes and creates the
  `RegulatoryReportDeadline` row with `deadline_at` in the past and immediately sets it to the
  missed-deadline state (FR-EHS-053) rather than silently back-dating or suppressing it — the gap
  between event and report is itself a compliance-relevant fact that must be visible, not hidden by
  clever date math.
- **10.2 — Incident later reclassified changing its regulatory category.** See FR-EHS-052. The
  system never mutates the original classification row; reclassification is additive history. If
  the newly-applicable deadline (anchored to the original `occurred_at`) is already past at the
  moment of reclassification, the resulting `RegulatoryReportDeadline` is created directly in the
  missed-deadline state with a `reason_created_late = RECLASSIFICATION` annotation, so downstream
  audit review can distinguish "we were late" from "we were late because we didn't know" without
  losing the original timeline.
- **10.3 — Hazard register entry referencing a since-decommissioned asset.** See FR-EHS-033. The
  entry is flagged, not auto-retired; a decommissioned asset does not imply the physical location
  hazard is gone (a machine is removed, but the electrical junction box it was wired into may still
  be a live hazard). Auto-retiring on asset decommission would be an unsafe default.
- **10.4 — Anonymous near-miss reporting.** See FR-EHS-004. A hard validation rule rejects any
  attempt to submit an anonymous report for a non-`NEAR_MISS` `incident_type`, and rejects any
  attempt to attach a `reporter_user_id` to a record already flagged `is_anonymous = true`
  (immutable once set — an anonymous report cannot later be "de-anonymized" even by the original
  reporter, to preserve the integrity of the anonymity guarantee that drove candid reporting in the
  first place).
- **10.5 — Duplicate incident reports for the same event.** Two associates present at the same
  event may each file a report. Unlike Issues & CAPA's non-blocking duplicate suggestion, this
  module surfaces a **blocking** confirmation step when a second report at the same `location_id`
  within a short window (default 4 hours) shares `incident_type` and overlapping free-text/keyword
  similarity: the reporter is asked "is this the same event as [existing incident]?" — answering yes
  attaches the second report as an additional witness statement on the first incident rather than
  creating a second `SafetyIncident` (regulatory reporting must not double-count one event as two
  recordable cases); answering no creates a distinct incident. This is stricter than Issues & CAPA
  because regulatory recordability counts are directly audited.
- **10.6 — Regulatory report filed late.** See FR-EHS-054. Filing is never blocked by lateness; the
  `late = true` flag is permanent even if later corrected/amended, since "was this ever late" is
  itself a fact regulators and auditors ask for.
- **10.7 — Retention immutability vs. legitimate correction.** A `SafetyIncident` inside its
  retention window still needs to support a legitimate reclassification or added witness statement
  (both are additive, non-destructive writes). Retention immutability (FR-EHS-061) blocks
  *deletion/purge*, not *append*; every mutable field on `SafetyIncident` itself (e.g.
  `regulatory_category`, `severity`) is versioned via the append-only
  `RegulatoryClassificationHistory`/`safety_incident_events` pattern (LLD §Schema) rather than
  hard-blocked from ever changing, so the record stays both correctable and non-erasable.
- **10.8 — Linked Issue closed before regulatory reporting completes, or vice versa.** See
  FR-EHS-062. Neither path implies the other is done; the `SafetyIncident` gates its own `CLOSED`
  state on both independently, which is precisely why this module cannot simply proxy `Issue.status`
  as its own lifecycle.

## 11. Non-Functional Requirements

- **NFR-EHS-001 — Regulatory-deadline tracking reliability (fail-loud).** Deadline computation,
  approaching-deadline alerting, and missed-deadline detection must never silently fail. Unlike a
  best-effort operational SLA (Issues & CAPA NFR-CAPA-002, which tolerates bounded timer drift), a
  failure in this module's deadline sweep is treated as a **P1 alerting/governance incident**: the
  background sweep job (LLD §Background Jobs) emits a heartbeat metric, and an absence of a
  successful sweep run within 2× its scheduled interval pages Platform Reliability, independent of
  and in addition to any per-incident missed-deadline notification. A deadline that silently never
  alerted anyone before lapsing is treated as a system defect, not an accepted risk.
- **NFR-EHS-002 — Retention immutability.** Enforced at the data-access layer via a Postgres rule/
  trigger rejecting `DELETE` (and destructive `UPDATE` of the classification/report/witness tables)
  against any `SafetyIncident`-linked row whose computed retention window has not lapsed, in
  addition to application-layer role checks — defense-in-depth consistent with the platform's
  tenant-isolation posture (source-spec §21.1: "a tenant identifier is not considered sufficient by
  itself"; analogously, an application-layer role check is not considered sufficient by itself for
  a legally-defensible retention guarantee).
- **NFR-EHS-003 — API performance.** Incident report creation and witness-statement submission meet
  the platform p95 < 300 ms target (source-spec §23) for the synchronous write; regulatory deadline
  resolution (which may evaluate multiple jurisdiction rules) is allowed a relaxed p95 < 500 ms,
  performed synchronously within the same creation transaction so a `SafetyIncident` is never
  created without at least a provisional deadline computation.
- **NFR-EHS-004 — Auditability.** Every classification change, deadline creation/status change,
  regulatory report submission, and lifecycle transition produces an immutable
  `safety_incident_events` row (mirroring Issues & CAPA's `issue_events` pattern), sufficient to
  reconstruct the full regulatory timeline of an incident without relying on mutable current-state
  tables — this is the primary artifact an external auditor or regulator inspection will request.
- **NFR-EHS-005 — Scale.** Sized against the platform's stated target (100k locations, source-spec
  §23); safety incident volume is materially lower than WorkItem/Submission volume but bursts occur
  (a multi-injury event, or a mass near-miss reporting campaign after a safety stand-down) and must
  not degrade the deadline-sweep job for unrelated tenants (see HLD §Scalability, tenant-fair
  scheduling, same pattern as Issues & CAPA's SLA-timer tenant fairness).

## 12. Dependencies

| Module | Dependency |
|---|---|
| Issues & CAPA (EPIC-07) | This module calls `create_issue`-equivalent internal API to spawn corrective-action Issues (`source = SAFETY_INCIDENT`); consumes `issue.resolved`/Issue `CLOSED` to close its own corrective-action feedback loop; never re-implements CAPA steps, RCA, or effectiveness verification. |
| Evidence (EPIC-06) | Incident photos, scene documentation, and witness statement recordings/documents are `Evidence` records referenced by ID; this module additionally stamps a regulatory retention override on Evidence linked to a `SafetyIncident`. |
| Learning (EPIC-09) | Hazard controls and corrective/preventive actions may link a `LearningAssignment`/`Course` (e.g. mandatory PPE refresher after a violation); consumes `training.completed` for context. |
| Analytics & Control Tower (EPIC-12/15) | Consumes `safety_incident.*`/`safety_observation.*`/`hazard_register.*` events and CDC feed for TRIR/LTIR-style KPIs, near-miss leading-indicator ratios, and hazard-register health rollups. |
| Visual AI (EPIC-10) | PPE-violation detections (§12.2 result contract) are a trigger source for `SafetyObservation`/suggested-incident drafts; this module never bypasses Visual AI's human-review gate for confirmed compliance facts. |
| Work Orchestration (EPIC-03) | Owns the `Incident / Observation` and `SAFETY_OBSERVATION` WorkItem types' scheduling/assignment/completion state machine; this module is the functional owner of what those WorkItem submissions mean. |
| Identity/Org (EPIC-01/02) | `UserScope`/`OrganizationNode` resolve `safety:report`/`safety:read`/`safety:spawn_issue` scope checks and the jurisdiction code applied per location. |
| Asset & Equipment Maintenance (CMMS, module 16) | `Asset` referenced by ID from incidents and hazard register entries; this module never writes asset lifecycle state and must tolerate a referenced asset later being decommissioned. |

## 13. Resolved Decisions

- **Which jurisdictions' regulatory taxonomies to ship first (FR-EHS-001, FR-EHS-050).** Firm
  decision: ship a **configurable `RegulatoryTaxonomy` + `RegulatoryReportingRule` table**
  (jurisdiction code, regulatory category set, severity mapping, deadline windows, retention years,
  report form shape), never a hardcoded single-country enum. At launch, seed exactly one
  fully-worked reference pack modeled on OSHA-style recordkeeping (injury/illness recordability,
  days-away/restricted/transferred categories, an annual-summary-style report) plus one empty/
  generic pack a tenant can configure for any other jurisdiction (UK RIDDOR-style, EU member-state
  variants, etc.) without an engineering change. Rationale: given the platform's stated multinational
  market shape (source-spec §1.2, "50 to 100,000+ locations"), a hardcoded OSHA-only taxonomy would
  not survive past a single US-only pilot tenant; a config-driven table costs no more to build and
  is the only option consistent with day-one multinational deployment.
- **`SafetyIncident.severity` reuses `Issue.severity` (FR-EHS-003).** Firm decision: yes, the same
  four-value enum (`LOW`/`MEDIUM`/`HIGH`/`CRITICAL`) is reused for cross-module UI/KPI consistency,
  kept as a separate field from the materially richer `regulatory_category`. Rationale: severity is
  the internal operational signal and regulatory category is the external compliance signal; a
  `FATALITY` and a `LOST_TIME` case are both plausibly `CRITICAL`/`HIGH` operationally while requiring
  different regulatory report types, so conflating the two fields was evaluated and rejected.
- **A `SafetyIncident` may exist with zero linked `Issue`s (FR-EHS-010).** Firm decision: yes — a
  near-miss logged purely for trending, or a first-aid-only case needing no corrective action, is
  valid with no spawned `Issue`. Spawning an Issue is an explicit manager/EHS Officer decision, never
  automatic, consistent with source-spec P5 (Policy before AI).
- **Hazard register `review_cadence_days` independence from `RiskAssessment` (FR-EHS-030).** Firm
  decision: an entry's `review_cadence_days` defaults from its linked `RiskAssessment.
  next_review_due_at`-derived cadence but supports an explicit per-entry override (already modeled).
  Full decoupling (independent cadence with no relationship to the underlying risk assessment) is
  rejected, because it would let a hazard register drift out of sync with the risk data that
  justifies its review urgency in the first place — the override covers every legitimate tenant case
  without introducing that drift risk.
- **Future Safety Agent auto-classification.** Firm decision: **out of scope for this baseline.** No
  agent may auto-classify or auto-reclassify an incident's `regulatory_category` at any autonomy
  level in this version. Rationale: regulatory misclassification carries legal exposure
  disproportionate to the automation benefit at this stage; if a Safety Agent is built in a future
  version, it enters at `L1 Recommend` only, with the EHS Officer as the mandatory human
  decision-maker for any classification-affecting action (source-spec P5, Policy before AI) — and
  because this is worker/incident-adjacent AI, any such future agent must also satisfy
  engineering-decisions.md §12's Article 14 human-oversight rule (mandatory human approval before any
  agent output materially affects a named individual's record).
