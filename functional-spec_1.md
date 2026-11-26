# Functional Specification — No-Code Forms & Workflow Builder

Module code: `FORM`. Source: source-spec.md §8 (No-Code Forms and Workflow Builder), §17.1 (Template /
TemplateVersion / Submission), §17.4 (Template State Machine), Appendix B (rule specification style),
Appendix E EPIC-04 (Form Platform). Owning service: **Form Service** (see high-level-design.md).

## 1. Purpose and Scope

### 1.1 Purpose
The No-Code Forms & Workflow Builder module lets Operations Admins translate an operating standard
(a checklist, audit, inspection, survey, or incident form) into a structured, versioned, logic-aware
digital form — without writing code — and lets Frontline Associates/Auditors fill those forms
reliably online or offline. It is the authoring and execution substrate for the platform's core loop
step "PLAN -> ... -> EXECUTE -> CAPTURE EVIDENCE": every `WorkItem` of type Task, Audit/Inspection,
Survey, Asset Inspection, Site Visit, or Incident/Observation that requires structured data capture
renders a `TemplateVersion` produced by this module, and every completed data capture becomes a
`Submission` owned by this module.

### 1.2 In Scope
- The form component library: field types, per-field configuration, and per-field validation rules
  (§2).
- Form logic: conditional visibility, mandatory conditions, conditional/cross-field validation,
  calculated fields, risk scoring, branching, dynamic repeating sections, reference-data lookups,
  cross-record validation (§3).
- The visual Builder UX for composing, arranging, and testing a template (§4).
- The `Template` / `TemplateVersion` lifecycle: `DRAFT -> REVIEW -> APPROVED -> PUBLISHED ->
  SUPERSEDED -> RETIRED`, immutability of published versions, schema hashing (§5).
- Impact Preview and simulated execution prior to production publish (§6).
- Template testing and staged/pilot rollout of a new version to a subset of locations (§7).
- Submission ingestion contract at the schema/validation level (the transport-level offline sync
  protocol itself is owned by the Mobile/Offline module; see Dependencies).
- Rendering behavior contracts required for offline/low-end-device form rendering (schema shape,
  caching contract) — the on-device rendering engine implementation is a Mobile/Offline concern.
- Composable card/layout blocks (image + text + button actions as one designed unit), a dialog/modal
  action primitive, and cross-template branching/forwarding — a button's outcome may hand off to an
  entirely different `Template`, not only a section within the same one (§12).

### 1.3 Out of Scope
- The generic Work Orchestration scheduling/assignment/SLA machinery that decides *when* and *to
  whom* a form-bearing `WorkItem` is delivered (owned by Work Orchestration module).
- The offline local database, sync queue, conflict UI, and resumable media upload protocol (owned by
  Mobile/Offline module) — this module defines the data contracts (schema payload, submission
  payload, idempotency key) that those mechanisms transport.
- Evidence storage, hashing, retention, and signed access for photo/video/audio/document/signature
  payloads (owned by Evidence module) — this module defines which fields are evidence-bearing and
  what metadata must accompany them.
- Issue/CAPA creation mechanics triggered by risk scoring (owned by Issues/CAPA module) — this module
  only emits the risk score and a `create_issue` intent from a rule; Issues/CAPA owns the resulting
  object lifecycle.
- Visual AI scoring of photo evidence (owned by Visual AI module) — a photo field may reference a
  Visual AI use case but the scoring pipeline itself is out of scope here.
- Natural-language-to-draft-workflow generation logic internal to the Workflow Builder Agent's model
  prompting (owned by Agent Runtime module) — this module defines the draft `Template`/`TemplateVersion`
  object shape the agent must produce and the human-approval gate the agent's output passes through
  before it can leave `DRAFT`.

## 2. Personas

| Persona | Job in this module | Primary surface |
|---|---|---|
| Operations Admin | Builds and maintains templates: adds fields, configures logic, defines risk scoring, tests, publishes, retires versions, reviews Impact Preview before a risky change. | Web Admin Console — Builder |
| Enterprise Admin | Approves template publication where enterprise policy requires a review gate; configures which template categories require approval before `PUBLISHED`. | Web Admin Console — Builder governance settings |
| Frontline Associate / Auditor | Fills out published templates as part of assigned `WorkItem`s, online or offline; sees only the runtime-rendered form, never the builder. | Mobile App |
| Regional/District Manager | Reviews submissions and reporting rollups derived from template data (read-only relative to this module); may be notified by Impact Preview of affected locations in their territory. | Manager Control Tower (consumer, not owner) |
| AI Agent — Workflow Builder Agent | Converts a natural-language request ("add a refrigeration check for the deli case") into a candidate `TemplateVersion` in `DRAFT`, with rationale and confidence, for a human Operations Admin to review, edit, and advance through the lifecycle. Never publishes directly (autonomy L1 Recommend per source-spec §13.1). | Builder (agent-authored draft surfaced in the same UI, labeled as agent-generated) |

## 3. Form Components

Each component below is a **field type**. Every field, regardless of type, has the common attribute
set in §3.0; type-specific attributes and validation follow.

### 3.0 Common Field Attributes (all types)

| Attribute | Type | Notes |
|---|---|---|
| `field_id` | string (stable, immutable once referenced by a published version) | Used as the key in submission payloads, calculated-field formulas, and conditional-logic expressions. Never reused for a semantically different field. |
| `label` | localized string | Shown to the filler. |
| `help_text` | localized string, optional | |
| `field_type` | enum (§3.1–§3.14) | Immutable after first publish reference (FR-FORM-041). |
| `required` | boolean or conditional rule | See §3.15 (mandatory conditions). |
| `visible` | boolean or conditional rule | See §4 (conditional visibility). |
| `default_value` | typed, optional | |
| `evidence_bearing` | boolean | True for photo/video/audio/document/signature/GPS/barcode types; drives Evidence-module linkage. |
| `pii_class` | enum: `NONE`, `PERSONAL`, `SENSITIVE` | Drives AI redaction eligibility per source-spec §21.3. |
| `order_index` | integer | Position within section/repeating group. |
| `section_id` | string | Grouping for layout and conditional branching. |

### FR-FORM-001 — Short/Long Text
- Sub-types: `TEXT_SHORT` (single line, max 255 chars default, configurable), `TEXT_LONG` (multi-line,
  max 10,000 chars default).
- Config: min/max length, regex pattern (e.g., asset tag format), input mask, placeholder,
  autocapitalize/keyboard hint (mobile only).
- Validation: length bounds; pattern match with a configurable human-readable error message per
  pattern; profanity/PII pattern flags are advisory only (do not block submission unless the tenant
  configures a hard validation rule).

### FR-FORM-002 — Number, Currency, Percentage
- Sub-types: `NUMBER` (integer/decimal, configurable precision), `CURRENCY` (decimal(19,4) internal
  storage, tenant default currency + optional per-field currency override), `PERCENTAGE` (stored as
  decimal 0–1 or 0–100, configurable, rendered with `%`).
- Config: min, max, step, decimal places, unit label (e.g., "°C", "kg").
- Validation: range bounds (may be conditional — see cross-field validation); non-numeric input
  rejected client-side before it reaches a rule evaluation.

### FR-FORM-003 — Single/Multi-Select
- Sub-types: `SELECT_SINGLE`, `SELECT_MULTI`.
- Options source: static list (versioned with the template) or reference-data lookup (§3.13).
- Config: min/max selections (multi), "Other, please specify" free-text escape option, option-level
  metadata (e.g., a risk weight per option consumed by risk scoring, §4.5).
- Validation: selected value(s) must exist in the option set resolved at fill-time (static list, or
  the lookup snapshot — see FR-FORM-046 for lookup staleness handling).

### FR-FORM-004 — Date/Time
- Sub-types: `DATE`, `TIME`, `DATETIME`.
- Config: min/max date (absolute or relative, e.g., "not before today", "not more than 24h in the
  past"), timezone handling (`DEVICE_LOCAL` default, or `LOCATION_TIMEZONE` for cross-location
  comparability), display format (locale-driven).
- Validation: range bounds; server re-validates relative bounds using **server time**, not client
  time (client clock is not trusted per source-spec §19.4).

### FR-FORM-005 — Yes/No
- `field_type = YES_NO`. Renders as a boolean toggle/segmented control. Supports a third
  `NOT_APPLICABLE` state only if explicitly enabled per field (`allow_na: boolean`).
- Commonly the trigger of conditional visibility/mandatory rules on other fields (e.g., "If No, `
  corrective_action` becomes required").

### FR-FORM-006 — Rating
- `field_type = RATING`. Config: scale (`1-3`, `1-5`, `1-10`, or custom labeled scale, e.g.,
  Excellent/Good/Fair/Poor), icon style (star/number/emoji — display only, no functional effect).
- Validation: value must be one of the configured scale points.

### FR-FORM-007 — Matrix
- `field_type = MATRIX`. A grid of sub-questions (rows) × a shared answer scale (columns), e.g., 10
  safety-checklist rows all answered on a Yes/No/N-A column set.
- Config: row definitions (each row has its own `field_id` for logic/formula reference), shared
  column/option set, optional per-row weight for scoring.
- Validation: every row independently required/optional per its own mandatory-condition; matrix as a
  whole may carry a cross-field validation rule (e.g., "at least 8 of 10 rows must be Yes").

### FR-FORM-008 — Barcode / QR
- `field_type = BARCODE`. Captures a scanned code string plus scan metadata (symbology, scan
  timestamp, device).
- Config: expected symbology allowlist, optional format regex (e.g., must match a 12-digit UPC),
  optional lookup validation against `Asset`/`Product` reference data (§3.13) — "scanned code must
  resolve to a known asset in this location."
- Validation: symbology/format match; lookup resolution success/failure is surfaced to the filler
  with a retry affordance, not a silent failure.
- `evidence_bearing = true`; the raw scan is retained as evidence metadata even if a subsequent
  lookup fails, to support audit of a bad/damaged label.

### FR-FORM-009 — GPS
- `field_type = GPS`. Captures device lat/long, accuracy radius, and capture timestamp at the moment
  the field is answered (typically auto-captured, not manually entered).
- Config: `capture_mode` = `AUTO_ON_FIELD_FOCUS` (default) or `MANUAL_BUTTON`; optional geofence
  validation against the assigned `WorkItem`'s `location_id` (radius in meters, tenant-configurable
  default 150m).
- Validation: if geofence validation is enabled and the capture falls outside radius, the submission
  is not blocked by default (configurable to `WARN` vs `BLOCK`) but the out-of-geofence flag is
  persisted and surfaced to reviewers — this supports the "verified completion" distinction in
  source-spec P4 without assuming GPS spoofing is impossible to rule out client-side.
- `evidence_bearing = true`.

### FR-FORM-010 — Signature
- `field_type = SIGNATURE`. Captures a vector or rasterized signature image plus signer name,
  timestamp, and device.
- Config: require typed signer name confirmation, optional dual-signature (filler + witness).
- `evidence_bearing = true`; stored via Evidence module; the Submission stores only the evidence
  reference and hash, never raw signature bytes.

### FR-FORM-011 — Photo / Video / Audio
- Sub-types: `PHOTO`, `VIDEO`, `AUDIO`.
- Config: min/max count (e.g., "at least 2 photos"), max duration (video/audio), max resolution/size
  (mobile-side compression target), required capture mode (`CAMERA_ONLY` — disallow gallery picker —
  vs `CAMERA_OR_LIBRARY`), optional linked Visual AI use case (advisory annotation only; scoring is
  out of scope of this module).
- Validation: count bounds; capture-mode enforcement (client blocks gallery selection when
  `CAMERA_ONLY`, server does not re-verify capture provenance beyond what Evidence-module metadata
  attests).
- `evidence_bearing = true` always.

### FR-FORM-012 — Document Upload
- `field_type = DOCUMENT`. Config: allowed MIME types/extensions, max file size (default 25MB, tenant
  configurable), max count.
- `evidence_bearing = true`; virus/malware scanning and storage handled by Evidence module — this
  module only defines the accepted-type policy and blocks submission client-side/server-side on
  disallowed types.

### FR-FORM-013 — Person / Location / Asset / Product Lookup
- `field_type = LOOKUP`, with `lookup_target` = `PERSON` | `LOCATION` | `ASSET` | `PRODUCT` |
  `CUSTOM_REFERENCE_SET`.
- Config: scope filter (e.g., "locations within the filler's org subtree only" — enforced via the
  same `UserScope` mechanism as source-spec §6.1), search-as-you-type vs. fixed list, allow
  multi-select, allow "not in system, enter manually" escape hatch (captured as free text with a
  `lookup_resolved = false` flag).
- Validation: selected reference must exist in the reference dataset **as of submission time**;
  reference-data snapshots used offline are timestamped so a stale-lookup scenario (§9) is
  detectable and surfaced.

### FR-FORM-014 — Repeating Groups
- `field_type = REPEATING_GROUP`. Contains a nested set of fields (any type, including nested
  repeating groups up to a configurable max nesting depth — default 2) that the filler can add/remove
  instances of at runtime (e.g., "add a line item per delivery pallet").
- Config: min/max instance count, per-instance field set, whether the group itself can be
  conditionally hidden.
- Each instance gets a stable `instance_id` (client-generated UUID) so calculated fields and
  cross-record validation can reference "sum across instances" deterministically regardless of
  add/remove/reorder during fill.
- Validation: instance count bounds; each instance's inner fields validate independently plus any
  group-level aggregate validation (e.g., "sum of `line_total` across instances must equal
  `invoice_total`" — a cross-field validation rule scoped to the repeating group, §4.4).

### FR-FORM-015 — Calculated Fields
- `field_type = CALCULATED`. Not directly editable by the filler; value is derived from a formula
  referencing other `field_id`s (including across repeating-group instances via aggregate functions).
- Config: `formula` (expression string, see §4.3 grammar), `output_type` (number/currency/text/
  boolean), `recalculate_on` = `ANY_DEPENDENCY_CHANGE` (default) or explicit field list.
- Validation: formula is validated for referential integrity (all referenced `field_id`s exist and
  are upstream in evaluation order) at **build time**, not first at fill time (FR-FORM-024).
- Calculated fields are always visible as read-only display unless explicitly hidden by the admin;
  they cannot themselves be the target of a mandatory-condition (a derived value cannot be "required"
  independent of its inputs).

## 4. Form Logic

### FR-FORM-020 — Conditional Visibility
A field or section's `visible` attribute may be a boolean expression over other fields' current
values in the same submission (same template instance), e.g.
`visible: {"field":"temperature","op":">","value":8}`. Hidden fields are excluded from validation and
from the submitted payload's "answered" set (their `default_value`, if any, is still recorded as
`inherited_default`, distinct from a filler-entered answer, so downstream analytics can distinguish
"not shown" from "shown and left blank").

### FR-FORM-021 — Mandatory Conditions
`required` may itself be a conditional expression (not just `true`/`false`), e.g. "require
`corrective_action` when `temperature.value > 8`" (this is the exact pattern in source-spec Appendix
B). A field that becomes required due to a rule and is currently visible must be answered before
submission; if the triggering condition later becomes false during the same fill session, the
previously entered value for the now-optional field is retained (not cleared) unless the admin
explicitly configures `clear_on_hide: true` for that field.

### FR-FORM-022 — Conditional & Cross-Field Validation
A field may carry one or more validation rules beyond its type's intrinsic bounds, expressed as a
condition + a validation predicate + an error message, and may reference **other fields** (cross-
field), e.g. "if `unit = kg`, then `weight` must be ≤ 50; if `unit = lb`, then `weight` must be ≤
110." Cross-field rules are evaluated after all referenced fields have a value or have been
determined not-required; a rule referencing a currently-hidden field is skipped (never blocks
submission on a field the filler cannot see).

### FR-FORM-023 — Branching
A `SELECT_SINGLE`/`YES_NO`/`RATING` field may declare a `branch_map`: a mapping from answer value to
the next visible section (skipping sections not on the resolved branch). Branching is implemented as
sugar over conditional visibility (§4.1) at the section level — the compiled schema (low-level-design.md
§7) always resolves branching down to per-field `visible` expressions so the evaluation engine has a
single code path.

This is same-template, section-level branching only. For a composable card/button layout, a dialog
raised from a button, or a hand-off to a genuinely different `Template`, see §12 (FR-FORM-080 through
FR-FORM-087), which generalizes "answer value -> next thing" into "answer/button -> action," of which
`branch_map` navigation is one action among several.

### FR-FORM-024 — Calculated Values (Formula Grammar)
Formulas use a restricted expression grammar (no arbitrary code execution): arithmetic operators
(`+ - * /`), comparison operators, boolean operators (`AND OR NOT`), a fixed function library
(`SUM`, `AVG`, `MIN`, `MAX`, `COUNT`, `ROUND`, `IF`, `COALESCE`), and aggregate functions scoped to a
repeating group (`SUM(group.field)`). The Builder validates at save time that:
1. every referenced `field_id` exists in the template,
2. the dependency graph is acyclic (FR-FORM-042),
3. referenced fields are evaluated in a valid topological order before the calculated field,
4. type compatibility holds (e.g., `SUM` over a text field is rejected).

### FR-FORM-025 — Risk Scoring
A template may declare one or more named risk scores, each composed of weighted contributions from
field values/ranges (mirroring source-spec Appendix B's `TEMP-001` pattern generalized to a
configurable rule set rather than one hardcoded rule):
```
risk_scores:
  - score_id: cold_chain_risk
    inputs:
      - field: temperature
        bands: [{max: 8, points: 0}, {max: 12, points: 40}, {max: null, points: 100}]
      - field: door_seal_intact
        bands: [{value: true, points: 0}, {value: false, points: 30}]
    thresholds:
      - {min: 0, max: 39, level: LOW}
      - {min: 40, max: 69, level: MEDIUM}
      - {min: 70, max: 100, level: HIGH}
    actions_by_level:
      HIGH: [{require_field: corrective_action}, {create_issue: true}, {notify_role: store_manager}]
      MEDIUM: [{require_field: corrective_action}]
```
The resolved `risk_level` and numeric score are persisted on the `Submission` and are available to
downstream rules (mandatory-condition triggers), to the emitted `submission.created` event payload,
and to Issue/CAPA and Work Orchestration as trigger inputs — this module computes and exposes the
score; it does not itself create the `Issue` object (see §1.3 Out of Scope).

### FR-FORM-026 — Dynamic Repeating Sections
Equivalent to FR-FORM-014 at the section level: an entire section (not just a field group) can repeat
based on a driving field (e.g., "one Delivery Inspection section per pallet scanned"), with the
repeat count itself either filler-controlled (add/remove) or driven by a prior field's value (e.g., a
number field `pallet_count` pre-seeds N instances, still filler-adjustable unless
`allow_count_override: false`).

### FR-FORM-027 — Lookups Against Reference Data
Covers the resolution mechanics behind FR-FORM-003 (option lists) and FR-FORM-013 (lookup fields).
Reference datasets (`Location`, `Asset`, `Product`, `Person`, tenant-defined custom lists) are
versioned snapshots distributed to devices for offline resolution (see Dependencies, and Mobile/
Offline module); a lookup field's compiled schema fragment carries a `reference_set_version` so a
device can detect it is resolving against a stale snapshot (FR-FORM-046).

### FR-FORM-028 — Cross-Record Validation
A validation rule may need to compare the current submission against **other records**, not just
other fields in the same submission — e.g., "flag if this month's shrinkage submission for this
location exceeds 150% of the trailing-3-month average," or "block a second `opening_checklist`
submission for the same location on the same calendar day" (duplicate-prevention). Because this
requires server-side data the device cannot always have fully cached, cross-record rules are
evaluated in two tiers:
- **Tier 1 (advisory, client-side where cacheable):** rules whose comparison data is part of the
  bounded offline dataset (e.g., "this location's last 3 submissions of this template," cached
  alongside the template) run on-device and warn the filler immediately.
- **Tier 2 (authoritative, server-side at ingestion):** all cross-record rules are re-evaluated by
  the Form Service at submission ingestion regardless of Tier-1 outcome; a Tier-2 failure sets the
  submission's `validation_status = FAILED_CROSS_RECORD` and routes it to a review queue rather than
  silently accepting or rejecting it (an offline-captured submission must never be discarded — see
  P4/P9 in source-spec §4).

## 5. Visual Builder UX

### FR-FORM-030 — Canvas and Palette
The Builder presents a drag-and-drop canvas (sections containing fields, in author-defined order) and
a component palette organized by the categories in §3. Fields can be reordered by drag, cloned,
grouped into a new section, and moved into/out of a repeating group.

### FR-FORM-031 — Field Configuration Panel
Selecting a field opens a side panel with tabs: **General** (label, help text, required), **Options**
(type-specific config from §3), **Logic** (visibility/mandatory/validation rule builder — a structured
condition editor, not free-text code, to keep authoring no-code; an "Advanced" toggle exposes the raw
JSON rule for power users/agent-authored drafts), **Evidence** (for evidence-bearing types).

### FR-FORM-032 — Logic Rule Builder
Conditions are authored as `IF <field> <operator> <value/field>` rows combinable with `AND`/`OR`,
rendered as a readable sentence ("Show this field IF Temperature > 8"). The rule builder only offers
operators valid for the referenced field's type (no ">" on a text field) and only lists fields that
are (a) already placed earlier in evaluation order or (b) in the same section for same-section
cross-field rules, preventing most authoring-time logic errors before they reach validation (§9.2).

### FR-FORM-033 — Preview / Test Mode
From any point in `DRAFT` or `REVIEW`, an admin can enter **Test Mode**: a live, fully-interactive
render of the current draft schema (same rendering engine contract used by the mobile client) inside
the web console, seeded with either blank inputs or a saved "test persona" input set, without
creating a real `Submission` (submissions from Test Mode are flagged `is_test = true` and excluded
from all reporting/analytics and from cross-record rule comparisons).

### FR-FORM-034 — Autosave and Draft History
The Builder autosaves the in-progress `DRAFT` `TemplateVersion` on every meaningful change (debounced,
≤5s), maintaining a linear undo history for the current editing session (last 50 operations,
in-memory + periodically flushed) distinct from the formal version history in §6. Autosave conflicts
are handled per §11 (concurrent editors).

## 6. Template / TemplateVersion Lifecycle, Versioning, Immutability

### FR-FORM-040 — Lifecycle States
A `TemplateVersion` moves through exactly the states in source-spec §17.4 /conventions.md:
`DRAFT -> REVIEW -> APPROVED -> PUBLISHED -> SUPERSEDED -> RETIRED`. (Note: source-spec §8.3 also
describes a product-facing `DRAFT -> TEST -> PILOT -> PUBLISHED -> SUPERSEDED -> RETIRED` framing for
the Workflow Builder narrative; this is reconciled as follows — `TEST` corresponds to Test Mode
activity that can occur at any time while the version is in `DRAFT`/`REVIEW` and is not itself a
persisted state, and `PILOT` is modeled as a **scoped rollout of a `PUBLISHED` version** to a subset
of `OrganizationNode`s, not a separate lifecycle state — see FR-FORM-045. The canonical persisted enum
is the five-plus-one state machine above, matching conventions.md exactly.)

| State | Meaning | Who can transition it | Mutable? |
|---|---|---|---|
| `DRAFT` | Being authored; not runnable by any real `WorkItem`. | Any admin with Builder write scope | Fully mutable |
| `REVIEW` | Submitted for approval; frozen from further silent edits — any edit reopens `REVIEW` or returns to `DRAFT` depending on tenant policy. | Reviewer/Approver role | Read-only to non-approvers; approver can request changes (→ `DRAFT`) or approve (→ `APPROVED`) |
| `APPROVED` | Passed review; awaiting publish action (schema hash computed, Impact Preview run). | Operations Admin / Enterprise Admin | Immutable content; publish is the only forward action |
| `PUBLISHED` | Live; assignable to `WorkItem`s and renderable by devices. | System (on publish action) | **Fully immutable** — schema, rules, scoring, and metadata are frozen (FR-FORM-041) |
| `SUPERSEDED` | A newer version of the same `Template` has been published; this version no longer accepts new `WorkItem` assignments but existing in-flight `WorkItem`s and historical `Submission`s continue to reference it. | System (automatic on next publish) | Immutable |
| `RETIRED` | Explicitly withdrawn by an admin; no new assignment, and existing in-flight `WorkItem`s bound to it are flagged for admin reassignment to the current published version (FR-FORM-044). | Operations Admin | Immutable |

### FR-FORM-041 — Immutability of Published Versions
Once a `TemplateVersion` transitions to `PUBLISHED`, none of the following may change: field
definitions (type, `field_id`, options, validation), logic rules, calculated-field formulas, risk
scoring configuration, or the computed `schema_hash`. Any change requires authoring a **new**
`TemplateVersion` (same `template_id`, incremented `version`) that goes through `DRAFT -> ... ->
PUBLISHED` again. This guarantees the QA acceptance test in source-spec §24.1 ("Historical submission
still renders exactly against the original template version after later versions publish").

### FR-FORM-042 — Schema Hash Determinism
`schema_hash` is a deterministic content hash (SHA-256) computed over a canonicalized serialization of
the compiled schema (field definitions + logic rules + scoring config), excluding purely cosmetic
metadata (help text wording changes, label localization additions do **not** change functional
behavior but **do** change the hash if they are part of the canonical payload — Decision needed,
default below). Determinism requirements: stable key ordering (canonical JSON per RFC 8785-style
canonicalization), no timestamps or randomly generated IDs inside the hashed payload other than
stable `field_id`s, and hash recomputed and compared server-side at publish time as a tamper/drift
check against what the Builder last autosaved.
- **Decision needed:** whether label/help-text-only edits require a new version. Default: **yes** —
  a `TemplateVersion` is the unit of "what a filler saw and answered," and reproducing the exact
  historical rendering (source-spec P7 Historical Truth) requires the label text to be part of the
  immutable, hashed snapshot. Purely non-user-facing metadata (internal admin notes, tags) is
  excluded from the hash.

### FR-FORM-043 — Publish Action
Publishing requires: (1) the version is in `APPROVED` state, (2) Impact Preview has been generated for
this exact `schema_hash` (re-run if any change occurred after the last preview), (3) if the tenant
policy requires dual control, a distinct approver from the author. On success: `schema_hash` is
finalized, `published_at`/`published_by` are stamped, the version moves to `PUBLISHED`, any prior
`PUBLISHED` version of the same `template_id` moves to `SUPERSEDED`, and a `template.published` event
is emitted (high-level-design.md §5).

### FR-FORM-044 — Effect on In-Flight Submissions and WorkItems on Republish
- A `Submission` already `IN_PROGRESS` (started but not yet submitted) against the now-superseded
  version **continues on that version** through to completion; the client does not hot-swap schema
  mid-fill. This is a hard rule: mid-fill schema changes would corrupt calculated-field dependency
  state and risk-scoring context.
- A `WorkItem` that has not yet been started, and is unassigned-schedule-only (e.g., a recurring
  schedule that generates a new instance per occurrence), picks up the newly `PUBLISHED` version on
  its **next generated occurrence**; already-generated, not-yet-started occurrences keep the version
  that was current when they were generated, unless the admin explicitly force-migrates them (an
  admin action, audited, available only while the `WorkItem` is still `AVAILABLE`, not
  `IN_PROGRESS`).
- `RETIRED` (FR-FORM-040 row) additionally flags any still-open `WorkItem`s bound to it for admin
  reassignment; the system does not silently delete or auto-migrate an in-progress user's work.

### FR-FORM-045 — Template Testing and Pilot Rollout
- **Test**: available at any pre-publish state via Test Mode (§4, FR-FORM-033); zero production
  effect.
- **Pilot**: after `PUBLISHED`, an admin may scope the version's *effective assignability* to a
  subset of `OrganizationNode`s (a rollout scope list) for a defined pilot window, while the
  previously published version remains active for all other nodes. This is implemented as a rollout
  policy attached to the `Template` (not a distinct `TemplateVersion` state) so that the immutability
  guarantee in FR-FORM-041 is never entangled with rollout percentage. Expanding the pilot scope
  (5% → 25% → 50% → 100%, per source-spec §24.3 Release Path) never requires a new `TemplateVersion`
  — only a rollout-scope update, which is itself audited.
- Pilot metrics (completion rate, validation failure rate, average fill time, risk-score
  distribution) are surfaced back to the admin to decide whether to widen or roll back the rollout
  scope (rollback = revert rollout scope to the prior version's nodes; the pilot `TemplateVersion`
  itself is untouched — it simply becomes assignable to fewer/no nodes).

## 6. Impact Preview and Simulated Execution

### FR-FORM-050 — Impact Preview Contents
Before allowing a `PUBLISHED` transition, the Builder must display, computed against the *current
production state* at click-time:
- **Affected locations**: count and list (paginated) of `OrganizationNode`s where the current
  `PUBLISHED`/pilot version of this `Template` is assignable today.
- **Affected work items**: count of open (`AVAILABLE`/`SCHEDULED`/`IN_PROGRESS`) `WorkItem`s currently
  bound to the prior version, broken down by state, since these will either finish on the old version
  or need admin migration (FR-FORM-044).
- **Affected users**: distinct count of users who have this template assigned to them directly or via
  role/location scope.
- **Active schedules**: recurring `WorkItem` schedules that will generate their *next* occurrence
  against the new version.
- **Downstream workflows**: any Workflow Engine rules, Issue-creation triggers, or Analytics
  dimensions keyed to `field_id`s that were removed or retyped in this version (a structural diff
  against the prior `PUBLISHED` version — see FR-FORM-052 field diffing).
- **Reporting dimensions**: named report/dashboard definitions that reference a `field_id` or
  `risk_score.score_id` from this template, flagged if the new version removed/retyped that
  reference.

### FR-FORM-051 — Simulated Execution
The admin can run the new version against a set of **representative inputs** before production
publish:
- Inputs are either hand-entered test cases (à la Test Mode, §4, FR-FORM-033) or sampled from the
  **last N real submissions** of the prior version (default N=20, de-identified per tenant PII policy
  if `pii_class != NONE` fields are involved), replayed through the new version's compiled logic to
  show: which fields would now be visible/hidden/required differently, which previously-passing
  submissions would now fail validation, what the new risk score/level would resolve to, and whether
  any calculated-field formula errors (e.g., a removed dependency) would occur.
- Simulated execution runs in an isolated evaluation sandbox — it never creates real `Submission`,
  `Issue`, `WorkItem`, or notification side effects; any rule action of type `create_issue`,
  `notify_role`, etc. is captured as a "would trigger: X" annotation only.
- Simulation results are attached to the Impact Preview report and must be re-run (stale-flagged) if
  the draft changes after the last simulation run.

### FR-FORM-052 — Structural Diff Against Prior Version
The Impact Preview computes a field-level diff between the new `DRAFT`/`APPROVED` version and the
current `PUBLISHED` version: `ADDED`, `REMOVED`, `RETYPED`, `RENAMED` (detected via a stable internal
`field_id` continuity marker distinct from the display label — see FR-FORM-041/§9 schema migration),
`OPTIONS_CHANGED`, `LOGIC_CHANGED`. `REMOVED` and `RETYPED` fields are surfaced as **high-severity**
warnings when any downstream workflow/report/risk-score references them (FR-FORM-050's downstream
check); the admin must explicitly acknowledge each high-severity warning before the Publish action is
enabled.

## 7. UX States

### FR-FORM-060 — Builder Empty State
A newly created `Template` with no fields shows a guided empty state: quick-start options (start
blank, start from a category-specific starter template if the tenant's library has one, or "describe
what you need" which routes to the Workflow Builder Agent). The empty state never blocks manual field
addition.

### FR-FORM-061 — Broken Logic Warnings
The Builder surfaces logic problems inline, at the point of authoring, not only at publish time:
- A rule referencing a deleted/renamed field is shown as a red badge on that field's Logic tab and
  listed in a persistent "Issues" panel with a jump-to-field action.
- A circular calculated-field dependency (FR-FORM-024 point 2) is detected at save time and rejected
  with the specific cycle path shown (e.g., `total -> tax -> total`); the offending edit is not saved
  server-side, but the client keeps the attempted edit in the local undo buffer so the admin does not
  lose their in-progress typing.
- An unreachable field (visibility condition that can never evaluate true given other rules'
  constraints) is flagged as a **warning**, not an error — full satisfiability analysis is not
  guaranteed (see §9.3), so this is a best-effort static check, not a hard block.

### FR-FORM-062 — Large-Form Performance on Low-End Devices
Templates are budgeted: the Builder warns (non-blocking, at 150 fields or 30 repeating-group nested
instances-worth of static definition — tenant-tunable) that the rendering engine's target of a
sub-200ms per-keystroke re-evaluation on a reference low-end Android device (source-spec §23 mobile
performance intent) may not be met. Mitigations exposed to the admin: paginating the template into
explicit multi-page sections (each page's fields evaluate independently unless a rule spans pages, in
which case the dependency is flagged), and marking rarely-relevant sections as `lazy_evaluate: true`
so their conditional-visibility rules only run when the section is scrolled into view.

### FR-FORM-063 — Offline Form Rendering from Cached Schema
On the mobile client, a `PUBLISHED` `TemplateVersion` is rendered purely from the locally cached
compiled schema (no network call to render or evaluate logic). If the cached schema is missing (never
synced) or its `schema_hash` fails local integrity verification (see low-level-design.md §11), the
form does not render a partial/corrupt form; it shows a "this form needs to sync before it can be
opened" state with a manual retry, and the underlying `WorkItem` remains `AVAILABLE` (not silently
skipped). If a lookup reference-data snapshot (FR-FORM-027) referenced by the schema is stale beyond
its configured TTL, lookup fields still render (never block the whole form) but show a "may be
out of date, will re-validate on sync" indicator on that field only.

### FR-FORM-064 — Permission-Denied State
An Associate/Auditor without scope for a `Template`'s owning `OrganizationNode` never sees it in the
Builder or in template pickers; a direct-link attempt returns a scoped 403/empty state, not a
schema-content leak, per source-spec §21.1.

## 8. Validation and Edge Cases

### FR-FORM-070 — Schema Migration When a Field Is Removed or Renamed on a New Version
- **Removal**: historical `Submission`s referencing the old version still render the removed field's
  answer (they render against their own immutable schema snapshot, FR-FORM-041 — the field was never
  "removed" from *that* version). New submissions against the new version simply do not have that
  field. Any calculated field or risk score in the new version that referenced the removed field must
  be re-authored before the new version can leave `DRAFT` (build-time validation error, not a runtime
  surprise).
- **Rename**: a rename is modeled as (a) same `field_id`, label changed (not a structural change,
  just a hash-affecting content change per FR-FORM-042), vs. (b) a genuinely new field replacing an
  old one (new `field_id`) — the Builder UI must make this distinction explicit ("Rename this field"
  vs. "Delete and add new field") because it changes downstream analytics continuity. A same-`field_id`
  rename preserves historical trend continuity for reporting; a delete+add does not, and the Impact
  Preview (FR-FORM-052) flags it as `REMOVED` + `ADDED`, not `RENAMED`, when the admin used delete+add.

### FR-FORM-071 — Circular Conditional Logic
Covered structurally in FR-FORM-024/FR-FORM-061 for calculated fields (hard build-time block, cycle
must be acyclic). For conditional-**visibility**/mandatory rules (not calculated values), a true
circular dependency (A's visibility depends on B's value, B's visibility depends on A's value) is
also rejected at build time using the same dependency-graph check, because the evaluation engine
(low-level-design.md §9) requires a topological evaluation order; the rule builder (§4, FR-FORM-032)
already limits which fields can be referenced to reduce the chance of authoring a cycle, but the
server-side compiler is the authoritative enforcement point (never trust client-side prevention
alone).

### FR-FORM-072 — Repeating Group with Calculated Totals
A calculated field aggregating across a repeating group (`SUM(line_items.line_total)`) must
recompute on every instance add/remove/edit within the group, and must handle the zero-instance case
(sum of empty set = 0, not null/error) and the "instance removed mid-fill after being referenced by a
cross-record validation snapshot" case (the aggregate simply reflects current state; cross-record
Tier-1 advisory checks re-run on every recompute, not just at submission time).

### FR-FORM-073 — Submission Referencing a Superseded Template
Fully expected and permanently supported — this is the normal state of most historical `Submission`
rows once any `Template` is updated. Rendering a historical submission (for review, audit, export)
always loads the exact `TemplateVersion` by `template_version_id`, never the current `PUBLISHED`
version of the `Template`, and the UI labels it ("Viewing as submitted — Version 3, published
2026-04-01, since superseded by Version 5") so a reviewer is never confused about which standard was
in force (source-spec P7 / §24.1 acceptance test).

### FR-FORM-074 — Submission Against a RETIRED Template
New submissions cannot be started against a `RETIRED` version under any circumstance (unlike
`SUPERSEDED`, which still permits already-in-progress fills to finish, FR-FORM-044). An offline device
holding a cached `RETIRED` version (retirement happened while offline) rejects starting a new fill
locally using the same cached lifecycle-state field, and syncs the retirement status down on next
connectivity so the `WorkItem` picker no longer offers it.

### FR-FORM-075 — Duplicate/Conflicting Publish Attempts
Two admins cannot both publish a new version of the same `Template` concurrently to `PUBLISHED`; the
publish action requires the `APPROVED` version's optimistic-concurrency `version`/`ETag` to match
current state (source-spec §18.1), returning `409 Conflict` with the current state to the loser,
consistent with platform-wide concurrency convention.

## 9. Non-Functional Requirements (Module-Specific)

| NFR ID | Requirement | Target |
|---|---|---|
| NFR-FORM-01 | Builder autosave latency | Debounced save completes ≤5s after last edit; visible "Saved"/"Saving…" indicator |
| NFR-FORM-02 | Schema-hash determinism | Identical canonical schema content always produces identical hash across service instances/restarts/language runtime versions |
| NFR-FORM-03 | Publish-time validation | Full structural + logic validation (cycles, dangling refs, type checks) completes ≤3s for a template up to the FR-FORM-062 budget |
| NFR-FORM-04 | Offline schema render | A cached, valid `PUBLISHED` schema renders its first field within 300ms on a reference low-end Android device with zero network calls |
| NFR-FORM-05 | Impact Preview generation | Affected-object counts for a template used by up to 50,000 locations return within 5s (background job for exact large-scale enumeration if exceeded — see low-level-design.md §12) |
| NFR-FORM-06 | Submission ingestion idempotency | Duplicate `client_uuid` + `tenant_id` submission returns original result with no duplicate row, per source-spec §19.4 |
| NFR-FORM-07 | Version immutability integrity | Any attempt (application bug, direct DB access, replay) to mutate a `PUBLISHED` version's schema/rules/scoring is rejected at the data layer, not only the application layer (DB trigger/constraint, see low-level-design.md §2) |
| NFR-FORM-08 | Chain-hop and dialog nesting bounds | An `INLINE_CHAIN` forward sequence hard-aborts beyond `max_chain_hops` (default 10, FR-FORM-087) rather than looping indefinitely; dialog nesting is limited to depth 1 by build-time validation (FR-FORM-082), not a runtime check alone |

## 10. Dependencies

| Module | Nature of dependency |
|---|---|
| Mobile/Offline | Consumes this module's compiled `TemplateVersion` schema for local caching and rendering; owns the local DB tables (`templates`, `template_versions` per source-spec §19.1), the sync transport, and the offline conflict UI. This module defines the schema/submission JSON contracts and the idempotency key (`client_uuid`) semantics that transport rides on. This includes queuing an offline-originated `SPAWN_WORK_ITEM` forward (FR-FORM-083) through the same mutation-queue mechanism as any other offline write. |
| Evidence | Owns storage, hashing, retention, and signed access for every `evidence_bearing` field's captured media/signature/GPS/barcode metadata. This module stores only the Evidence reference (`evidence_id`, hash) inside the submission payload, never raw media. |
| Work Orchestration | Owns the `WorkItem` that triggers a form to be rendered (assignment, scheduling, SLA) and consumes `submission.created`/risk-score output to progress the `WorkItem` to `SUBMITTED`/`PENDING_REVIEW`. This module does not schedule or assign work itself. A `FORWARD_TO_TEMPLATE` action in `SPAWN_WORK_ITEM` mode (FR-FORM-083) calls Work Orchestration's `create_work_item` synchronously; Work Orchestration owns assignment resolution and the `NEEDS_ASSIGNMENT` fallback state for the spawned item — this module never implements a second assignment mechanism. |
| Issues/CAPA | Consumes risk-scoring `create_issue` action intents and `submission.created` events to open `Issue` records; owns all subsequent CAPA lifecycle. |
| Agent Runtime (Workflow Builder Agent) | Produces candidate `DRAFT` `TemplateVersion` objects via the standard Builder data contract; this module enforces that agent-authored drafts pass through the identical `REVIEW`/`APPROVED` human gate as human-authored drafts — no autonomy-level bypass of the publish gate exists for this module (source-spec §13.1 lists Workflow Builder Agent default autonomy as Recommend/L1). |
| Analytics/Control Tower | Consumes published `field_id`s, `risk_score.score_id`s, and `Submission` facts (via CDC) for reporting; this module's structural-diff (FR-FORM-052) exists specifically to warn admins before they break an analytics dependency. |
| Identity & Authorization | Supplies `UserScope`/RBAC context that scopes which `OrganizationNode`s a Builder admin may author for and which locations/users a filler's assigned templates are drawn from. |

## 11. Open Questions / Assumptions

- **Assumption**: the product-facing `TEST`/`PILOT` labels in source-spec §8.3 are UX/marketing labels
  layered over the canonical five-state machine (`DRAFT/REVIEW/APPROVED/PUBLISHED/SUPERSEDED/RETIRED`)
  per conventions.md, reconciled in FR-FORM-040. Flagging for product sign-off since source-spec text
  literally lists `TEST -> PILOT` as states.
- **Decision needed**: whether label/help-text-only changes require a full new `TemplateVersion` or a
  lighter "metadata patch" that doesn't bump the schema hash. Default taken in FR-FORM-042: full new
  version, to preserve exact historical rendering fidelity.
- **Decision needed**: default sample size (N=20) and de-identification approach for simulated
  execution against real historical submissions (FR-FORM-051) when fields are `pii_class=SENSITIVE` —
  recommend excluding those submissions from the sample pool entirely rather than attempting
  redaction, pending tenant AI-data-control configuration (source-spec §21.3).
- **Open question**: maximum nesting depth for repeating groups (default assumed 2, FR-FORM-014) and
  maximum matrix row/column counts — needs UX/perf validation against the low-end-device budget in
  FR-FORM-062.
- **Open question**: whether Pilot rollout scope changes require the same dual-control approval as a
  new publish, or a lighter single-admin action — assumed lighter (single admin, audited) since the
  underlying `TemplateVersion` content never changes during a rollout-scope update.

## 12. Interactive Cards, Dialog Actions & Cross-Template Forwarding

This section generalizes the platform's original "one field, one line" and "branch to a section"
model into three related capabilities: (1) a composable, designed layout unit (image + text +
button(s)) as a first-class component, (2) a dialog/modal action a button can trigger, and (3) a
button's outcome forwarding the filler to an **entirely different `Template`**, not just a section of
the current one — optionally spawning an independently tracked `WorkItem` rather than continuing
inline. All three are additive: every existing field type, `branch_map` (FR-FORM-023), and validation
rule continues to work unchanged; a template that uses none of this section renders exactly as it did
before.

### FR-FORM-080 — Composite Card / Layout Block
A new, non-data-bearing schema element type — `block_type: CARD` — that a section may contain
alongside its `fields` (positioned by the same `order_index`/`section_id` mechanism, §3.0). A `CARD`
composes:

| Attribute | Type | Notes |
|---|---|---|
| `block_id` | string, stable | Analogous role to `field_id`, but a pure `CARD` never appears as a key in `submission_answers` (see the two authoring patterns below). |
| `image` | object, optional | `{asset_id, alt_text}` — a **builder-uploaded static asset** (uploaded once when authoring the template), not filler-captured Evidence; rendered above the text by default. |
| `heading` / `body_text` | localized rich text, optional | Restricted rich-text subset (bold, italic, lists, links) — no arbitrary HTML or embedded scripting, consistent with the no-embedded-scripting principle already applied to the formula grammar (FR-FORM-024 rationale, high-level-design.md §8). |
| `buttons` | ordered list of 1–4 `CardButton` | `{button_id, label, style}` + one `action` each (FR-FORM-081). At least one button is required; a card with zero buttons is a plain content block (still valid — e.g., a pure instructional image+text card with no decision to make) but must not declare `record_as_field` (below). |
| `record_as_field` | boolean, default `false` | See authoring patterns below. |

Two authoring patterns, and the Builder must make the choice explicit (not inferred) because it
changes what lands in `submission_answers`:
- **Pattern A — Decorated Field** (`record_as_field: true`, or equivalently authored by adding a
  `card_decoration` to an existing `YES_NO`/`SELECT_SINGLE`/`RATING` field, §3.5/§3.3/§3.7): the
  card's buttons *are* that field's options; selecting one persists a normal `submission_answer` exactly
  as it would without the decoration. This is the exact pattern in the motivating example — "image on
  top, text below, Yes/No" where the Yes/No answer is itself a reportable fact.
- **Pattern B — Pure Navigation Card** (`record_as_field: false`, the default for a standalone `CARD`
  block not tied to a field): the button choice is not itself persisted as data — only its *action*
  (FR-FORM-081) executes. Used for menu-like "choose your path" screens where the choice is a UI
  branch point, not a fact worth reporting (e.g., "What would you like to do next?" with buttons
  "Report an issue" / "Continue checklist").
A template mixing many Pattern-B cards with no underlying field is still fully deterministic to
evaluate: Pattern-B button clicks are logged as `submission_navigation_events` (low-level-design.md
§2) for UX analytics/debugging, but never populate `submission_answers` and are never inputs to risk
scoring or cross-field validation — only real fields are.

### FR-FORM-081 — Button / Option Action Types
Every button — on a `CARD` block (FR-FORM-080), or attached to an individual option of an existing
`YES_NO`/`SELECT_SINGLE`/`RATING` field (extending `branch_map`, FR-FORM-023, to a richer action
rather than only a section target) — declares exactly one `action`:

| Action | Effect |
|---|---|
| `NAVIGATE_SECTION` | Unchanged existing behavior: jump to a section within the same `TemplateVersion` (the `branch_map` case, FR-FORM-023). |
| `SHOW_DIALOG` | Opens a `DialogDefinition` (FR-FORM-082) as an on-device modal. |
| `FORWARD_TO_TEMPLATE` | Hands off to a different `Template` (FR-FORM-083). |
| `SUBMIT_FORM` | Ends the fill session immediately and submits, skipping any remaining optional sections (still subject to all mandatory-condition checks, FR-FORM-021, for whatever is currently visible). |
| `NONE` | Decorative/no-op button (e.g., a "Learn more" label with no action) — valid only in combination with a sibling button that does carry an action, per FR-FORM-080's "at least one button required" rule meaning at least one non-`NONE` action must exist somewhere on the card. |

A button carries exactly one action — actions do not chain directly off one another — but a
`SHOW_DIALOG`'s own buttons may each carry a *second* action (nesting depth exactly 1, enforced at
build time, FR-FORM-082), which is how "confirm, then forward" composes without a fourth primitive.

### FR-FORM-082 — Dialog Action Primitive
A `DialogDefinition`: `{dialog_id, title, body_text, image (optional), dismissible (boolean, default
true), buttons: [DialogButton]}`, where each `DialogButton` is `{label, style, action}` drawn from the
same action set as FR-FORM-081 **except** `SHOW_DIALOG` itself — a dialog's button cannot open a
second dialog. This is a hard build-time rule (`DIALOG_NESTING_NOT_ALLOWED`, §8), not a soft warning,
because unbounded modal-stacking is both a poor UX and a way to author an unreachable/looping
authoring state.

- Rendering is entirely on-device from the cached compiled schema — no network round-trip to display
  a dialog — consistent with the offline-first rendering contract (FR-FORM-063).
- A dialog with zero buttons is invalid at build time (`DIALOG_NO_BUTTONS`, §8): every dialog needs at
  least one way out, whether that is a plain dismiss (`NONE`) or a real action, so the filler is never
  stuck in an unrecoverable modal.
- `dismissible: false` disables the platform back/hardware-back gesture while the dialog is open,
  forcing an explicit button choice — intended for genuine acknowledgements ("This reading is outside
  safe range — you must choose Escalate or Continue"), not general-purpose UX polish; the Builder
  surfaces a warning encouraging admins to reserve `dismissible: false` for cases that actually need
  forced acknowledgement.
- Dialogs never themselves create a `submission_answer`. If a dialog's real purpose is to capture a
  value ("Enter a reason for overriding this warning"), that value must be modeled as a normal field
  (e.g., a `TEXT_SHORT` field made conditionally required/visible off the button that opened the
  dialog) — a dialog is informational, confirmatory, or branching UI, never a data-entry surface in
  its own right.

### FR-FORM-083 — Cross-Template Branching & Forwarding
Extends FR-FORM-023 to a hand-off between two **different** `Template`s. A button's (or a
`YES_NO`/`SELECT_SINGLE`/`RATING` option's) `action = FORWARD_TO_TEMPLATE` config carries:

- `forward_to_template_id` — resolved to the target `Template`'s **`current_published_version_id` at
  the moment the forward is taken**, never a version pinned inside the source schema. This mirrors how
  a `WorkItem` points at a `Template` for scheduling while a `Submission` pins to one exact
  `TemplateVersion` once started (conventions.md immutability principle): the *link* between templates
  always follows "whatever is current," but each individual hop, once taken, is as immutable as any
  other submission.
- `forward_mode` — one of two values, with materially different runtime effects:
  - **`INLINE_CHAIN`** — the target template renders immediately in the same fill session, on the
    same device, without leaving the originating `WorkItem`. Produces a second `Submission`, linked to
    the first by a shared `submission_chain_id` and an ordered `chain_sequence` (FR-FORM-084). The
    originating `WorkItem` does not progress to `SUBMITTED` until the chain reaches a hop with no
    further outstanding forward or the filler explicitly triggers `SUBMIT_FORM`. Motivating example:
    "No, the equipment was not sealed properly" immediately opens a short "Equipment Damage Detail"
    template as part of finishing the same site visit — no separate task is needed.
  - **`SPAWN_WORK_ITEM`** — forwarding creates a new, **independently trackable** `WorkItem` via a
    synchronous call to Work Orchestration's `create_work_item` (Work Orchestration functional-spec,
    Work Item Creation), of a configurable `work_item_type`, assigned per a configurable assignment
    rule (reusing Work Orchestration's existing assignment primitives — role-at-location, specific
    user, manager-of-submitter — not a new mechanism), with a configurable SLA. The **current**
    `WorkItem`/submission completes normally against its own template; the spawned `WorkItem` is
    related but independent, linked back via `originating_submission_id`. Motivating example: "No, the
    door seal is damaged" spawns a full Maintenance Work Order assigned to the Facilities queue, while
    the inspector's own checklist still finishes and submits on its normal schedule.
- `data_mapping` (optional, either mode) — an ordered list of `{source_field_id, target_field_id}`
  pairs pre-filling the target template's fields from the source submission's already-entered answers
  at hand-off time (e.g., carry `location_id`, `asset_id`, and the triggering photo forward
  automatically). Pre-filled values remain filler-editable unless the pair is marked `locked: true`,
  and land on the target's `submission_answers` with a new `is_forwarded_prefill = true` flag
  (distinct from `is_inherited_default`, §3.0) so analytics can distinguish a carried-over value from
  an admin-configured default.
- The target `Template` must be `ACTIVE` (not `ARCHIVED`) with a resolvable
  `current_published_version_id` at the moment the forward is taken — see FR-FORM-087 for the
  unavailable-target runtime behavior.

### FR-FORM-084 — Submission Chain Integrity
A new entity, `SubmissionChain` (low-level-design.md §2), is the authoritative ledger of every
`INLINE_CHAIN` hop: the ordered list of `submission_id`s, the button/action that caused each hop, and
an overall chain status (`IN_PROGRESS`, `COMPLETE`, `ABANDONED`). Chaining never merges two templates'
schemas into one document — each hop's `Submission` still points at its own `template_version_id`
independently, and the version-immutability guarantee (FR-FORM-041) applies per hop exactly as it does
to a standalone submission. A reviewer opening any submission that is part of a chain sees a "part of
a chain — step 2 of 3" indicator linking to sibling hops.

Offline behavior: an `INLINE_CHAIN` hop that begins while offline requires the target
`TemplateVersion` to already be present in the device's local cache (the same contract as
FR-FORM-063, applied per hop). If it is not cached, the forward button renders in a disabled
"unavailable offline" state rather than allowing a jump into a form the device cannot render.

### FR-FORM-085 — Builder Authoring UX for Cards, Dialogs, and Forwarding
Extends §5 Visual Builder UX. The canvas palette gains a "Layout & Actions" category (Card block,
Dialog) alongside the existing field-type palette (FR-FORM-030). Configuring a button's action reuses
the existing Logic Rule Builder (FR-FORM-032) with a new "Action" tab alongside the existing
"Visibility"/"Validation" tabs, so authors do not learn a second rule-authoring surface.

Selecting `FORWARD_TO_TEMPLATE` shows a live picker scoped to `Template`s the current admin already
has `UserScope` authorship/read access to (never an opaque ID field) and surfaces the target
template's current field list to drive `data_mapping` configuration via a simple two-column
source-field -> target-field mapper, type-checked at save time (e.g., mapping a `PHOTO` field to a
`NUMBER` field is a build-time error, `FORWARD_MAPPING_TYPE_MISMATCH`, §8).

Test/Preview Mode (FR-FORM-033) simulates the whole chain: testing a card's "Yes" button walks the
tester through the forwarded template inline for `INLINE_CHAIN`, or shows a non-mutating "Would create
WorkItem: `<type>`, assigned to `<resolved assignment>`" summary for `SPAWN_WORK_ITEM` — a test run
must never create a real `WorkItem`.

Impact Preview (§6) gains a new downstream-diff category, `FORWARDING_CHANGED`, alongside the existing
field-level categories (FR-FORM-052): publishing a change to a template's forwarding configuration
(adding, removing, or repointing a `FORWARD_TO_TEMPLATE` action, or changing a `data_mapping`) can
silently redirect where a filler ends up without changing any of the source template's own fields, so
it must surface as its own warning category rather than being invisible to the Impact Preview scan.

### FR-FORM-086 — UX States (extends §7)
- **Dialog open** — a modal overlay state; the platform back/hardware-back gesture dismisses the
  dialog only when `dismissible: true` (default); a mandatory-acknowledgement dialog
  (`dismissible: false`) disables back and requires an explicit button choice.
- **Forwarding transition** (`INLINE_CHAIN`) — a brief, explicit "Continuing to: `<target template
  name>`" transition state; a hand-off between templates is never a silent, unexplained screen swap.
- **Work item spawned** (`SPAWN_WORK_ITEM`) — a lightweight, dismissible confirmation ("A Maintenance
  Work Order has been created and assigned to Facilities") shown inline; this is advisory UI only —
  the current form continues and completes independently of whether the filler dismisses it quickly.

### FR-FORM-087 — Validation and Edge Cases (extends §8)
- **Forward target retired/archived between publish and fill-time**: at render time, if
  `forward_to_template_id` no longer resolves to an `ACTIVE` `Template` with a current published
  version, the button renders disabled with an "this action is currently unavailable" label — never a
  runtime error or crash. The event is logged (`form.forward_target_unavailable`) so an admin can be
  alerted to repoint or remove the action. The **source** template is never blocked from being filled
  or submitted because one of its forward targets became unavailable — forwarding is an optional
  continuation, not a submission gate.
- **Circular forwarding** (Template A forwards to B, B forwards back to A, directly or transitively):
  detected at **publish time** as a best-effort static reachability check across the
  `forward_to_template_id` edges reachable from the template being published (a warning, since a
  target template can independently change its own forwards later — this is not a hard cross-template
  lock). At **runtime**, a hard `max_chain_hops` limit (default 10, NFR-FORM-08) aborts an
  `INLINE_CHAIN` that exceeds it with a "this workflow could not be completed automatically" state,
  flagging the submission for admin review rather than looping indefinitely.
- **Data-mapping type mismatch introduced later** (the *target* template's field type changes on a
  newer version): re-validated whenever the target template publishes; a `data_mapping` entry that no
  longer type-checks is automatically disabled (never silently coerced), and the **source** template's
  Impact Preview surfaces a new `FORWARDING_CHANGED` warning the next time it is opened in the
  Builder — even though the source template's own content did not change.
- **Multiple forward actions from the same field/card** (e.g., "Yes" forwards to Template B, "No"
  forwards to Template C): fully supported; each option/button carries an independent action, with no
  limit on the number of distinct forward targets from one field — only the chain-depth limit above
  applies.
- **`SPAWN_WORK_ITEM` assignment resolution fails** (e.g., "assign to my manager" configured, but the
  filler has no manager on record): the spawned `WorkItem` is still created (never silently dropped)
  but lands in Work Orchestration's own `NEEDS_ASSIGNMENT` state — this module reuses that existing
  fallback rather than inventing a second one.
- **Offline `SPAWN_WORK_ITEM`**: the create-work-item call is queued through the same
  offline-mutation-queue mechanism as any other offline-originated write (Mobile/Offline module), not
  treated as a special case; it syncs on reconnect, and Work Orchestration performs assignment
  resolution at that point using data as of sync time (except for whatever was already captured via
  `data_mapping`, which is snapshotted at forward time).

### Open Questions for This Extension
- **Decision needed**: whether a Pattern-B (FR-FORM-080) navigation-only button click should be
  retained indefinitely in `submission_navigation_events` for UX analytics, or purged after a shorter
  retention window than actual submission data — assumed shorter (default 90 days), since it is
  debugging/analytics telemetry, not a record of what the filler attested to.
- **Decision needed**: whether `SPAWN_WORK_ITEM` should support a "wait for the spawned WorkItem's
  outcome" mode (i.e., the originating submission's own status reflects whether the spawned Maintenance
  Work Order was later resolved) — out of scope for this iteration; the two WorkItems are linked for
  traceability (`originating_submission_id`) but progress fully independently.
- **Open question**: maximum `buttons` count per `CARD` (assumed 4, FR-FORM-080) and maximum image
  asset size/dimensions for builder-uploaded card images — needs UX/perf validation against the
  low-end-device budget (FR-FORM-062), since a `CARD` image is not subject to Evidence's
  capture-time compression pipeline (it is a static authored asset, not filler-captured media).
