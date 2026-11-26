# Learning — Functional Specification

Module corresponds to **EPIC-09 (Learning)**. Source: source-spec.md §11.3 (Learning), §13.1 (Agent
Portfolio — Training Agent), §18.3 (`training.assigned`/`training.completed` events), §7.1 (Training
Assignment WorkItem type), §5 (PRD Capability Map — Learning: **Basic in MVP, full in Phase 2**),
Appendix E EPIC-09.

## 0. Priority Tagging Convention

Per the PRD Capability Map (source-spec §5), Learning ships as **Basic** in MVP and **Full** in
Phase 2. Every requirement below is tagged:

- **[MVP]** — must exist for initial GA.
- **[P2]** — Phase 2 scope; MVP schema/APIs must not preclude it (additive, non-breaking).

MVP scope is intentionally narrow: link/document/video content, manual + role-based + failure-
triggered assignment, a basic knowledge-check quiz, certification with expiry, and the full
repeated-failure -> microlearning -> knowledge-check -> re-inspection -> measured-improvement loop
in its simplest form (the loop is explicitly called out in source-spec §11.3 and is core
differentiation, so it is **not** deferred to Phase 2 even though richer authoring is). Phase 2 adds
richer interactive/SCORM-class content, branching learning paths, and deeper outcome analytics.

## 1. Purpose and Scope

### 1.1 Purpose
The Learning module closes the loop between operational failure and workforce capability. It turns
training into a governed, trackable `WorkItem` type (Training Assignment, source-spec §7.1) with
outcome linkage: every assignment, completion, and certification exists to be measured against
downstream operational results (repeat-failure rate, verified-execution rate), not merely tracked as
"content consumed." Learning is the mechanism behind the product promise "every repeated exception
can become an automated intervention... every intervention can be measured for effectiveness"
(source-spec §2.2) when the intervention is a skills/knowledge gap.

### 1.2 In Scope
- Course and course-version authoring: **[MVP]** link-out to external content (URL) or upload of a
  simple document/video asset; **[P2]** richer interactive/multi-lesson modules, SCORM/xAPI-class
  packages, branching content.
- Assignment models: manual (by an admin/manager to a user, role, or location), role-based (assign
  to a role at a location or org subtree), and failure-triggered (system/agent-generated in response
  to a repeated-failure signal from Issues/CAPA or Analytics).
- Knowledge-check quiz mechanics: question bank, pass threshold, retry policy, scoring.
- Certification: definition, issuance on qualifying completion, expiry, renewal reminders, and
  (tenant-policy-gated) blocking of safety-critical work when a required certification has lapsed.
- The repeated-failure -> identify gap -> assign microlearning -> knowledge check -> re-inspection ->
  measure improvement loop (source-spec §11.3), including explicit trigger conditions and the
  outcome-measurement definition.
- Offline access to assigned training content on the frontline mobile app (content caching, offline
  quiz attempt, deferred completion sync).
- The `LearningAssignment` lifecycle and its projection as a `TRAINING_ASSIGNMENT` WorkItem.
- Training Agent (source-spec §13.1) behavior specific to Learning: mapping a detected failure
  pattern to a targeted course/microlearning unit and creating the assignment (default autonomy
  **L1 Recommend** in MVP; **L2 Execute low-risk** eligible in Phase 2 for narrowly-scoped,
  pre-approved course-to-failure mappings).

### 1.3 Out of Scope (owned by other modules, referenced here)
- The `WorkItem` state machine, scheduling, SLA/escalation mechanics — **Work Orchestration module
  (EPIC-03)**. Learning owns `LearningAssignment` as the Learning-domain record; Work Orchestration
  owns the `TRAINING_ASSIGNMENT` WorkItem wrapper that represents it on the frontline Today screen.
- Failure/repeat-failure detection logic itself (pattern thresholds, root cause) — **Issues & CAPA
  module (EPIC-07)** and **Analytics module (EPIC-15)**. Learning consumes their signals
  (`issue.resolved`, repeat-failure pattern events) as assignment triggers; it does not compute
  repeat-failure statistics itself.
- Re-inspection scheduling and execution mechanics — **Work Orchestration** (the re-inspection is a
  normal `AUDIT_INSPECTION` WorkItem). Learning owns the *linkage* (which re-inspection corresponds
  to which assignment) and the before/after outcome comparison.
- General knowledge document search/retrieval — **Knowledge module (EPIC-08, source-spec §11.2)**.
  A course may *link to* a `KnowledgeDocument` as content, but document versioning/search/RAG
  retrieval is owned there.
- Reminder/notification delivery mechanics — **Communications / Notification Hub**. Learning decides
  *when* a reminder is due (certification nearing expiry, assignment overdue) and calls the
  notification interface; it does not own channels, preferences, or delivery retries.
- Offline local-DB sync protocol mechanics (conflict resolution framework, sync token, tombstones) —
  **Mobile/Offline module (EPIC-05)**. Learning defines what content/records are cached and the
  completion-sync payload shape; the generic sync transport is owned there.
- Agent Runtime mechanics (context building, policy engine, model abstraction, approval inbox) —
  **Agent Runtime module (EPIC-11)**. Learning defines the Training Agent's domain-specific tools and
  triggers only.

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Frontline Associate (learner) | Views assigned training on Today screen / a "My Learning" surface; completes content and knowledge checks, including offline; sees certification status |
| Store/Site Manager | Manually assigns training to individuals/roles at their location; monitors team completion and overdue/lapsed-certification status; sees which assignments were failure-triggered and why |
| Operations Admin | Authors courses and course versions; defines certifications (validity period, renewal rule, required roles); defines the failure-to-course mapping rules used by the Training Agent; reviews outcome-linkage reporting |
| Training Agent (AI Agent) | Observes repeated-failure signals; maps a failure pattern to a targeted microlearning course; creates a failure-triggered `LearningAssignment`; schedules the re-inspection linkage; default autonomy L1 Recommend (MVP), L2 eligible for pre-approved mappings (P2) |
| Regional/District Manager | Views rollup of certification compliance and training-to-outcome improvement across territory (Analytics-fed) |

## 3. Course and Content Authoring

- **FR-LEARN-001 [MVP].** A `Course` has a stable logical identity (`id`, `title`, `tenant_id`,
  `status`) and one or more immutable `CourseVersion` rows, mirroring the Template/TemplateVersion
  pattern used platform-wide (source-spec §17.1, §8.3). Publishing a new `CourseVersion` never
  mutates a prior version; in-progress assignments continue against the version they were assigned
  against (see Validation, §10).
- **FR-LEARN-002 [MVP].** MVP content types per `CourseVersion`: `EXTERNAL_LINK` (URL to
  externally-hosted content, opens in an in-app browser/webview), `DOCUMENT` (uploaded PDF/slide
  asset via Evidence-style object storage), `VIDEO` (uploaded video asset, transcoded to an
  adaptive-bitrate offline-cacheable format). Exactly one primary content type per `CourseVersion`
  plus an optional attached knowledge-check (§4).
- **FR-LEARN-003 [P2].** Phase 2 adds `INTERACTIVE_MODULE` (multi-lesson, branching, SCORM/xAPI-class
  package import) and multi-part courses composed of ordered `CourseUnit` sub-objects, each
  independently completable and offline-cacheable. The MVP schema's `course_versions.content_type`
  enum and `content_ref` JSONB are designed to add this value without a breaking migration
  (§ low-level-design.md schema notes).
- **FR-LEARN-004 [MVP].** Course authoring fields: `title`, `description`, `category`, `duration_estimate_minutes`,
  `language`, `audience_scope` (role(s) + org-node subtree, reusing the `UserScope`/`OrganizationNode`
  scoping model), `requires_knowledge_check` (bool), `passing_score_pct`, `is_certification_granting`
  (bool, links to §5), `tags` (used by the Training Agent's failure-to-course mapping, §7).
- **FR-LEARN-005 [MVP].** `CourseVersion` lifecycle mirrors `TemplateVersion` (source-spec §17.4):
  `DRAFT -> REVIEW -> APPROVED -> PUBLISHED -> SUPERSEDED -> RETIRED`. Only `PUBLISHED` versions are
  assignable. `RETIRED` versions cannot receive new assignments but remain readable for
  already-assigned/completed learners (historical truth, source-spec P7).
- **FR-LEARN-006 [MVP].** Content version update while an assignment is in progress: updating/
  publishing a new `CourseVersion` does not retroactively change an already-created
  `LearningAssignment`'s bound `course_version_id`. Admins may optionally trigger a bulk
  "re-issue to latest version" action for open assignments, which creates a superseding assignment
  and cancels the original with an audit trail (never a silent content swap under an in-progress
  learner).

## 4. Knowledge-Check Quiz Mechanics

- **FR-LEARN-010 [MVP].** A knowledge check is an ordered set of `QuizQuestion` rows attached to a
  `CourseVersion`: `question_text`, `question_type` (`SINGLE_SELECT`, `MULTI_SELECT`, `TRUE_FALSE` in
  MVP; `SHORT_ANSWER` free-text with manual grading, and matching/ordering types in **[P2]**),
  `options[]`, `correct_option_ids[]`, `points` (default 1).
- **FR-LEARN-011 [MVP].** A `QuizAttempt` records one learner's attempt: selected answers, computed
  `score_pct`, `passed` (bool, `score_pct >= course_version.passing_score_pct`), `started_at`,
  `submitted_at`, `attempt_number`. Default `passing_score_pct` = 80% if not set at course level;
  tenant/course-configurable.
- **FR-LEARN-012 [MVP].** Retry policy: default max 3 attempts per assignment, tenant/course
  configurable (including `unlimited`). On exhausting max attempts without passing, the assignment
  transitions to `FAILED` (§9 lifecycle) and is visible to the manager as an exception requiring
  intervention (reassign, extend attempts, or escalate) rather than silently disappearing.
- **FR-LEARN-013 [MVP].** Question order and, optionally, answer-option order are randomized
  per-attempt (`randomize_questions`, `randomize_options` flags, default true) to reduce
  answer-sharing between learners completing the same failure-triggered assignment.
- **FR-LEARN-014 [MVP].** Quiz retry UX: on fail, the learner sees `score_pct`, pass threshold, and
  (tenant-configurable `show_correct_answers` flag, default **false** for failure-triggered
  compliance-sensitive courses to preserve re-test validity) which questions were missed. A "Retry"
  action is presented if attempts remain; if content re-review is required before retry
  (`require_content_reread_before_retry`, default true for failure-triggered assignments), the
  content-viewed gate (§ FR-LEARN-030) resets and must be satisfied again before the retry unlocks.
- **FR-LEARN-015 [P2].** Adaptive follow-up questions and per-question remediation content
  (show a short clarifying content snippet immediately after a missed question) are Phase 2.

## 5. Certification

- **FR-LEARN-020 [MVP].** A `Certification` is a definition (`name`, `required_course_version_id` or
  a set of qualifying course versions, `validity_period_days`, `applicable_roles[]`,
  `is_safety_critical` bool). Completing a qualifying course (and passing its knowledge check, if
  `requires_knowledge_check`) issues a `CertificationRecord` for the user with `issued_at`,
  `expires_at = issued_at + validity_period_days`.
- **FR-LEARN-021 [MVP].** Renewal reminders are scheduled at tenant-configurable offsets before
  `expires_at` (default: 30 days, 7 days, 1 day before, and on the day of expiry) via the
  Communications/Notification Hub interface (§13 Dependencies). Reminder scheduling uses Temporal
  durable timers per the platform scheduling convention (consistent with Work Orchestration's
  SLA-timer approach), not application-level polling, so reminders survive process restarts.
- **FR-LEARN-022 [MVP].** Certification expiry enforcement is **policy-gated**, not hardcoded: a
  tenant sets `expiry_policy ∈ {REPORT_ONLY, WARN, BLOCK_NEW_WORK}` per `Certification`.
  `BLOCK_NEW_WORK` prevents new `WorkItem` assignment/claim for WorkItem types tagged as requiring
  that certification (enforced by Work Orchestration at assignment/claim time via a
  certification-status check call into Learning) but never retroactively invalidates already
  in-progress or already-submitted work (see Validation §10 for the safety-critical open-work edge
  case).
- **FR-LEARN-023 [MVP].** Renewal is a new `CertificationRecord` created by completing the same or a
  designated renewal `Course`; the prior record is not deleted (append-only certification history
  for audit, consistent with the platform's historical-truth principle P7).
- **FR-LEARN-024 [P2].** Multi-tier certification (e.g., Level 1 -> Level 2 prerequisite chains) and
  cross-tenant/industry-recognized certification import are Phase 2.

## 6. Assignment Models

- **FR-LEARN-030 [MVP].** Manual assignment: an admin/manager with `learning:assign` permission at
  the relevant scope assigns a published `CourseVersion` to a specific user, a role at a location/org
  subtree, or an org subtree broadly (mass-assignment fan-out, see high-level-design.md
  Scalability). Creates one `LearningAssignment` per resolved target user.
- **FR-LEARN-031 [MVP].** Role-based assignment: assigning to a role at a node resolves the current
  `UserScope` holders at assignment time (a snapshot list, not a live query) and additionally
  registers a "standing assignment rule" so users who *later* acquire that role/location within a
  configurable window (default 90 days) are automatically assigned the same course
  (`auto_assign_on_role_grant=true`, mirroring Work Orchestration's FR-WORK-025 role-triggered
  pattern). This is how new-hire/role-onboarding training is modeled without a separate subsystem.
- **FR-LEARN-032 [MVP].** Failure-triggered assignment: created by the Training Agent (or, if the
  agent is disabled/unconfigured for a tenant, a deterministic rule-engine fallback) in response to a
  repeated-failure signal. See §7 for the full loop and explicit trigger conditions.
- **FR-LEARN-033 [MVP].** Every `LearningAssignment` records `origin ∈ {MANUAL, ROLE_STANDING,
  FAILURE_TRIGGERED}` and, for `FAILURE_TRIGGERED`, an `origin_ref` (the triggering `Issue.id` or
  repeat-failure pattern ID) so the causal chain is auditable end-to-end from failure to training to
  re-inspection outcome.
- **FR-LEARN-034 [MVP].** A `LearningAssignment` is projected into Work Orchestration as a
  `TRAINING_ASSIGNMENT` WorkItem (source-spec FR-WORK-005) for Today-screen visibility, due-date/SLA
  handling, and manager Exceptions surfacing; Learning remains the system of record for the
  learning-domain fields (quiz attempts, certification linkage), Work Orchestration for the
  generic work lifecycle wrapper. The two are kept in sync via `training.assigned`/
  `training.completed` events (§18.3), not a shared table.

## 7. The Repeated-Failure -> Microlearning -> Knowledge-Check -> Re-Inspection -> Measured-Improvement Loop

This is the loop named explicitly in source-spec §11.3 and is Learning's core differentiator; it
ships in MVP even though richer content authoring does not.

```
REPEATED FAILURE -> IDENTIFY GAP -> ASSIGN MICRO-LEARNING -> KNOWLEDGE CHECK -> RE-INSPECTION -> MEASURE IMPROVEMENT
```

- **FR-LEARN-040 [MVP] — Trigger conditions.** A failure-triggered assignment is eligible to fire
  when **all** of the following hold (evaluated by the Training Agent / fallback rule engine on
  receipt of an `issue.resolved` event or an Analytics repeat-failure pattern event):
  1. The same `(failure_category, location_id or user_id, template_field or check_id)` combination
     has occurred **N times within a rolling window W** (tenant-configurable; default N=3 occurrences
     within 30 days at the same location, or N=2 occurrences by the same user across any location
     within 30 days — the user-level path targets a skills gap, the location-level path targets a
     process/equipment gap and routes to Issues/CAPA rather than Learning if no user-level pattern
     exists).
  2. A `FailureToCourseMapping` rule exists connecting the `failure_category`/`tags` to a specific
     published `CourseVersion` (Operations Admin-authored mapping table, keyed by
     `failure_category -> course_id`, with an optional `location_type`/`role` qualifier). If no
     mapping exists, the agent escalates as an **L1 Recommend** "no mapping found, suggest creating
     one" insight rather than silently failing.
  3. The target user does not already have an **open** (`ASSIGNED`/`IN_PROGRESS`) assignment for the
     same course from the same `origin_ref` chain (dedupe — see Validation §10 for the
     overlapping-trigger edge case).
- **FR-LEARN-041 [MVP] — Identify gap.** "Gap identification" in MVP is the deterministic mapping in
  FR-LEARN-040.2 (rule-table lookup). Phase 2 (**[P2]**) enriches this with a Compliance Monitor
  Agent-derived root-cause classification (source-spec §13.1) so the mapping can be
  root-cause-aware (e.g., distinguish "did not know the procedure" from "knew but rushed") rather
  than purely failure-category-keyed.
- **FR-LEARN-042 [MVP] — Assign micro-learning.** The Training Agent calls its `schedule_training`
  tool (source-spec §20.2) to create a `LearningAssignment` with `origin=FAILURE_TRIGGERED`,
  `origin_ref` = the triggering Issue/pattern, a short due-date SLA (default 7 days, tenant
  configurable, shorter than standard role-based training since it is remediation), and
  `is_targeted_microlearning=true` for reporting/analytics segmentation.
- **FR-LEARN-043 [MVP] — Knowledge check.** The assigned course's knowledge check (§4) must be
  passed for the assignment to reach `COMPLETED`; a failure-triggered assignment defaults
  `requires_knowledge_check=true` even if the base course does not require it for standard
  (non-remediation) assignees — remediation always verifies retained understanding, not just content
  viewed.
- **FR-LEARN-044 [MVP] — Re-inspection.** On `LearningAssignment` reaching `COMPLETED`, the module
  calls Work Orchestration's `schedule_reinspection` tool contract (source-spec §20.2) to create a
  follow-up `AUDIT_INSPECTION` WorkItem targeting the same `(location_id, check/template_field)` that
  produced the original failure, within a configurable window (default 14 days). The
  `LearningAssignment.reinspection_work_item_id` FK links the two records.
- **FR-LEARN-045 [MVP] — Measure improvement.** "Improvement" is defined precisely (not left
  qualitative) as: the re-inspection's result for the linked check **passes** where the original
  triggering occurrence **failed**, recorded as an `OutcomeMeasurement` row
  (`baseline_result=FAIL`, `followup_result`, `improved` bool, `measured_at`). Aggregate
  outcome-linkage reporting (the PRD success signal "Training-to-outcome improvement," source-spec
  §5) rolls these up as: improved re-inspections / total measured re-inspections, segmented by
  course, failure category, location, and time period. See low-level-design.md for the
  before/after comparison job.
- **FR-LEARN-046 [MVP] — Re-inspection fails (no improvement).** If the linked re-inspection also
  fails the same check, `OutcomeMeasurement.improved=false`, and the system does **not** auto-loop
  indefinitely into another identical microlearning assignment (which would indicate the gap is not
  purely knowledge-based). Instead it raises the case to the Compliance Monitor Agent /
  Issues-CAPA module as an escalation candidate (possible process, equipment, or capacity root cause
  rather than a training gap), consistent with source-spec JTBD "understand whether a recurring
  failure is caused by people, process, equipment, capacity, or policy."

## 8. Offline Access to Assigned Training

- **FR-LEARN-050 [MVP].** Assigned training content (course metadata, `DOCUMENT`/`VIDEO` binary, and
  the knowledge-check question bank) is included in the mobile app's bounded offline cache set
  (source-spec §9.2, §19.1 `templates`/`template_versions`-equivalent local tables:
  `learning_assignments`, `course_content_cache`). Only content for a user's **currently open**
  assignments is proactively cached; a course library browse/search experience (if any) is
  online-only in MVP.
- **FR-LEARN-051 [MVP].** A learner can view cached content and complete a knowledge check fully
  offline. `QuizAttempt` rows are created locally with a `client_uuid` and synced on reconnect using
  the same idempotent-submission pattern as `Submission` (source-spec §19.2).
- **FR-LEARN-052 [MVP].** `EXTERNAL_LINK` content type is **not** guaranteed available offline (it is
  a webview to an external URL); the assignment detail clearly labels link-based content as
  "requires connectivity" so learners are not surprised offline. `DOCUMENT`/`VIDEO` content types are
  the offline-capable types in MVP by design.
- **FR-LEARN-053 [MVP].** Offline content caching limits: default per-device cap of 500 MB or the 10
  most-recently-assigned items' content (whichever is smaller), tenant-configurable; video content is
  capped to a mobile-optimized bitrate/resolution profile (target ≤ 15 MB per 5 minutes of video) to
  keep the sync size budget realistic on constrained frontline devices/networks (see NFR-LEARN-01).
  When the cap would be exceeded, oldest-assigned non-in-progress content is evicted first; an
  in-progress item is never evicted mid-attempt.

## 9. LearningAssignment Lifecycle and UX States

The `LearningAssignment` lifecycle (detailed state diagram in architecture-diagram.md):
`ASSIGNED -> IN_PROGRESS -> COMPLETED -> VERIFIED | FAILED -> (re-assigned)`. `VERIFIED` applies only
to failure-triggered assignments once the linked re-inspection outcome is recorded; assignments with
no re-inspection linkage (most manual/role-based training) terminate at `COMPLETED`.

| State | UX / Behavior |
|---|---|
| Assigned-training badge | A badge count on the frontline "Today"/"My Learning" surface for open (`ASSIGNED`/`IN_PROGRESS`) assignments; failure-triggered assignments are visually distinguished (e.g. a "targeted retraining" tag) so the learner understands the remediation context, not presented as generic training. |
| Overdue-certification blocking state | Governed by `expiry_policy` (§ FR-LEARN-022): `REPORT_ONLY` shows a passive banner only; `WARN` shows a blocking-looking but dismissible banner on relevant WorkItem types plus manager notification; `BLOCK_NEW_WORK` prevents claiming/starting new WorkItems of the gated type with an explicit "Certification required — renew to continue" screen linking directly to the renewal course. |
| Quiz retry flow | Attempt -> immediate scored result -> pass (assignment progresses) or fail-with-attempts-remaining (retry CTA, optional forced content re-review per FR-LEARN-014) or fail-exhausted (assignment -> `FAILED`, manager-visible exception). |
| Offline video/content caching | Download progress indicator on assignment; "available offline" checkmark once fully cached; explicit "requires connection" state for `EXTERNAL_LINK` items and for content evicted under the cache cap (§ FR-LEARN-053), with a one-tap re-download action when connectivity returns. |
| Empty state | "My Learning" with zero assignments shows a neutral empty state, distinct from a loading/sync-in-flight state, consistent with the platform's Today-screen empty-state convention (Work Orchestration FR-WORK-070/UX table). |
| Certification status view | Per-user list of held certifications with `issued_at`/`expires_at`/status chip (`ACTIVE`, `EXPIRING_SOON`, `EXPIRED`); manager rollup view aggregates by team/location for compliance monitoring. |

## 10. Validation and Edge Cases

- **FR-LEARN-060.** **Training assigned to a user who leaves the role before completion.** On a
  `user.role_changed`/`user.deactivated` event (Identity/Org) affecting a user with open
  assignments: if the assignment's eligibility was role-derived (`origin=ROLE_STANDING`) and the user
  no longer holds the qualifying role, the assignment is auto-cancelled with reason
  `ROLE_NO_LONGER_APPLICABLE` (not silently orphaned) and, if it was `auto_assign_on_role_grant`
  bound, removed from that binding. Manually-assigned (`origin=MANUAL`) or failure-triggered
  (`origin=FAILURE_TRIGGERED`) assignments are **not** auto-cancelled on role change alone — a
  targeted remediation follows the person, since the underlying skills gap is personal, not
  role-scoped — but are flagged for manager review if the person also changed location/tenant
  standing. On full deactivation, all open assignments (any origin) are closed with reason
  `USER_DEACTIVATED`, mirroring Work Orchestration's FR-WORK-045 pattern, and historical
  `QuizAttempt`/`CertificationRecord` rows are preserved for audit (never deleted).
- **FR-LEARN-061.** **Certification lapses while user has open safety-critical work.** An
  already-`IN_PROGRESS` or `SUBMITTED` WorkItem is never retroactively invalidated by a certification
  lapsing mid-execution (consistent with platform historical-truth/immutability principle). However,
  if `Certification.is_safety_critical=true` and `expiry_policy=BLOCK_NEW_WORK`, the system: (a)
  immediately notifies the user's manager (not just the user) that a safety-critical certification
  has lapsed with an open item in flight, (b) prevents the user from being assigned *new*
  safety-critical work of that type until renewed, and (c) does not auto-remove the user from the
  in-flight item (manager decides whether to reassign) — this favors explicit human judgment over
  automatic interruption of active safety-critical work, per source-spec P5 (policy before AI /
  deterministic-but-human-governed defaults).
- **FR-LEARN-062.** **Repeated-failure trigger firing multiple overlapping assignments.** Guarded at
  three levels: (1) the dedupe check in FR-LEARN-040.3 (no duplicate open assignment for the same
  course from the same `origin_ref` chain); (2) a broader per-user cooldown
  (`max_concurrent_failure_triggered_assignments`, default 2) so a user experiencing several
  different failure categories simultaneously is not overwhelmed — additional eligible triggers queue
  as `PENDING_CAPACITY` rather than firing immediately, dispatched FIFO as active ones complete; (3)
  idempotent trigger processing keyed on `(origin_ref, course_id)` so event redelivery
  (e.g. `issue.resolved` redelivered) cannot create a second assignment.
- **FR-LEARN-063.** **Content version updated after assignment already in progress.** Per
  FR-LEARN-006: the in-progress assignment's `course_version_id` is fixed at assignment time and does
  not change; the learner completes against that version even if superseded. An
  admin-initiated "re-issue to latest" bulk action is the only path to move an open assignment to a
  new version, and it is explicit and audited, never automatic.
- **FR-LEARN-064.** **Quiz attempt submitted offline against a since-retired course version.** The
  server accepts the sync using the `course_version_id` recorded on the local `LearningAssignment` at
  assignment time (immutable reference), regardless of the course's current published state —
  identical principle to Work Orchestration's FR-WORK-095 stale-template-version handling.
- **FR-LEARN-065.** **Re-inspection cannot be scheduled (e.g. location closed, check retired).** The
  `LearningAssignment` remains `COMPLETED` (not blocked indefinitely) with
  `reinspection_status=UNSCHEDULABLE` and a reason; outcome measurement for that assignment is
  excluded from the improvement-rate denominator rather than counted as a non-improvement.
- **FR-LEARN-066.** **Duplicate offline completion/quiz-attempt sync.** Idempotent on
  `client_uuid`, identical pattern to platform-wide submission idempotency (source-spec §19.2/§19.4)
  — a retried sync returns the original server outcome, never a duplicate `QuizAttempt` or duplicate
  `training.completed` event emission.

## 11. Non-Functional Requirements (module-specific)

- **NFR-LEARN-01 — Offline content sync size budget.** Default per-sync-cycle content download
  budget of 50 MB on a standard reconnect (aligned with the platform's general Sync recovery NFR
  target of p95 < 30s for standard payloads, source-spec §23); larger video assets download via
  resumable, deferrable background transfer (WorkManager/BGTaskScheduler) rather than blocking the
  main sync cycle, consistent with the platform's "large media uploaded/downloaded separately with
  resumable semantics" rule (source-spec §19.4).
- **NFR-LEARN-02 — Completion-tracking latency.** A `training.completed` event (online) is emitted
  and the corresponding `TRAINING_ASSIGNMENT` WorkItem/certification status updated within p95 ≤ 5
  seconds of quiz submission, matching the platform-wide WorkItem update latency target
  (Work Orchestration NFR-WORK-01). Offline completions sync on next reconnect with no hard latency
  bound, per the Offline Design Rule (source-spec §9.2).
- **NFR-LEARN-03 — Mass-assignment fan-out throughput.** A role/org-subtree assignment targeting up
  to 100k users (source-spec §23 scale target) must complete `LearningAssignment` row creation and
  `training.assigned` event emission within a bounded batch-processing window (target: complete
  within 15 minutes for 100k targets) without blocking the initiating admin's request past
  acknowledgement (async job, admin sees progress, not a synchronous 100k-row transaction).
- **NFR-LEARN-04 — Outcome-measurement job latency.** The pre/post re-inspection comparison job
  (§ low-level-design.md) evaluates a completed re-inspection against its linked baseline within 1
  hour of the re-inspection WorkItem reaching `VERIFIED`, so outcome-linkage reporting is
  near-real-time rather than a nightly-only batch.

## 12. Dependencies on Other Modules

| Module | Direction | Nature |
|---|---|---|
| Issues & CAPA | Consumes | `issue.resolved` events and repeat-failure pattern signals as failure-triggered assignment inputs (§7); Learning escalates back to Issues/CAPA when re-inspection shows no improvement (FR-LEARN-046) |
| Work Orchestration | Bidirectional | Learning produces `TRAINING_ASSIGNMENT` WorkItem records (via `training.assigned`) and consumes WorkItem completion/re-inspection outcome signals; certification-status gating is enforced by Work Orchestration at claim/assignment time via a Learning status check |
| Analytics | Produces / Consumes | Produces outcome-linkage facts (`OutcomeMeasurement`, completion/certification events) for the Training-to-outcome-improvement KPI (source-spec §5, §26.2); consumes repeat-failure pattern signals as an alternate trigger source to Issues/CAPA direct events |
| Communications | Produces (calls) | Certification-expiry reminders and overdue-assignment nudges routed through the Notification Hub interface, not delivered directly by Learning |
| Agentic Operations (Agent Runtime) | Bidirectional | Hosts the Training Agent; Learning exposes `schedule_training`-class typed tool semantics and the `FailureToCourseMapping` rule table the agent reads; consumes `agent.action_approved`/`agent.tool_executed` audit signals for failure-triggered assignments the agent created |
| Mobile / Offline | Consumes | Offline content caching, local quiz-attempt storage, and idempotent completion sync ride on the platform's generic offline local-DB and sync-protocol infrastructure |
| Knowledge | Consumes | A `CourseVersion` may reference a `KnowledgeDocument` as underlying content; Knowledge owns document versioning/search |
| Identity/Org | Consumes | `OrganizationNode`/`UserScope`/`Role` for audience scope resolution, role-standing assignment eligibility, and role-change/deactivation triggers |

## 13. Resolved Decisions

- **Rolling-window parameters for the repeated-failure trigger (FR-LEARN-040.1)**: fixed at **N=3
  occurrences / 30 days** at location level and **N=2 occurrences / 30 days** at individual-user
  level. These are platform defaults (not pilot-only placeholders) chosen to fire on a genuinely
  repeating pattern rather than isolated incidents, while being tight enough (30-day window) that a
  remediation assignment still lands close enough to the failures to be instructionally relevant.
  Tenant admins may tune both the count and window per `FailureToCourseMapping` rule (the platform
  default applies where a rule does not override it); no further pilot validation is a precondition
  for GA — the platform's usual post-launch tuning process (Analytics module dashboards on
  trigger-to-improvement rate) covers ongoing calibration.
- **`EXTERNAL_LINK` content for failure-triggered/safety-critical courses (FR-LEARN-052)**: blocked.
  `EXTERNAL_LINK` is disallowed as the sole content type for any course flagged
  `is_certification_granting=true` or referenced by a `FailureToCourseMapping` rule; course authoring
  validation rejects publishing such a course with only `EXTERNAL_LINK` content. `DOCUMENT`/`VIDEO`
  are required for anything that must be completable offline to meet a compliance deadline, since a
  webview-only course cannot be guaranteed available to a frontline user without connectivity.
- **Course catalog/browse UX**: out of scope for MVP. MVP is assignment-driven only — a learner sees
  only what has been assigned to them; there is no self-directed catalog browsing surface. Self-
  service catalog browsing is deferred to Phase 2; this is a scope boundary, not an ambiguity, and
  requires no further decision to build against.
- **Training Agent default autonomy (failure-triggered assignments)**: fixed at **L1 Recommend** for
  MVP, platform-wide default. The agent's proposed assignment always requires manager confirmation
  before `training.assigned` fires, unless a tenant explicitly opts a specific
  `FailureToCourseMapping` rule into **L2 Execute low-risk** auto-assignment (opt-in, per-rule, never
  a blanket tenant-level default). This matches source-spec §13.1's stated default autonomy for the
  Training Agent and keeps human confirmation in the loop until per-mapping-rule precision is proven
  out; a tenant may loosen it per rule once its own data supports doing so.
- **Certification renewal path**: MVP requires completing the renewal course/quiz; a passed
  re-inspection alone never satisfies renewal in MVP, even when it demonstrates the same skill in the
  field, because the renewal course is also the vehicle for delivering any regulatory/policy content
  that changed since the original certification (a re-inspection only tests the practical skill, not
  awareness of updated policy). Field-demonstrated-skill-as-renewal is deferred to Phase 2.
- **`QuizQuestion` authorship**: tenant-authored only in MVP; no AI-generated quiz content ships in
  MVP. AI-assisted quiz-question generation from course content is a Phase 2 Agent Runtime capability
  and is explicitly out of this module's MVP scope, keeping human review in the loop for all
  assessment content used to gate certifications.
