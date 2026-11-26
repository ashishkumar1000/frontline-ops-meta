# Gamification & Recognition — Functional Specification

Module corresponds to a **new Tier-3 module** per `new-modules-addendum.md` (row 22, shortcode
`GAM`). Traces loosely to **EPIC-09 (Learning)**, **EPIC-07/EPIC-03** (streak/completion signals from
Issues-CAPA-adjacent Work Orchestration), and the new **EPIC-18-class Employee Engagement** module —
Gamification does not introduce its own Epic in Appendix E; it is tracked as **EPIC-22 (new,
Gamification & Recognition)** for traceability purposes going forward. Source: `new-modules-addendum.md`
row 22 ("badges/pathways layer on Learning module 08's Course/Certification; Employee Engagement
module 18's recognition events can award points"), `08-learning/functional-spec.md` (Course/
CourseVersion/Certification/LearningAssignment model this module layers on), source-spec.md §5 (PRD
Capability Map — Learning), §11.3 (Learning loop), §26.2 (Engagement metrics: training completion,
announcement acknowledgement).

## 0. Priority Tagging Convention

- **[MVP]** — must exist for initial GA of this module.
- **[P2]** — Phase 2 scope; MVP schema/APIs must not preclude it (additive, non-breaking).

This module is explicitly **Tier-3**: a lighter, cross-cutting UX/engagement layer, not a new core
operational domain. It ships after the Tier-1/Tier-2 modules it depends on (Learning, Work
Orchestration, Employee Engagement, Analytics) are live, since it has no independent value without
their events to react to. MVP scope here is deliberately narrow — badges, a points ledger, one
leaderboard scope, and pathway progress visualization — with richer pathway authoring, cross-tenant
benchmarking, and reward redemption deferred to Phase 2.

## 1. Purpose and Scope

### 1.1 Purpose

Gamification & Recognition adds a motivational, visible-progress layer on top of work the platform
already tracks: completed audits, finished courses, task-completion streaks, and peer recognition. It
does not change what "correct execution" means anywhere in the platform — it makes correct execution
and skill growth *feel* rewarding and *visible* (badges, points, leaderboards, pathway progress),
which the product's Engagement KPI set (source-spec §26.2: training completion, announcement
acknowledgement) already treats as a first-class signal of a healthy frontline deployment. The module
exists to increase Frontline Adoption and Verified Operational Execution Rate indirectly, by making
good behavior socially and visibly rewarding, never by weakening evidence, verification, or audit
integrity elsewhere in the system.

### 1.2 In Scope

- **Badge definition & event-driven award rules** [MVP]: an Operations Admin defines a `Badge` and one
  or more `BadgeRule`s that describe the achievement condition (e.g. "complete 10 audits with zero
  violations in a rolling 90 days," "complete the onboarding Pathway," "reach a 30-day task-completion
  streak"). Rules are evaluated against events consumed from other modules, never by Gamification
  polling or recomputing other modules' domain state.
- **Points ledger** [MVP]: an append-only, source-attributed transaction log of points earned (and, when
  a triggering event is later invalidated, reversed) — not merely a running balance. Sources include
  task-completion streaks (Work Orchestration), course/certification completion (Learning), and
  recognition events (Employee Engagement).
- **Leaderboard computation** [MVP]: ranked views scoped to location, team, or tenant-wide, over a
  configurable time window (weekly/monthly/all-time), materialized rather than computed live, with
  tenant-level and per-employee opt-out.
- **Learning pathway gamified progression** [MVP]: a `Pathway` is an ordered sequence of Courses
  and/or Badges building toward a capstone credential, with a visual progress state per employee
  (`PathwayProgress`).
- **Anti-gaming safeguards** [MVP]: rate-limiting and plausibility checks on how fast points/badges can
  accrue, to blunt event-replay abuse, self-dealing recognition loops, and rapid-fire low-quality task
  completion farmed purely for points.
- **Points reversal on upstream invalidation** [MVP]: when a triggering event's underlying record is
  later invalidated (e.g., an audit is found fraudulent, a course completion is revoked), the points
  ledger and any badge awarded solely because of that event are reversed/revoked with a visible,
  non-punitive-toned UX treatment.
- **Reward redemption catalog** [P2]: converting points into tenant-configured redeemable rewards
  (gift cards, PTO, merchandise) is explicitly Phase 2 — MVP points have recognition/status value
  only, not transactional value, to avoid the compliance and finance-integration surface of a rewards
  marketplace in the first release.
- **Manager-curated/ad-hoc badge grants** [P2]: MVP badges are rule-evaluated only (fully event-driven);
  a manager manually awarding a discretionary badge outside a rule is Phase 2 (it would otherwise blur
  with Employee Engagement's `RecognitionEvent`, which already covers ad-hoc manager/peer recognition).

### 1.3 Out of Scope (owned by other modules, referenced here)

This is the section that matters most for this module: Gamification & Recognition **does not own
training content, certifications, or recognition events themselves**. It is a cross-cutting
points/badges/leaderboard layer that **subscribes** to achievement-worthy events published by other
modules and reacts to them; it never calls into those modules synchronously to drive their behavior,
and it never becomes a system of record for anything those modules already own.

- **Course/Certification content, authoring, versioning, and completion semantics** — **Learning
  module (08)**. Gamification subscribes to `training.completed`/certification-issued events; it does
  not define what "completed" means for a course, does not store course content, and does not gate a
  learner's ability to take a course.
- **Recognition events themselves (who gave recognition to whom, message content, feed distribution,
  anonymity/visibility rules)** — **Employee Engagement module (18)**. Gamification only consumes
  `recognition.event.created` as an optional points-earning trigger; it never creates, edits, or
  displays a recognition post.
- **Task/audit/inspection execution, the `WorkItem` state machine, streak definitions' underlying
  completion data** — **Work Orchestration module (02)**. Gamification consumes
  `task.completed`-class events (and a streak-classification signal Work Orchestration or Analytics
  emits) as badge-rule and points inputs; it does not compute streaks from raw WorkItem history itself
  in MVP (see §12 Resolved Decisions — streak-classification ownership — and §11 dependencies).
- **Evidence integrity, audit findings, fraud determination** — **Evidence module (05)** and **Issues
  & CAPA module (06)**. Gamification reacts to an `audit.invalidated`/`issue.opened`-class signal for
  a previously-completed record by reversing points/badges; it does not investigate or determine
  fraud itself.
- **Org hierarchy, scoping, employee identity, privacy/anonymity policy for engagement data** —
  **Identity & Organization module (01)** and **Employee Engagement module (18)**'s aggregation-
  threshold pattern, which Gamification's leaderboard opt-out and small-group suppression logic
  reuses conceptually (see FR-GAM-031) rather than reinventing.
- **Analytics correlation between engagement and operational KPIs** — **Analytics & Control Tower
  module (11)**. Gamification produces the raw facts (badge awards, points events, leaderboard
  participation); Analytics owns the cross-module correlation analysis (e.g., "does leaderboard
  participation correlate with verified execution rate").

## 2. Personas Served

| Persona | Interaction |
|---|---|
| Frontline Associate | Earns badges and points as a byproduct of normal work (no separate "gamification workflow" to learn); views own points history, badges, and pathway progress; sees own position on an opted-in leaderboard; can opt out of leaderboard visibility |
| Store/Site Manager | Views team leaderboard for their location/team scope; sees badge/pathway completion as a proxy for team engagement and skill progression; cannot see leaderboard entries for employees who opted out beyond an aggregate "N opted out" note |
| Operations Admin | Configures badge definitions and award rules, point values per source-event class, leaderboard scopes/time windows, and pathway definitions (course/badge sequences + capstone credential); sets tenant-wide leaderboard enablement and anti-gaming thresholds |
| Regional/District Manager | Views leaderboard/badge/pathway rollups across territory (Analytics-fed), primarily to spot low-engagement locations, not to rank individuals across large populations |

## 3. Badge Definition & Award Rules

- **FR-GAM-001 [MVP].** A `Badge` has a stable logical identity (`id`, `tenant_id`, `name`,
  `description`, `icon_ref`, `category`, `status ∈ {DRAFT, ACTIVE, RETIRED}`) and one or more versioned
  `BadgeRule` rows, mirroring the platform's Template/TemplateVersion pattern (source-spec §17.1) so a
  rule can be tightened/loosened over time without silently rewriting history (see Validation §10 for
  the rule-changed-after-partial-qualification edge case).
- **FR-GAM-002 [MVP].** A `BadgeRule` is expressed declaratively against one or more **source event
  classes** it subscribes to (`task.completed`, `training.completed`, `recognition.event.created`,
  `audit.submitted` with a zero-violations qualifier, a streak-classification event from Work
  Orchestration/Analytics) plus a **condition** (count threshold, rolling window, zero-defect
  qualifier, sequence completion) and an **idempotency scope** (see FR-GAM-060). Example rule shapes:
  - "Award `PERFECT_AUDITOR` when the same user has 10 `audit.submitted` events with
    `violations_count = 0` within a rolling 90-day window."
  - "Award `ONBOARDING_COMPLETE` when a user's `Pathway` (`onboarding-v1`) reaches `COMPLETED`."
  - "Award `STREAK_30` when a user's task-completion-streak classification event reports
    `streak_days >= 30`."
- **FR-GAM-003 [MVP].** Rule evaluation is strictly **event-driven and asynchronous**: the Badge Rule
  Evaluator consumes events from the Event Backbone and evaluates matching rules against accumulated
  per-user rolling counters; it never polls Learning, Work Orchestration, or Issues/CAPA for state, and
  it never blocks the originating module's write path (badge-award latency is asynchronous, see NFR-
  GAM-01).
- **FR-GAM-004 [MVP].** A `BadgeRule` can require the event's source record to remain **valid at
  evaluation time** (e.g., an audit submission that is later marked fraudulent before the rule's
  rolling-window evaluation completes is excluded from the count) — the rule engine treats "count of
  currently-valid qualifying events," not "count of events ever received," as its running total (see
  §10 reversal handling for the case where invalidation happens *after* the badge already awarded).
- **FR-GAM-005 [MVP].** Each `BadgeAward` is a point-in-time, immutable record: `badge_id`, `user_id`,
  `awarded_at`, `badge_rule_version_id` (which version of the rule produced it), and
  `source_event_ids[]` (the specific qualifying events that satisfied the condition, for auditability
  and for reversal — see FR-GAM-062). A `BadgeAward` is never edited in place; a revocation is a
  separate `BadgeRevocation` record referencing it (§10).
- **FR-GAM-006 [MVP].** Badge categories in MVP: `COMPLIANCE` (audit/inspection quality), `LEARNING`
  (course/certification/pathway), `CONSISTENCY` (streaks), `RECOGNITION` (peer/manager recognition
  volume or pattern). Categories drive default iconography and the celebratory-moment copy template
  (§9), not functional behavior.
- **FR-GAM-007 [P2].** Composite/multi-condition rules (e.g., "10 zero-violation audits AND a 20-day
  streak within the same quarter") and negative/decay conditions (e.g., a badge that can be "lost" on
  sustained poor performance) are Phase 2; MVP rules are single-source-class, monotonic-accumulation
  conditions only, to keep the evaluator's semantics easy to reason about and easy to make idempotent.

## 4. Points Ledger

- **FR-GAM-010 [MVP].** Points are recorded as an **append-only transaction log**
  (`PointsLedgerEntry`), never merely as a mutable balance column — the user's current balance is a
  derived aggregate (`SUM(amount) WHERE reversed = false`), not a source of truth. This is a
  deliberate architectural choice (event-sourced ledger) for auditability and safe reversal (§10,
  low-level-design.md rationale).
- **FR-GAM-011 [MVP].** Points-earning sources in MVP: (a) task-completion-streak milestones from Work
  Orchestration/Analytics streak-classification events, (b) course/certification completion from
  Learning (`training.completed`), (c) recognition events from Employee Engagement
  (`recognition.event.created` carrying a `suggested_points` value), (d) badge awards themselves may
  optionally carry a bonus point value (`Badge.bonus_points`, default 0) credited at award time.
- **FR-GAM-012 [MVP].** Each `PointsLedgerEntry` records `user_id`, `amount` (signed integer; positive
  for earn, negative for reversal), `source_type ∈ {STREAK, COURSE_COMPLETION, RECOGNITION,
  BADGE_BONUS, MANUAL_ADJUSTMENT, REVERSAL}`, `source_event_id` (idempotency key, see FR-GAM-061),
  `reason` (human-readable), `related_badge_award_id` (nullable), and `created_at`.
  `MANUAL_ADJUSTMENT` requires `operations_admin` scope and is itself ledgered (never a silent balance
  edit) with a mandatory `reason` field for audit.
- **FR-GAM-013 [MVP].** Point values per source-event class are tenant-configurable
  (`PointsRule`: `source_type`, `condition` (e.g., streak-day threshold tiers), `points_amount`),
  administered by the Operations Admin, defaulting to a conservative reference table (e.g., course
  completion = 50 pts, certification issuance = 100 pts, zero-violation audit = 10 pts, recognition
  received = the `suggested_points` value from Employee Engagement capped per FR-GAM-050).
- **FR-GAM-014 [MVP].** A user's points-history ledger view (§9) shows every entry chronologically with
  source attribution and running balance, including reversed entries shown in-line (struck-through
  style, not deleted) so the history remains a complete, honest record rather than a curated one.
- **FR-GAM-015 [P2].** Point expiry (points that lapse after N months of inactivity) and tiered point
  multipliers (e.g., double points during a campaign window) are Phase 2.

## 5. Leaderboard Computation

- **FR-GAM-020 [MVP].** A `Leaderboard` is a materialized, ranked view of point balances (or, per
  scope config, badge counts) for a defined `scope ∈ {LOCATION, TEAM, TENANT}`, a defined
  `time_window ∈ {WEEKLY, MONTHLY, ALL_TIME}`, and a defined `metric ∈ {POINTS, BADGE_COUNT}`.
  Leaderboards are **precomputed by the Leaderboard Materializer background job**, never computed live
  per-request against the raw ledger (see high-level-design.md §5 scalability rationale).
- **FR-GAM-021 [MVP].** Leaderboard ranking ties are broken deterministically (by `user_id` ascending)
  so re-materialization produces a stable order for equal scores across runs, avoiding a
  visibly-flickering rank for tied users between refreshes.
- **FR-GAM-022 [MVP].** **Tenant-wide leaderboard toggle**: an Operations Admin can disable
  leaderboards entirely for the tenant (`tenant_gamification_settings.leaderboards_enabled = false`).
  When disabled, no leaderboard is computed or displayed anywhere in the tenant, though badges, points,
  and pathway progress continue to function independently — leaderboards are the one gamification
  surface with an explicit, easily-discoverable off switch, per the psychological-safety
  recommendation in §12.
- **FR-GAM-023 [MVP].** **Per-employee leaderboard opt-out**: an associate can opt out of appearing on
  any leaderboard (`user_gamification_preferences.leaderboard_opt_out = true`, default **false** —
  leaderboards are opt-out, not opt-in, to preserve the motivational value for the majority who don't
  mind, while making the exit easy and unconditional). An opted-out user: (a) never appears in any
  leaderboard row anywhere, at any scope, (b) still earns points/badges normally and can see their own
  private points history and pathway progress, (c) is counted only in an aggregate
  "N teammates not shown" footer on their team's leaderboard, never named. This mirrors Employee
  Engagement's aggregation-threshold philosophy (module 18) — a person's individual signal must never
  leak into a view they excluded themselves from, even indirectly.
- **FR-GAM-024 [MVP].** Leaderboard display defaults to showing the top N (default 10, tenant-
  configurable) plus the viewer's own rank/row if outside the top N ("You are #47 of 212"), never the
  full raw ranked list of an entire large tenant-wide scope by default, to keep the UI meaningful and
  to reduce the surface area exposed to comparison anxiety for low performers (a design choice, not
  purely a performance one, though it also assists the scalability goal in high-level-design.md §5).
- **FR-GAM-025 [MVP].** Manager and admin leaderboard views additionally support a small-group
  suppression floor (`min_group_size`, default 5) below which a scoped leaderboard (e.g., a 3-person
  satellite location's "team" leaderboard) is not rendered as a ranked list at all, only as an
  unranked list of participants, to avoid a de-facto public individual comparison in a group too small
  for the ranking to feel psychologically safe — reusing the same small-group-suppression instinct as
  Employee Engagement's aggregation threshold (module 18 FR-ENG-017), applied here to ranking rather
  than survey results.
- **FR-GAM-026 [P2].** Cross-tenant/industry-benchmark leaderboards and configurable custom leaderboard
  metrics beyond points/badge-count (e.g., a specific badge category's count) are Phase 2.
- **FR-GAM-027 [MVP].** **Leaderboard/points data is never an input to performance review or
  discipline.** `Leaderboard`, `PointsLedgerEntry`, `BadgeAward`, and `PathwayProgress` data is
  structurally recreational/motivational and is **never** surfaced as an input, score, or signal
  within any formal employee performance-review, promotion, disciplinary, or termination process or
  system — not through a UI surface, not through an API, and not through an Analytics export tagged
  for HR/performance use. This module exposes no API scope or export path that an HR/performance
  system could consume, and no manager-facing "performance summary" view in this module includes
  gamification data alongside performance metrics. This is enforced structurally (a missing
  integration, not a policy notice a tenant could override), because Gamification's leaderboards
  would otherwise fall within the EU AI Act's Annex III §4 high-risk "employment/worker-management
  AI" classification if used in a performance-review context (engineering-decisions.md §12); keeping
  this module's data out of that context by design, not by admin configuration, is what keeps it
  outside that classification. No Enterprise Admin setting can re-enable this integration — it does
  not exist to be toggled.

## 6. Learning Pathway Gamified Progression

- **FR-GAM-030 [MVP].** A `Pathway` is an ordered sequence of `PathwayStep` entries, each referencing
  either a `Course`/`CourseVersion` (Learning module) or a `Badge` (this module) as its completion
  criterion, culminating in an optional `capstone_credential_name` (e.g., "Certified Shift Lead")
  awarded as a `Badge` in its own right when all steps complete. A Pathway is authored by an Operations
  Admin and versioned like a `Badge` (DRAFT/ACTIVE/RETIRED), never mutated in place once employees have
  active progress against it (see §10 rule-change handling, applied identically to pathway steps).
- **FR-GAM-031 [MVP].** `PathwayProgress` is the per-user projection: `pathway_id`, `user_id`,
  `status ∈ {NOT_STARTED, IN_PROGRESS, COMPLETED}`, `current_step_index`, and a per-step completion
  timestamp array. Steps are completed **in order** by default (`sequential = true`, the MVP-only
  mode); an admin-configurable non-sequential ("complete any N of M steps") mode is **[P2]**.
- **FR-GAM-032 [MVP].** `PathwayProgress` advances purely by consuming the same underlying events as
  badge rules (`training.completed` for a course step, `badge.awarded` for a badge step) — it is not a
  parallel tracking mechanism the learner interacts with directly; a learner's normal act of completing
  an assigned course is what silently advances their pathway.
- **FR-GAM-033 [MVP].** Reaching `COMPLETED` on a `Pathway` with a `capstone_credential_name` triggers
  award of the capstone `Badge` (an internally-generated `BadgeRule` of type `PATHWAY_COMPLETION`
  bound 1:1 to that Pathway) through the same Badge Rule Evaluator pipeline described in §3, not a
  special-cased code path — the capstone is "just a badge" from the ledger/award-audit point of view.
- **FR-GAM-034 [MVP].** Pathway progress visual (§9) shows a linear step tracker with completed,
  current, and locked (not-yet-reached, for sequential pathways) steps, and — for a step bound to a
  Learning `Course` — deep-links directly into that course's assignment surface.

## 7. Anti-Gaming Safeguards

- **FR-GAM-040 [MVP].** **Per-source rate limiting**: each `source_type` has a tenant-configurable
  maximum points-earning velocity (default: no more than 500 points per user per rolling 24-hour
  window from `RECOGNITION` source, no more than 3 `COMPLIANCE`-category badge awards per user per
  rolling 24-hour window). Events that would exceed the cap are still recorded as valid completions in
  their owning module (Gamification never blocks or invalidates the underlying work) but do not
  generate a points-ledger credit or badge award beyond the cap — the excess is logged as
  `capped: true` on the would-be ledger entry for admin visibility, not silently dropped without a
  trace.
- **FR-GAM-041 [MVP].** **Reciprocal-recognition throttling**: if two users' mutual
  `recognition.event.created` volume (A→B and B→A) exceeds a tenant-configurable ratio/frequency
  threshold within a window (default: more than 4 mutual recognitions between the same pair in 7 days),
  subsequent points credit from that pair's recognition is suppressed (recognition itself is
  unaffected — this is Employee Engagement's domain event and its own `rate_flagged` field already
  covers this at the recognition layer per module 18 FR; Gamification independently re-checks at the
  points layer as defense-in-depth since Gamification cannot assume every recognition source integration
  will always set `rate_flagged` correctly).
- **FR-GAM-042 [MVP].** **Minimum quality gate on streak-derived points**: a task-completion-streak
  milestone only credits points if the streak classification event Gamification consumes already
  encodes Work Orchestration's own quality qualifiers (e.g., excludes days where completions were
  bulk-submitted implausibly fast or flagged `low_quality` upstream) — Gamification trusts the upstream
  classification rather than re-deriving completion-speed heuristics itself, consistent with the "does
  not own training content/recognition/work execution" scope boundary in §1.3.
- **FR-GAM-043 [MVP].** **Idempotent, replay-safe evaluation** (detailed in §10/low-level-design.md):
  every badge-rule evaluation and points credit is keyed on the source event's `event_id`, so
  redelivery of the same event (a documented platform-wide possibility, source-spec §18.3/conventions)
  can never double-count toward a rolling threshold or double-credit points.
- **FR-GAM-044 [MVP].** An Operations Admin has a **"flagged activity" review view** listing users who
  hit a rate-limit cap or reciprocal-throttle in a period, without automated punitive action (no
  auto-suspension of points, no auto-revocation) — flags are informational for manager follow-up, since
  a false positive (a genuinely high-performing, high-collaboration employee) is a real risk of a
  purely automated anti-gaming system and the platform's P5 principle (deterministic policy, human
  judgment for consequential actions) applies here too.

## 8. UX States

| State | UX / Behavior |
|---|---|
| Badge-earned celebratory moment | A full-screen (or prominent in-context) celebratory animation/toast the first time the learner opens the app after a badge award (server-driven "unseen award" flag, not a push notification alone, so it works for offline-then-reconnect too); shows badge art, name, and the specific qualifying achievement in plain language ("10 zero-violation audits — nice work"), with a "Share to team feed" action that, if taken, creates an Employee Engagement `RecognitionEvent`-style post (a one-way integration call, not a shared data model — see dependency note §11). |
| Leaderboard opted-out state | The associate's own leaderboard tab shows a clear, non-guilt-inducing explanation ("You've opted out of leaderboards — your points and badges are still tracked privately") with a one-tap re-opt-in action; a manager viewing a team leaderboard containing opted-out members sees an unranked "+3 teammates not shown (opted out)" footer, never a placeholder row with a blank name (which would itself be identifying by elimination in a small team). |
| Pathway-progress visual | Linear step tracker (§ FR-GAM-034); a `NOT_STARTED` pathway shows a "Start your onboarding journey" entry point; `IN_PROGRESS` highlights the current unlocked step with a deep link; `COMPLETED` shows the capstone badge prominently with completion date, distinguished from an in-progress state, not merged into the general badge grid. |
| Points-history ledger view | Reverse-chronological list, each row showing amount (+/-), source label, and reason; reversed entries shown struck-through with a brief neutral explanation ("Reversed — linked audit was later invalidated"), never deleted from the view (§ FR-GAM-014); running balance shown as a running total, not just a final number, so the history is self-verifying. |
| Empty state (no badges/points yet) | Neutral "Complete your first audit or course to start earning" empty state, distinct from a loading state, consistent with the platform's empty-state convention (Work Orchestration FR-WORK-070 pattern). |
| Leaderboard disabled tenant-wide | The leaderboard tab/surface is hidden entirely (not shown-but-empty) when `leaderboards_enabled = false`, so its absence does not read as "you have zero points" — badges/points/pathway surfaces remain fully visible and functional. |
| Badge revoked | A quiet, non-shaming in-app notice on next app open ("A badge tied to a since-corrected record was removed from your profile") rather than a celebratory-style full-screen treatment inverted — reversal UX is deliberately low-key relative to the award UX, since the associate is very often not the party at fault (e.g., a manager falsified the audit, not the associate who nonetheless received the badge for it — see §10 FR-GAM-064). |
| Offline | Badge/points/pathway data is read-only cached content on the frontline app (last-synced snapshot); no gamification action is performed client-side offline — a badge award always originates server-side from a synced event, so there is no offline "pending badge" state to reconcile, only a "data as of last sync" freshness indicator. |

## 9. Validation and Edge Cases

- **FR-GAM-060.** **Event replay must not double-award a badge.** Every inbound event carries a
  platform-standard `event_id` (source-spec §18.4 envelope). The Badge Rule Evaluator persists a
  `(event_id, badge_rule_id)` processed-marker before/atomically-with updating the rolling counter and
  before issuing any `BadgeAward`; a redelivered event with an already-processed marker is a no-op,
  never a second increment or a second award. This is the module's single most important idempotency
  guarantee (see low-level-design.md §4 pseudocode) and is tested explicitly per the platform's
  Idempotency Everywhere principle (source-spec P9).
- **FR-GAM-061.** **Points ledger idempotency.** Identical mechanism applied to points: a
  `PointsLedgerEntry` is keyed with a unique constraint on `(user_id, source_event_id, source_type)`;
  a redelivered `training.completed` event for the same course-completion record cannot create a
  second credit, enforced at the database constraint level (not application-logic-only) as defense in
  depth.
- **FR-GAM-062.** **Employee transferred between locations mid-leaderboard-period.** A user's
  leaderboard membership for a given materialized period is determined by their **location as of the
  leaderboard's period-end (or "as of now" for a rolling all-time board)**, not a historical snapshot
  per point-earning event — i.e., a mid-month transfer means the user's whole month's points count
  toward their *new* location's leaderboard once the transfer is reflected in `UserScope`, not split
  proportionally between old and new locations. This is a deliberate simplicity-over-precision choice
  (documented as a resolved decision in §12) appropriate for a lightweight engagement layer; it explicitly
  does **not** attempt the effective-dated point-in-time attribution rigor that core operational
  reporting (source-spec P7) requires, since a leaderboard rank is a motivational device, not an audit
  record.
- **FR-GAM-063.** **Badge rule changed after some employees already qualified under the old rule.**
  Per FR-GAM-001's versioning: changing a `BadgeRule`'s condition (e.g., raising the audit count from
  10 to 15) creates a new `BadgeRuleVersion`; awards already issued under a prior version are **never
  retracted** by a rule tightening (historical awards stand — consistent with platform historical-truth
  principle P7, applied at module scale). Employees who had *accumulated qualifying events but not yet
  crossed the old threshold* when the rule changes are evaluated against the **new** rule going
  forward — there is no "grandfathering" of partial progress under a superseded rule, and this is
  surfaced transparently: if a rule change would meaningfully raise the bar, the admin's "Impact
  Preview" (mirroring the Forms module's publish-impact preview, source-spec §8.4) shows how many users
  currently have partial progress and how the change affects their remaining distance to the badge,
  before the admin confirms the rule change.
- **FR-GAM-064.** **Points earned then reversed because the triggering event is later invalidated**
  (e.g., an audit is subsequently found fraudulent by Issues/CAPA, or a course completion record is
  revoked by Learning due to a data-entry error). On receiving the corresponding invalidation event
  (e.g., `issue.created` with `source_type=AUDIT_FRAUD` referencing the original `audit.submitted`
  event, or a `training.completion_revoked` event), Gamification: (a) creates a `REVERSAL`
  `PointsLedgerEntry` with a negative amount equal to the original credit, referencing the original
  entry, (b) if a `BadgeAward` depended on that event as one of its `source_event_ids` and, with that
  event excluded, the rule condition would no longer be met, creates a `BadgeRevocation` record and
  removes the badge from the user's active profile (while retaining the historical
  award-then-revoked record for audit), (c) does **not** treat the affected user as at fault by
  default — the UX (§9) is deliberately low-key/neutral, and a tenant policy flag
  (`penalize_user_on_fraud_reversal`, default **false**) governs whether repeated reversals attached to
  the *same* user (as opposed to, e.g., a manager who falsified records affecting several associates)
  should surface as a manager-visible pattern for HR follow-up — never an automated account penalty.
- **FR-GAM-065.** **A user's balance goes negative after a large reversal exceeding their current
  balance** (e.g., points were already partly consumed by a Phase-2 redemption feature, or points
  decayed/expired before the reversal). MVP has no redemption (§1.2), so this scenario is narrow in
  practice, but the ledger explicitly **permits a negative running balance** rather than floor-clamping
  it at zero — floor-clamping would silently understate the true reversal amount and break the ledger's
  auditability invariant (sum of entries always equals displayed balance). The UI displays a negative
  balance plainly rather than as zero.
- **FR-GAM-066.** **User deactivated with open pathway progress or pending badge evaluation.** On
  `user.deactivated` (Identity/Org), in-flight `PathwayProgress` and rolling badge-rule counters are
  frozen (not deleted) — a deactivated user's historical badges/points/pathway state remains visible
  read-only to an admin (audit/reference), consistent with the platform-wide historical-attribution
  rule, but the user no longer appears on any active leaderboard (which is a live/current-population
  view by definition, not a historical one).
- **FR-GAM-067.** **Duplicate/near-simultaneous qualifying events for a rolling-window rule** (e.g.,
  two `audit.submitted` events for the same underlying audit due to an upstream retry that was not
  itself idempotent). Gamification's own idempotency (FR-GAM-060) is keyed on `event_id`, which
  protects against re-delivery of the *same* event; it explicitly does **not** attempt to deduplicate
  two *distinct* `event_id`s that an upstream module's own bug produced for what a human would consider
  "the same audit" — that correctness burden belongs to the emitting module (Work Orchestration/Evidence),
  consistent with the module boundary in §1.3. This is called out here as an explicit assumption, not
  silently glossed over.

## 10. Non-Functional Requirements (module-specific)

- **NFR-GAM-01 — Badge-award latency.** From the qualifying event's `occurred_at` to the `BadgeAward`
  row being created and a `badge.awarded` event emitted: p95 ≤ 2 minutes online. This is intentionally
  looser than Work Orchestration's sub-5-second WorkItem-update latency (NFR-WORK-01-class targets)
  because badge celebration is a delight feature, not an operational-state signal nothing else depends
  on synchronously — batched/near-real-time evaluation is an acceptable and preferred trade-off for
  throughput (see high-level-design.md §5).
- **NFR-GAM-02 — Leaderboard freshness.** Materialized leaderboards refresh at a tenant-configurable
  cadence, default every 15 minutes for `WEEKLY`/`MONTHLY` scopes and hourly for `ALL_TIME`; the
  leaderboard UI always shows an explicit "updated N minutes ago" freshness indicator rather than
  implying live/real-time ranking, so users never mistake a stale rank for a live one after just
  earning points.
- **NFR-GAM-03 — Points-ledger read latency.** A user's own points-history view and current balance
  read p95 ≤ 300 ms (standard platform API performance target, source-spec §23), served from an
  indexed running-balance read, not a full ledger re-scan per request (see low-level-design.md
  caching).
- **NFR-GAM-04 — Backfill reconciliation.** A missed-event backfill/reconciliation job (high-level-
  design.md §7, low-level-design.md §8) can process a full tenant's badge-rule re-evaluation for up to
  100k users within a bounded overnight batch window (target: complete within 4 hours) without
  requiring re-processing of already-idempotently-recorded events (it is additive, not a full replay).

## 11. Dependencies on Other Modules

| Module | Direction | Nature |
|---|---|---|
| Learning (08) | Consumes | `training.completed` and certification-issuance events as points/badge/pathway-step triggers; Gamification never reads Learning's tables directly and never influences course content, assignment, or certification logic |
| Work Orchestration (02) | Consumes | `task.completed`/`audit.submitted` events and a streak-classification signal (task-completion-streak milestones) as points/badge triggers; consumes an `audit.invalidated`-class signal (routed via Issues/CAPA, see below) for reversal |
| Issues & CAPA (06) | Consumes | An `issue.created`/fraud-determination-class event referencing a previously-completed audit/submission is the trigger for points/badge reversal (FR-GAM-064); Gamification does not investigate fraud itself |
| Employee Engagement (18) | Consumes | `recognition.event.created` (carrying `suggested_points`, `rate_flagged`) as an optional points-earning trigger; Gamification never creates or displays a recognition post itself, and independently re-applies its own anti-gaming throttle as defense-in-depth (FR-GAM-041) |
| Identity & Organization (01) | Consumes | `OrganizationNode`/`UserScope` for leaderboard scope resolution (location/team) and `user.deactivated`/`user.role_changed`/transfer events for freezing progress and leaderboard-membership recalculation |
| Analytics & Control Tower (11) | Produces | Publishes badge-award, points-ledger, leaderboard-participation, and pathway-completion facts for Analytics' engagement-vs-operational-outcome correlation work; Gamification does not itself perform that correlation analysis |
| Communications (07) | Produces (calls) | Optional "badge earned — share to team feed" action is a one-way call into Employee Engagement/Communications-style feed posting, not a shared data model |
| Notification Hub | Produces (calls) | Badge-earned and pathway-milestone push/in-app notifications routed through the platform Notification Hub interface, not delivered directly |

## 12. Resolved Decisions

- **Can leaderboards be disabled tenant-wide or per-employee, and at what default (FR-GAM-022,
  FR-GAM-023)?** Firm decision: **yes, at both levels.** Tenant-wide default **enabled** (leaderboards
  are a proven engagement driver and most tenants will want them); per-employee default **not opted
  out** (opt-out available, off by default) — both toggles easy to find and reversible. Rationale:
  frontline-worker psychological safety (competitive visibility can demotivate lower performers, and
  can be uncomfortable in some cultures/tenant industries such as healthcare-adjacent site ops)
  outweighs the marginal engagement lift from a mandatory leaderboard for the minority who would opt
  out.
- **Leaderboard/points data and performance review (new, FR-GAM-027).** Firm decision:
  leaderboard/points/badge/pathway data is **never** surfaced as an input to a formal
  performance-review or disciplinary process, structurally — not merely by tenant policy. See
  FR-GAM-027 for the enforced mechanism. Rationale: this keeps Gamification outside the EU AI Act's
  Annex III §4 high-risk "employment/worker-management AI" classification by design
  (engineering-decisions.md §12), rather than relying on every tenant to configure it correctly — the
  same design principle §12 already applies to Employee Engagement's `EngagementScore` (module 18) via
  its N=5 aggregation threshold.
- **Streak-classification ownership (FR-GAM-042).** Firm decision: streak classification (what counts
  as a "streak" and its quality qualifiers) is computed and emitted by Work Orchestration or
  Analytics, never by Gamification itself — Gamification is strictly a consumer of an already-
  classified signal, consistent with its "does not own work execution" scope boundary (§1.3). This
  signal is a **build-sequencing dependency**: Work Orchestration/Analytics must ship the
  streak-classification event before Gamification's streak-based badges/points can be enabled for a
  tenant; streak-based rules are sequenced after that signal exists (high-level-design.md §7 documents
  the corresponding failure-mode/rollout handling), and every other MVP surface (badges, points from
  non-streak sources, leaderboards, pathways) ships independently of this dependency.
- **Leaderboard membership timing (FR-GAM-062).** Firm decision: MVP leaderboard membership uses
  "current location as of materialization time," not point-in-time effective-dated attribution.
  Rationale: a leaderboard rank is a motivational, non-audit artifact, so the platform's usual
  effective-dated attribution rigor (P7) is deliberately not applied here; this is revisited only if
  real tenant feedback shows the simplification creates a fairness-perception problem at scale.
- **Reward redemption and monetary exchange rate.** Firm decision: reward redemption (converting
  points to real-world value) is out of scope for the entire MVP release (§1.2); the points ledger's
  `amount` field has no defined monetary exchange rate, and no finance/payroll integration exists for
  this module. The platform communicates to tenants at rollout that points are recognition/status
  only, never an implied compensation mechanism, to avoid a wage-and-hour or tax-treatment question the
  platform is not positioned to answer in this version.
- **Manager-curated/ad-hoc discretionary badge grants.** Firm decision: **out of scope for MVP**
  (already tagged [P2] in §1.2); when built, it ships in coordination with Employee Engagement so a
  discretionary badge grant does not diverge into a second "give someone credit for something"
  mechanism alongside `RecognitionEvent`. No MVP surface allows a manager to award a badge outside a
  rule.
- **Public-facing (external) recognition wall.** Firm decision: **out of scope for this spec
  entirely**, not merely deferred silently. Badge/points data is never exposed externally (e.g. for
  franchise-brand "great places to work" marketing) in this version. Any future external-facing use
  requires a dedicated privacy/consent review and a Communications-module collaboration before it is
  designed, since it would cross from an internal engagement tool into external-facing brand content
  and would need its own consent model distinct from the internal opt-out in FR-GAM-023.
