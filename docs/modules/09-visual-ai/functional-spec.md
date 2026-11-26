# Visual AI — Functional Specification

Module corresponds to **EPIC-10 (Visual AI)**. Source: source-spec.md §12 (Visual AI, incl. the
exact JSON result contract), §13.1 (Visual Inspection Agent row), §24.1 ("Visual AI review"
acceptance test), Appendix E EPIC-10. Cross-references §10 (Evidence, Issues and CAPA), §20
(AI Runtime and Governance FSD), §21.3 (AI Data Controls).

## 1. Purpose and Scope

### 1.1 Purpose
Visual AI turns a captured image (or video frame) into a **structured, reviewable compliance
signal** rather than an opaque pass/fail. It lets an enterprise define, per tenant and per use case,
what "good" looks like in a photo (a compliant shelf, a clean floor, a worker wearing required PPE,
signage placed correctly), have a model score newly captured evidence against that definition, and
route the result through a confidence-governed workflow so that only sufficiently reliable AI
findings are treated as compliance fact without a human in the loop. This module is the concrete
implementation of Product Principle P5 ("Policy before AI") and P8 ("Explainability") applied to
computer vision: **the result contract, confidence threshold, and human-review path are the
product**, not merely the model.

### 1.2 In Scope
- The seven Visual AI use cases (§12.1): planogram/shelf compliance, promotional signage, brand
  execution, cleanliness, PPE/safety, asset condition, vehicle/site condition — as a configurable
  `UseCaseConfig` catalog, not seven hardcoded pipelines.
- The structured result contract (§12.2): `compliance_score`, `confidence`, `status`,
  `violations[]` with `type`, `severity`, `confidence`, `bounding_box` — exactly how each field is
  produced, stored, and consumed by downstream modules.
- Confidence-threshold-driven routing: auto-pass, auto-fail, and human-review, per tenant and per
  use case.
- The human-review workflow: reviewer queue, bounding-box overlay UI requirements, accept/
  override/reject actions, and the resulting `ReviewDecision` record.
- The appeals workflow for a store/site that disputes a reviewed (or auto-decided) result.
- Model/version traceability: every `VisualAiResult` is pinned to the exact model backend, model
  version, and `UseCaseConfig` version used to produce it.
- Precision/recall monitoring per tenant and per use case, computed from reviewer ground truth.
- The retraining feedback loop: reviewer overrides become labeled training data for model
  improvement (data capture only — model retraining/deployment itself is owned by the model
  backend / Agentic Ops model governance, §13.1/§20).

### 1.3 Out of Scope (owned by other modules, referenced here)
- Image/video capture, upload, hashing, storage, retention — **Evidence module (EPIC-06)**. Visual
  AI is triggered by `evidence.uploaded` and reads media via signed URLs; it never owns the media
  object itself.
- Creation and lifecycle of the `WorkItem` that requested the photo (e.g., an audit checklist item
  requiring a shelf photo) — **Work Orchestration (EPIC-03)**. Visual AI reports a result against
  an `evidence_id`/`work_item_id`; it does not manage assignment, scheduling, or SLA for the
  underlying work.
- Corrective task creation and CAPA lifecycle triggered by a violation — **Issues & CAPA
  (EPIC-07)**. Visual AI emits an event and, for `HIGH` severity, requests issue creation; it does
  not own root-cause analysis or corrective-action state.
- The Visual Inspection Agent's autonomy policy, tool allowlist, and audit trail as an `AgentRun` —
  **Agentic Operations (EPIC-11)**. Visual AI is the *system of record for inference results*; the
  Visual Inspection Agent (when enabled) is a *consumer/orchestrator* that may call Visual AI's
  APIs as a typed tool and separately produces its own `AgentRun` record.
- Model hosting, GPU/inference infrastructure ownership, and the model-provider-agnostic gateway
  implementation itself — **Agentic Ops / AI Runtime platform layer (§20)**. Visual AI depends on
  that gateway as a pluggable backend; it does not run inference infrastructure itself.
- General analytics/warehouse rollups beyond the module's own precision/recall metrics — **Analytics
  and Control Tower (EPIC-12/15)**.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Frontline Associate / Auditor | Captures the photo/video that triggers Visual AI inference; sees a processing-in-progress state and, once scored, sees pass/fail/needs-review feedback inline in their work item; may submit an appeal against a result they believe is wrong. |
| Store/Site Manager | Reviews flagged (low-confidence or auto-failed, tenant-configurable) results in the Human Review queue for their location(s); accepts, overrides, or rejects AI findings; resolves appeals raised by their team. |
| Operations Admin | Configures which use cases are enabled per tenant, defines/edits `UseCaseConfig` (model backend selection, confidence thresholds, severity mapping, auto-fail eligibility), reviews precision/recall dashboards, approves model version upgrades before cutover. |
| Regional/District Manager | Views aggregated Visual AI compliance and override-rate trends across their territory (via Analytics); may be a second-level appeal resolver per tenant policy. |
| Visual Inspection Agent | System actor (per §13.1, default autonomy `L1 Recommend`) that may be configured to auto-trigger re-inspection, summarize a location's recurring visual failures, or propose (not execute) corrective work based on `VisualAiResult` history; consumes this module's APIs/events as typed tools, never bypasses the confidence-routing policy. |

## 3. Result Contract — Field-by-Field

The contract in source-spec §12.2 is the wire and storage format. This section defines exactly how
each field is produced and who consumes it.

```json
{
  "compliance_score": 81,
  "confidence": 0.94,
  "status": "AI_RESULT_UNREVIEWED",
  "violations": [
    {"type": "missing_product", "severity": "HIGH", "confidence": 0.96,
     "bounding_box": [120, 80, 320, 410]}
  ]
}
```

- **FR-VAI-001 — `compliance_score` (0–100, integer).** Produced by the model backend's use-case-
  specific scoring function (e.g., percentage of planogram slots correctly filled; weighted
  inverse of detected violation severity for cleanliness/PPE use cases). The exact scoring formula
  is use-case-specific and versioned as part of `UseCaseConfig.scoring_version`; the platform does
  not assume a single universal formula, only that it is deterministic given the same model output
  and configuration. Consumed by: the result contract builder (routing input, §4), Work
  Orchestration (may gate a WorkItem's `VERIFIED` transition on a minimum score), Analytics
  (compliance trend KPIs).
- **FR-VAI-002 — `confidence` (0.0–1.0, float, top-level).** The model backend's own calibrated
  confidence in the **overall result** (not any single violation) — i.e., "how sure is the model
  that this compliance_score and violation set are correct." This is the primary input to
  confidence-threshold routing (§5). Model backends that do not natively emit a calibrated overall
  confidence MUST derive one deterministically (e.g., minimum or weighted-average of per-violation
  confidences, or a backend-specific calibration curve) — a use case cannot be enabled without a
  defined confidence derivation (§10, validation).
- **FR-VAI-003 — `status`.** The `VisualAiResult` lifecycle state (full enum and transitions in
  `architecture-diagram.md` state diagram). Values used by this module, extending the source-spec
  seed value `AI_RESULT_UNREVIEWED`: `AI_RESULT_PENDING`, `AI_RESULT_SCORED`,
  `AI_RESULT_UNREVIEWED` (retained as the alias for "auto-decided or queued, not yet human-touched,"
  — see §10 for why the source-spec name is kept rather than replaced), `AI_RESULT_AUTO_PASSED`,
  `AI_RESULT_AUTO_FAILED`, `AI_RESULT_PENDING_REVIEW`, `AI_RESULT_REVIEWED`,
  `AI_RESULT_APPEALED`, `AI_RESULT_RESOLVED`, `AI_RESULT_FAILED_PROCESSING`.
- **FR-VAI-004 — `violations[]`.** Zero or more detected deviations. Each violation is a
  first-class stored row (`Violation`, not just a JSON blob) so it can be individually reviewed,
  overridden, linked to a CAPA issue, and used as a labeled training example (§9, §11). Fields:
  - `type` — a use-case-scoped taxonomy value (e.g., `missing_product`, `wrong_facing`,
    `expired_signage`, `missing_ppe_item`, `spill_hazard`, `damaged_asset`). Taxonomy is part of
    `UseCaseConfig` (tenant may extend the default taxonomy with custom violation types).
  - `severity` — `LOW | MEDIUM | HIGH`, mapped from the model's raw output via a tenant-configurable
    `severity_map` in `UseCaseConfig` (e.g., a tenant may classify `missing_ppe_item` as HIGH while
    another treats it as MEDIUM pending a grace period).
  - `confidence` — per-violation confidence, independent of the top-level `confidence`; a single
    high-confidence-overall result can still contain one low-confidence violation, which is
    surfaced distinctly in the review UI (§7).
  - `bounding_box` — `[x_min, y_min, x_max, y_max]` in source-image pixel coordinates at the
    resolution the model scored (stored alongside the scored image's `width`/`height` so the UI can
    scale the overlay to whatever resolution it renders, including a downscaled thumbnail).
- **FR-VAI-005 — Contract versioning.** The result contract itself is versioned
  (`result_contract_version`, currently `1`) and stored on every `VisualAiResult` row, distinct from
  `model_version` and `use_case_config_version`, so a future field addition (e.g., a segmentation
  mask alongside bounding boxes) can be introduced without breaking consumers pinned to v1.

## 4. Per-Use-Case Model Configuration

- **FR-VAI-010.** Each of the seven use cases (§12.1) is configured per tenant as one or more
  `UseCaseConfig` rows (a tenant may run more than one config per use case, e.g., a stricter
  planogram config for flagship stores). A `UseCaseConfig` binds: `use_case` (enum), a
  `model_backend_ref` (which pluggable backend/model family to call, resolved through the
  model-provider-agnostic gateway per conventions), `model_version` pin, violation taxonomy,
  severity map, scoring version, and the confidence thresholds in §5.
- **FR-VAI-011.** A use case is **disabled by default** for a tenant until an Operations Admin
  explicitly configures and activates it (`UseCaseConfig.status = ACTIVE`). Evidence uploaded
  against a work item that references an unconfigured use case does not silently fail; see §10
  edge case "use-case model not yet configured."
- **FR-VAI-012.** `UseCaseConfig` is versioned (`config_version`, monotonic per `use_case` +
  tenant); a `VisualAiResult` always records the exact `config_version` in force at scoring time,
  independent of later config edits (mirrors the platform-wide immutability principle, P7).
- **FR-VAI-013.** Which `WorkItem`/form field triggers which `UseCaseConfig` is declared at
  authoring time on the Evidence field (Forms module owns the field; it stores a
  `visual_ai_use_case` reference resolved to the tenant's active config for that use case at
  submission time — not at authoring time — so a later-activated or re-versioned config always
  applies to new evidence without a form republish).

## 5. Confidence-Threshold-Driven Routing

- **FR-VAI-020.** Every `UseCaseConfig` defines two thresholds: `auto_pass_confidence_threshold`
  and `auto_fail_confidence_threshold` (both 0.0–1.0, `auto_pass` need not equal `auto_fail`).
  Routing decision (see `low-level-design.md` §4.1 for full pseudocode):
  - `confidence >= auto_pass_confidence_threshold` AND no violation with `severity = HIGH` above
    its own per-violation threshold → **`AI_RESULT_AUTO_PASSED`**.
  - `confidence >= auto_fail_confidence_threshold` AND at least one violation meets the tenant's
    auto-fail violation criteria (§5, FR-VAI-022) → **`AI_RESULT_AUTO_FAILED`**.
  - Otherwise → **`AI_RESULT_PENDING_REVIEW`** (the safe default; low confidence never
    auto-decides in either direction).
- **FR-VAI-021.** `auto_fail` is a **separately gated capability**, off by default per use case
  (`UseCaseConfig.auto_fail_enabled = false` at creation). It can only be turned on after the
  minimum precision/recall gate in NFR-VAI-03 is met and an Operations Admin explicitly enables it.
  Until enabled, any result that would otherwise auto-fail instead routes to
  `AI_RESULT_PENDING_REVIEW` — this directly satisfies the §24.1 acceptance test ("Low-confidence
  image result enters configured human-review state rather than auto-failing") and generalizes it:
  a tenant that has not yet earned auto-fail behaves as if every result were "low confidence" for
  fail purposes.
- **FR-VAI-022.** Auto-fail violation criteria are configurable per `UseCaseConfig`:
  `min_violation_severity_for_autofail` (default `HIGH`) and
  `min_violation_confidence_for_autofail` (default 0.90). A HIGH-severity violation scored at 0.80
  confidence, for example, does not trigger auto-fail even if the top-level `confidence` is high —
  it routes to review, since the specific violation driving the fail decision is itself uncertain.
- **FR-VAI-023.** Thresholds are per tenant **and** per use case (never a single global default
  silently applied); the Operations Admin console must show, for each active `UseCaseConfig`, the
  current thresholds, `auto_fail_enabled` state, and a live-updated precision/recall summary
  alongside the threshold editor so threshold changes are made with the relevant evidence in view.
- **FR-VAI-024.** A threshold or `auto_fail_enabled` change takes effect only for evidence scored
  after the change; in-flight/queued inference is unaffected (routing decision uses the
  `UseCaseConfig` version resolved at the time scoring completes, per FR-VAI-012).

## 6. Human-Review Workflow

- **FR-VAI-030 — Review queue.** Results in `AI_RESULT_PENDING_REVIEW` appear in a per-location
  (and roll-up per-region) Human Review queue, visible to any user with `visual_ai:review`
  permission at the result's location scope. Queue is orderable/filterable by use case, severity of
  detected violations, confidence (ascending — least confident first, the default sort, so reviewer
  attention goes to the hardest cases first), and age (oldest first, for SLA visibility).
- **FR-VAI-031 — Review UI requirements.** The reviewer sees: the source image (or the specific
  video frame) rendered at full resolution, a bounding-box overlay drawn from stored
  `Violation.bounding_box` coordinates for every violation (color-coded by severity), the
  `compliance_score` and `confidence`, and the model/version that produced the result
  (`model_backend`, `model_version`, `use_case_config_version` — human-readable, not opaque IDs).
  Overlay must remain correctly positioned regardless of the image's rendered zoom/pan state
  (coordinates are stored relative to the original scored resolution and transformed client-side).
- **FR-VAI-032 — Reviewer actions.** Per violation and for the result as a whole:
  - **Accept** — reviewer agrees with the AI finding as-is; no data changes except recording the
    `ReviewDecision`.
  - **Override** — reviewer changes the outcome: adjust `compliance_score`, change a violation's
    `severity`, mark a violation as a false positive (removed from the count that drives
    compliance), or add a violation the model missed (manually drawn bounding box + type +
    severity). Every override is a first-class, individually attributable field-level change (not a
    single freeform "reviewer disagreed" flag) so it is usable as labeled training data (§9).
  - **Reject** — reviewer disputes the result as a whole (e.g., wrong photo, unusable image
    quality) without asserting a specific corrected outcome; this routes the underlying evidence
    back to the associate for recapture rather than closing with a fabricated score.
  - A required `reason` (free text, minimum length enforced) is captured for Override and Reject,
    always attributed to `reviewer_id` and timestamped server-side.
- **FR-VAI-033 — Review outcome.** A completed review transitions the result to
  `AI_RESULT_REVIEWED` and creates a `ReviewDecision` row (§ LLD schema) referencing the exact
  `VisualAiResult` version reviewed. The `compliance_score`/`status` visible to downstream consumers
  (Work Orchestration, Issues/CAPA, Analytics) after review reflects the **reviewer's final
  determination**, not the original AI output — the original AI output is retained unmodified on
  the result row for audit/training purposes (§9).
- **FR-VAI-034 — Conflicting reviews.** If a tenant policy requires dual review (configurable,
  default off — single reviewer sufficient), and two reviewers submit different determinations, the
  result enters a `NEEDS_ADJUDICATION` sub-state (surfaced as a distinct queue) requiring a
  third, higher-scope reviewer (default: next OrganizationNode level up) to resolve; both original
  `ReviewDecision` rows are retained.
- **FR-VAI-035 — Review SLA.** Each `UseCaseConfig` (or tenant default) defines a review SLA
  (default 24 hours from `AI_RESULT_PENDING_REVIEW` entry); breach triggers the same
  escalation-ladder mechanism used elsewhere in the platform (Work Orchestration's escalation
  engine is reused — a `PENDING_REVIEW` Visual AI result registers a durable SLA timer the same way
  a WorkItem does), notifying the location manager and, on further breach, the district manager.

## 7. Appeals Workflow

- **FR-VAI-040.** Any result that has reached a terminal-ish decision state
  (`AI_RESULT_AUTO_PASSED`, `AI_RESULT_AUTO_FAILED`, or `AI_RESULT_REVIEWED`) can be appealed once
  by a user with visibility into that result (typically the frontline associate/auditor who
  captured the evidence, or their manager) within a tenant-configured appeal window (default 7
  days from decision).
- **FR-VAI-041.** An appeal requires a reason and may attach supplemental evidence (e.g., a second
  photo demonstrating correction or context the original image lacked). Filing an appeal
  transitions the result to `AI_RESULT_APPEALED` — this does not retroactively unwind any
  downstream action already taken (e.g., a CAPA issue already created stays open and is
  independently updated/closed if the appeal succeeds) but does pause the module's own SLA/
  escalation timers for that result.
- **FR-VAI-042.** Appeals route to a resolver (default: the location's manager if the original
  decision was AI-only/auto; the next OrganizationNode level up if the original decision was
  already human-reviewed, to guarantee a different/higher-scoped reviewer). The resolver can
  **uphold** or **overturn** the original decision (overturn requires the same field-level
  correction structure as a reviewer Override, §6). Resolution transitions the result to
  `AI_RESULT_RESOLVED` and creates a `ReviewDecision` row with `decision_source = APPEAL`.
- **FR-VAI-043.** Appeal SLA (default 5 business days) is monitored by a background sweeper
  (§ LLD) that escalates unresolved appeals the same way review SLA breaches are escalated.
- **FR-VAI-044.** An overturned auto-failed result that had already triggered a HIGH-severity
  CAPA issue (§8) emits an event so the Issues/CAPA module can flag the issue for manager review of
  continued relevance; Visual AI does not directly close another module's Issue record.

## 8. Model/Version Traceability

- **FR-VAI-050.** Every `VisualAiResult` permanently records: `model_backend_id`,
  `model_version`, `use_case_config_version`, `result_contract_version`, and the `inference_run_id`
  correlating to the underlying Agent Runtime/model-gateway call (if the request passed through the
  Agent Runtime rather than a direct backend call — see HLD). This is the concrete instantiation of
  the platform-wide "model-version traceability" AI data control (source-spec §21.3).
- **FR-VAI-051.** A `model_versions` catalog (§ LLD) tracks every model version ever used per
  `use_case`, its rollout stage (`SHADOW`, `ACTIVE`, `DEPRECATED`, `RETIRED`), and its own
  precision/recall snapshot at the time it was promoted to `ACTIVE`, so historical results can be
  interpreted in the context of the model that produced them (e.g., explaining a step-change in a
  compliance trend chart as a model version upgrade, not an actual operational shift).
- **FR-VAI-052.** Model version upgrades follow the platform Release Path pattern for AI (§24.3:
  `OFFLINE EVALUATION -> SHADOW -> RECOMMENDATION -> APPROVAL MODE -> CONTROLLED AUTONOMY`), applied
  here as: a candidate model version is registered `SHADOW` and scored in parallel on live traffic
  without affecting routing decisions (shadow score stored but not used for auto-pass/auto-fail),
  then promoted to `ACTIVE` (replacing the prior version for new inferences) only after an
  Operations Admin reviews a shadow-vs-production precision/recall comparison (see LLD sequence
  diagram §3.3).

## 9. Precision/Recall Monitoring and Retraining Feedback Loop

- **FR-VAI-060.** Precision and recall are computed per tenant, per use case, per model version,
  using `ReviewDecision` outcomes as ground truth against the original AI output: a reviewed/
  appealed result where the reviewer **Accepted** the AI's violation/pass call counts as a true
  positive/true negative; an **Override** or overturned appeal counts as a false positive or false
  negative depending on the direction of the correction. Computation detail in
  `low-level-design.md` §4.2.
- **FR-VAI-061.** Every `ReviewDecision` (Accept, Override, or Reject with a corrected outcome, and
  appeal resolutions) is captured as a labeled training example: the source image reference, the
  original model output, and the corrected/confirmed ground truth, tagged with `use_case`,
  `model_version`, `tenant_id`, and `config_version`. This labeled set is exported (batch, not
  real-time) to the model training pipeline owned by Agentic Ops model governance; Visual AI's
  responsibility ends at producing a well-formed, correctly attributed labeled dataset extract —
  it does not run training jobs itself.
- **FR-VAI-062.** Per platform-wide tenant isolation (source-spec §21.1) and the AI data control
  "no cross-tenant retrieval" (§21.3), a tenant's labeled review data is **never** used to train or
  fine-tune a model version deployed for a different tenant unless that tenant has explicitly
  opted in to a shared/aggregate model program (opt-in flag on `Tenant`, default off). A dedicated
  per-tenant model is trained only from that tenant's own data plus any shared base model provided
  by the platform vendor.

## 10. UX States

| State | Behavior |
|---|---|
| Processing in progress (on upload) | Immediately after `evidence.uploaded`, the associated WorkItem/evidence UI shows a distinct "Analyzing photo..." indeterminate state (not a generic spinner shared with upload progress — upload completion and inference completion are separate signals). If inference has not completed within a tenant-configurable timeout (default 60s), the UI switches to "Still analyzing — you can continue" and does not block the associate from proceeding with other work; the result appears asynchronously when ready (via sync/push) rather than forcing a wait. |
| Low-confidence / needs-review badge | A result in `AI_RESULT_PENDING_REVIEW` shows a distinct amber "Needs manager review" badge to the capturing associate (not a pass or fail signal) so they are not falsely told they failed or passed while a human decision is pending. |
| Auto-pass / auto-fail badge | Green "Compliant" / red "Non-compliant" shown immediately when `AI_RESULT_AUTO_PASSED`/`AI_RESULT_AUTO_FAILED`, with a visible "AI-verified" tag distinct from a human-reviewed result (badge style differs from a `AI_RESULT_REVIEWED` result, which shows "Manager-verified"). |
| Reviewer override recorded | After a reviewer submits an Override, the capturing associate sees an updated result with a "Reviewed by [manager] — corrected" indicator and, if the correction changed their compliance status, an in-app notification explaining what changed and why (reason text surfaced, not just the new score). |
| Appeal submitted | Appealing user sees "Appeal submitted — awaiting review" with the resolver's name/role (not just "pending"); result badge shows an "Under appeal" overlay on top of its prior state rather than reverting to neutral. |
| Appeal resolved (upheld) | Original status badge restored with an added "Appeal reviewed — upheld" note, reason visible. |
| Appeal resolved (overturned) | Status badge updates to the corrected outcome with "Appeal reviewed — overturned" and reason visible; any dependent CAPA issue shows a cross-reference flag (§ FR-VAI-044). |
| Model backend unavailable / inference failed | Result shows `AI_RESULT_FAILED_PROCESSING` with a neutral "Unable to analyze — will retry" state, never presented as a compliance fail; work item completion is not blocked on this (configurable per use case whether a photo is still "accepted" as evidence pending later scoring). |
| Poor image quality detected pre-scoring | If the model backend or a pre-check step flags the image as unscoreable (blur, darkness, wrong subject), the UI immediately prompts "Photo quality too low for automatic check — retake or submit for manual review" rather than silently producing a low-confidence score. |

## 11. Validation and Edge Cases

- **FR-VAI-070 — Image quality too poor for reliable inference.** The model backend (or a
  lightweight pre-inference quality classifier run first, tenant-configurable) can return a
  `quality_gate_failed` signal instead of a scored result. This produces a `VisualAiResult` in a
  distinct `AI_RESULT_PENDING_REVIEW` state flagged `quality_flag = true` (routed to review rather
  than fabricating a low-confidence auto-decision) and the capture UX offers an immediate retake
  (§10). Repeated quality failures on the same evidence field are tracked so Operations Admins can
  see if a use case's capture instructions need improvement.
- **FR-VAI-071 — Use-case model not yet configured for a tenant.** Evidence submitted against a
  form field that declares a `visual_ai_use_case` for which the tenant has no `ACTIVE`
  `UseCaseConfig` does not silently drop the request. The evidence is accepted normally (Evidence
  module responsibility, unaffected); Visual AI creates a `VisualAiResult` row in
  `AI_RESULT_FAILED_PROCESSING` with `failure_reason = USE_CASE_NOT_CONFIGURED`, visible to
  Operations Admins as an actionable "N evidence items awaiting Visual AI configuration for
  [use case]" indicator, and does not block or fail the underlying WorkItem.
- **FR-VAI-072 — Model version upgraded mid-campaign, changing score distributions.** Because
  `model_version` and `use_case_config_version` are pinned per result (FR-VAI-050), a manager
  comparing week-over-week compliance for an in-flight campaign sees a "model version changed on
  [date]" annotation on any trend chart spanning the change (Analytics-module rendering, sourced
  from this module's `model_versions` catalog); Visual AI does not retroactively rescore historical
  results against the new model version, and does not blend scores across versions in a single
  trend line without that annotation.
- **FR-VAI-073 — Conflicting human reviews on the same result.** See FR-VAI-034
  (`NEEDS_ADJUDICATION`); the edge case additionally covers two reviewers acting concurrently on
  the same `AI_RESULT_PENDING_REVIEW` item — the second reviewer's submission is accepted only if
  the result is still in `PENDING_REVIEW` at write time (optimistic concurrency on the result's
  `version`); a lost race returns `409 CONFLICT` with the first reviewer's decision shown, letting
  the second reviewer decide whether to concur or dispute (which itself becomes the
  `NEEDS_ADJUDICATION` path).
- **FR-VAI-074 — Offline-captured image processed after significant delay.** A photo captured
  offline and synced days later triggers inference on arrival at the server (via the normal
  `evidence.uploaded` event, emitted at sync time, not capture time). The result's `confidence`
  routing and SLA timers (review/appeal) are computed from **server receipt time**, not capture
  time, consistent with the platform-wide rule that client clocks are never authoritative (§19.4).
  However, the UI surfaces the original capture timestamp prominently on the result so a manager
  reviewing a week-old photo is not confused into thinking it reflects current store state; if the
  work item's due window has since closed, the result is still recorded and reported but does not
  reopen a closed WorkItem — it is attached as supplementary evidence.
- **FR-VAI-075 — Duplicate inference request on event replay.** Idempotency handling for
  `evidence.uploaded` redelivery is specified in `low-level-design.md` §6 (idempotent on
  `(evidence_id, use_case_config_id)`); functionally, a redelivered event must never create a
  second `VisualAiResult` for the same evidence/use-case pair nor re-trigger a review/appeal SLA
  timer that is already running.
- **FR-VAI-076 — Violation type not in tenant's configured taxonomy.** If a model backend returns
  a violation `type` value the current `UseCaseConfig` taxonomy does not recognize (e.g., after a
  model upgrade introduces new detection classes), the violation is still stored (not dropped) but
  flagged `unmapped_type = true` and defaults to `severity = MEDIUM` pending Operations Admin
  taxonomy/severity-map update; unmapped violations are excluded from auto-fail eligibility
  (FR-VAI-022) until mapped, since their severity is not yet tenant-confirmed.

## 12. Non-Functional Requirements (module-specific)

- **NFR-VAI-01 — Inference latency.** p95 end-to-end latency from `evidence.uploaded` receipt to a
  `VisualAiResult` reaching a terminal pre-review state (`SCORED`/routed) ≤ 15 seconds for
  synchronous-eligible use cases (single-image classification/detection); ≤ 90 seconds p95 for
  batch/video-frame-sampling use cases. Beyond p99 latency budgets, the UX degrades to the
  asynchronous "still analyzing" state (§10) rather than blocking.
- **NFR-VAI-02 — Review SLA.** Default 24-hour review SLA (FR-VAI-035), tenant-configurable down to
  a documented minimum (recommended floor: 2 hours, to avoid unreviewable queues on short-staffed
  sites) and up to 72 hours.
- **NFR-VAI-03 — Minimum precision/recall gate before auto-fail.** A `UseCaseConfig` may not set
  `auto_fail_enabled = true` unless its most recent rolling-30-day precision and recall (per
  FR-VAI-060) both meet a platform-wide floor: precision ≥ 0.90 and recall ≥ 0.85 for HIGH-severity
  violations specifically (the auto-fail-eligible class). The threshold-editor UI enforces this
  gate at configuration time (attempting to enable auto-fail below the gate returns a
  `422 PRECISION_RECALL_GATE_NOT_MET` with the current metrics shown); an Enterprise Admin may
  override the gate only with an explicit, audit-logged justification (break-glass, logged as a
  distinct AuditLog action).
- **NFR-VAI-04 — Inference queue backpressure.** The inference queue must sustain burst uploads
  from large recurring campaigns (e.g., 100k locations submitting a planogram photo within the
  same 2-hour window) without unbounded queue growth causing review-SLA-breaching delays; see
  `high-level-design.md` §Scalability for the autoscaling and shed-load strategy.
- **NFR-VAI-05 — Availability independent of a single model provider.** Visual AI inference
  availability must not be fully dependent on a single upstream model provider; the
  model-provider-agnostic gateway (conventions) must support a configured fallback backend per use
  case so a provider outage degrades to "queued, delayed" rather than "all inference unavailable"
  (see HLD Failure Modes).

## 13. Dependencies on Other Modules

| Module | Direction | Nature |
|---|---|---|
| Evidence | Consumes | Source of truth for the image/video object; Visual AI is triggered by `evidence.uploaded` and reads media via Evidence's signed-URL access; never stores media itself. |
| Work Orchestration | Bidirectional | A WorkItem's evidence field declares the triggering `visual_ai_use_case`; Visual AI results may gate a WorkItem's `SUBMITTED -> VERIFIED` transition (tenant-configurable) and can trigger a `schedule_reinspection`-class follow-up WorkItem after a failed/overturned result. |
| Issues & CAPA | Produces | A `HIGH`-severity violation on an `AI_RESULT_AUTO_FAILED` or `AI_RESULT_REVIEWED` (confirmed) result triggers automatic Issue creation (tenant-configurable which severities/use cases qualify); Visual AI does not own Issue lifecycle beyond initiating it and cross-referencing on appeal overturn (FR-VAI-044). |
| Agentic Operations | Bidirectional | Hosts the Visual Inspection Agent, which consumes `VisualAiResult`/`Violation` data and Visual AI's APIs as typed tools (e.g., to summarize recurring failures or propose re-inspection) at its configured autonomy level; also owns the model-provider-agnostic gateway and model governance/training pipeline that Visual AI's model backends plug into and that consumes the labeled feedback export (§9). |
| Analytics and Control Tower | Produces | `VisualAiResult`/`ReviewDecision` events feed compliance-score trends, precision/recall dashboards, and the "AI Override Rate" enterprise KPI (source-spec §14.2). |
| Notification Hub | Produces (calls) | Review-queue assignment, SLA breach, and appeal-status notifications routed through Notification Hub. |
| Identity/Org | Consumes | `UserScope`/`Role` for review-queue visibility and appeal-resolver routing by location scope; `OrganizationNode` parent walk for appeal escalation (FR-VAI-042). |

## 14. Resolved Decisions

- **Default review-queue sort**: ascending confidence (least-confident-first), platform-wide default,
  because it directs reviewer attention to the results most likely to need correction rather than
  processing arbitrary arrival order. Tenant-configurable to FIFO/oldest-first for a tenant that
  prioritizes strict SLA-driven ordering over review-accuracy ordering.
- **Mandatory dual review (FR-VAI-034)**: not platform-mandated for any use case at MVP; dual review
  is tenant-configurable per use case, defaulting to **off**. Even for regulated categories (e.g.
  safety/PPE), the platform does not hard-code a regulatory dual-review requirement into the product
  because regulatory review requirements vary by jurisdiction and industry in ways a single global
  default cannot correctly encode; a tenant operating under a regulation that mandates dual review
  configures it as a tenant policy for the affected use case(s), and Security/Compliance module (13)
  tenant compliance profiles are the mechanism for surfacing that obligation to the tenant admin.
- **Appeal depth (FR-VAI-040)**: fixed at a single appeal per result for MVP; there is no second-level
  appeal path. A second-level appeal is deferred to Phase 2 and will be scoped using MVP appeal-volume
  and overturn-rate data rather than being speculatively designed now.
- **Pre-inference image-quality gating (FR-VAI-070)**: runs as a lightweight step inside the same
  inference queue/worker, not as a separate service, avoiding an extra network hop before every
  inference call. This is the permanent architecture, not a placeholder; it would only be revisited if
  a specialized third-party quality-classifier vendor were adopted in place of the in-house check.
- **`AI_RESULT_UNREVIEWED` (source-spec §12.2) backward-compatibility alias**: retained permanently as
  an alias status, mapped internally to whichever of `AUTO_PASSED`/`AUTO_FAILED`/`PENDING_REVIEW`
  applies, so any integration built directly against the source contract keeps working. New consumers
  must read the more specific status values; the alias is not deprecated on any current roadmap
  because removing it would be a breaking change with no compensating benefit.
- **Retraining feedback export data-sharing scope (FR-VAI-061)**: strictly single-tenant only through
  MVP and Phase 2 (§9, FR-VAI-062) — no cross-tenant "contribute to shared base model" opt-in exists
  until a formal data-sharing/marketplace program is built. This is the firm scope boundary because a
  cross-tenant model-contribution feature requires its own data-use consent flow, contractual terms,
  and (per engineering-decisions.md §12) AI-governance review before it can exist at all; it ships, if
  it ships, alongside the Phase 3 Agent Marketplace roadmap item, not before.
