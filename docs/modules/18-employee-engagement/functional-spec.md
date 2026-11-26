# Employee Engagement & Voice of Frontline — Functional Specification

Module: `18-employee-engagement`
Tier: 2 (competitive gap closure — new-modules-addendum.md)
Epic: EPIC-08 extension (Comms & Knowledge infra reuse) + new EPIC-16 (Engagement) — see §7 traceability
Source: `source-spec.md` §7.1 (`Survey` WorkItem type already declared), §11.1 (Communications
targeting model this module reuses), §17 (Domain Model conventions); `new-modules-addendum.md`
Module Map row 18.

## 1. Purpose

Frontline organizations run on the people who work the floor, and this module gives that population
a governed, two-way voice channel — distinct from Communications, which is one-way broadcast +
acknowledgement. This module owns three related capabilities: (1) **pulse surveys and eNPS
measurement** — short, recurring, targeted surveys with anonymous or attributed response modes; (2)
**recognition/rewards events** — peer-to-peer and manager-to-associate recognition, optionally
linked to points in the Gamification module; (3) **engagement scoring** — a computed, rolled-up
metric per location/team/tenant that Analytics and Control Tower consume to answer "how do our
frontline people feel, and where is morale a leading indicator of operational risk."

The trust-critical property of this module is **anonymity**: when a survey is configured as
anonymous, no path through the product — including broad-scope admin roles — may reconstruct which
individual gave which response once the configured aggregation threshold is not met. This is a
harder guarantee than ordinary RBAC scoping and is treated as a first-class design constraint
throughout this document, not an afterthought.

## 2. Scope

### 2.1 In scope

- **Pulse survey design**: campaign-style surveys built from a constrained subset of the Forms
  component library (source-spec §8.1): `rating` (1–5 or 1–10 scale), `single_select`,
  `multi_select`, `open_text` (short free text). Matrix/repeating-group/calculated-field form
  components are explicitly excluded from survey question types (§2.2).
- **eNPS-style scoring**: a dedicated `eNPS` question subtype (0–10 "how likely are you to
  recommend working here" scale) with the standard promoter/passive/detractor computation.
- **Survey campaign scheduling & targeting**: recurrence (one-time, monthly pulse, quarterly, custom
  cadence), and audience targeting — this module does **not** re-implement audience resolution; it
  calls Communications & Knowledge's audience-resolution API (see §2.2, HLD §5).
- **Anonymous vs. attributed response modes**: per-survey configuration of whether individual
  responses are linked to a respondent identity, plus the aggregation-threshold suppression rule
  that governs when any aggregated view (by location, team, role, shift, tenure band) may be
  displayed.
- **Recognition events**: peer-to-peer and manager-to-associate recognition posts (a short message +
  optional category, e.g. "Teamwork," "Above & Beyond"), an in-app recognition feed, and an optional
  points award that this module emits as an event for Gamification (module 22) to act on — this
  module does not maintain a points ledger itself.
- **Engagement score computation & rollup**: a composite score per location/team/role derived from
  survey response rates, eNPS, sentiment/rating trends, and recognition activity, rolled up the
  `OrganizationNode` tree with the same aggregation-threshold suppression as raw survey results.
  Trend reporting over time (week/month/quarter) at each rollup level.
- **Duplicate-response prevention, mid-campaign transfer handling, and recognition abuse/spam
  guardrails** (see §6).

### 2.2 Out of scope (owned elsewhere, referenced here)

- **Multi-channel delivery and audience-targeting infrastructure** — owned entirely by
  Communications & Knowledge (Module 07). This module calls Communications' audience-resolution API
  (the same `resolve_audience(tenant_id, audience_rules, as_of)` contract documented in Module 07's
  LLD §4) to compute a survey campaign's recipient set, and calls its delivery/fan-out API
  (push/email/in-app) to notify recipients that a survey is open. This module does not implement a
  second notification pipeline, a second audience-rule engine, or a second translation workflow.
- **The generic Forms component library and template versioning engine** — owned by Forms &
  Workflow Builder (Module 03). This module consumes a constrained subset of question/component
  types from that library (rating, single/multi-select, open text, plus the eNPS subtype it adds) —
  it does not fork or reimplement the form rendering engine.
- **The `Survey` WorkItem type's generic state machine mechanics** (`AVAILABLE → IN_PROGRESS →
  SUBMITTED → ...`) — owned by Work Orchestration (Module 02); this module is that WorkItem type's
  **functional owner**, meaning it defines what a `Survey` WorkItem's `payload` contains, what
  "submission" means for a survey, and how survey-specific business rules (anonymity, aggregation)
  layer on top of the generic WorkItem lifecycle. It does not redefine the WorkItem state machine.
- **Points ledger, badges, leaderboards** — owned by Gamification (Module 22). This module only
  produces a `recognition.event.created` event carrying an optional `suggested_points` value;
  Gamification decides whether/how to credit points.
- **Engagement KPI dashboards and cross-module analytics correlation** (e.g., "does low engagement
  correlate with higher repeat-failure rate") — owned by Analytics & Control Tower (Module 11); this
  module produces the underlying facts (`engagement.score.computed`, survey/eNPS results at
  authorized aggregation levels) that Analytics consumes and visualizes.
- **Identity, `UserScope` resolution, RBAC/ABAC primitives** — owned by Identity & Organization
  (Module 01). This module consumes resolved scope for targeting and for the anonymity-safe
  aggregation query, and it does not define new authorization primitives — it defines an additional
  **data-access rule** (the aggregation threshold) that sits on top of ordinary scope-based RBAC.

## 3. Personas served

| Persona | Relationship to this module |
|---|---|
| Frontline Associate | Responds to pulse surveys (anonymously or attributed, per campaign config); gives and receives peer recognition; sees their own recognition feed and any engagement content shared back with their team. |
| Store/Site Manager | Gives manager-to-associate recognition; views their team's engagement score and survey results **only where the aggregation threshold is met**; cannot see individual attributed responses unless the survey was explicitly configured as attributed and the respondent consented to that mode. |
| Regional/District Manager | Views engagement rollups across their subtree, subject to the same aggregation-threshold suppression at every level of the rollup, not just the leaf level. |
| Enterprise/Operations Admin | Designs survey campaigns (questions, targeting, cadence, anonymity mode, aggregation threshold policy), publishes/closes campaigns, configures recognition categories and abuse guardrails. |
| HR / People Analytics Consumer | Consumes engagement/eNPS trend reports and recognition activity summaries, typically at higher organizational levels; subject to the same anonymity/aggregation rules as any other role — HR access does not bypass suppression. |
| Executive | Read-only visibility into enterprise-level engagement score and eNPS trend via Analytics & Control Tower (not this module directly). |

## 4. Functional Requirements

### 4.1 Pulse survey design

- **FR-ENG-001**: The system shall let an authorized admin create a `PulseSurvey` with `title`,
  `description`, `survey_type` (`PULSE`, `ENPS`, `CUSTOM`), `anonymity_mode` (`ANONYMOUS`,
  `ATTRIBUTED`), `aggregation_threshold` (tenant-default or campaign-level override, integer ≥
  tenant-configured hard floor — see §8), and one or more `SurveyQuestion` rows.
- **FR-ENG-002**: Each `SurveyQuestion` shall have `question_type` constrained to: `RATING` (scale
  configurable 1–5 or 1–10), `SINGLE_SELECT`, `MULTI_SELECT`, `OPEN_TEXT` (max 2,000 chars), or
  `ENPS` (fixed 0–10 scale, at most one `ENPS` question per survey, reserved wording: "How likely
  are you to recommend working at [tenant/location] to a friend or colleague?" — tenant may localize
  wording but not the 0–10 scale or promoter/passive/detractor banding).
  `MATRIX`, `REPEATING_GROUP`, and `CALCULATED_FIELD` component types from the Forms library
  (source-spec §8.1) are not offered in the survey question type picker.
- **FR-ENG-003**: The system shall support marking any non-`ENPS` question `required` or optional; a
  survey with zero required questions is still valid (a fully-optional pulse is a supported pattern
  for sensitive topics).
- **FR-ENG-004**: The system shall support a survey question bank (reusable question templates
  tagged by category, e.g. "Manager Support," "Workload," "Growth") so admins compose new campaigns
  from previously validated questions rather than re-authoring each time.
- **FR-ENG-005**: `anonymity_mode` shall be set at survey creation and **shall not be changed after
  the survey transitions to `SCHEDULED` or later** (immutable once respondents may have been told
  what mode applies — changing it retroactively would violate the anonymity promise made to
  respondents). Changing anonymity mode requires cloning the survey into a new draft.

### 4.2 Survey campaign scheduling & targeting

- **FR-ENG-006**: The system shall support recurrence patterns for a `PulseSurvey` campaign:
  one-time, fixed interval (weekly/monthly/quarterly), and custom cron-style cadence, reusing Work
  Orchestration's scheduling primitives (source-spec §7.2) to generate a new `Survey` WorkItem
  instance (and underlying `PulseSurveyRun`, see HLD §4) per occurrence.
  campaign occurrence.
- **FR-ENG-007**: Audience targeting for a survey campaign shall be defined using the identical
  audience-rule shape as Communications & Knowledge (`scope_node_ids`, `role_ids`,
  `attribute_predicate`, unioned across rules) and resolved by calling Communications'
  audience-resolution API — this module does not implement its own resolver (see §2.2).
- **FR-ENG-008**: The system shall provide an audience preview (estimated recipient count, no
  individual sampling for `ANONYMOUS`-mode surveys — showing "sample respondents" before an
  anonymous survey opens could itself leak the small-group problem it's designed to avoid; sampling
  is permitted for `ATTRIBUTED`-mode surveys only, mirroring FR-COMMS-003).
- **FR-ENG-009**: The system shall deliver survey availability notifications via Communications'
  delivery API (push/email/in-app), and shall create the corresponding `Survey` WorkItem
  (`AVAILABLE` state) for each resolved recipient, reusing the canonical WorkItem state machine.
- **FR-ENG-010**: A survey campaign shall have a `close_at` (explicit close time or "N days after
  open," configurable); after close, no further responses are accepted and aggregation/publication
  proceeds per §4.3–4.4.

### 4.3 Anonymous vs. attributed response modes & aggregation-threshold rule

- **FR-ENG-011**: For an `ANONYMOUS` survey, the system shall **not persist any identifier that maps
  a stored `SurveyResponse` row back to the responding `user_id`**, either directly or via a
  reversible lookup, beyond what is strictly required to prevent duplicate submission (see
  FR-ENG-014 and LLD §1.3/§8 for the pseudonymization mechanism used to satisfy both constraints
  simultaneously).
- **FR-ENG-012**: For an `ATTRIBUTED` survey, `SurveyResponse` rows carry `user_id` directly, and
  individual attributed responses are visible only to roles explicitly authorized for
  individual-level survey data (e.g., the respondent's direct manager, or HR, per tenant policy) —
  attribution does not imply "visible to any admin with broad org scope"; it is still subject to
  ordinary RBAC/scope rules (not the aggregation-threshold rule, which applies only to derived
  aggregates and remains relevant even for attributed surveys' *aggregate* views).
- **FR-ENG-013**: The system shall enforce an **aggregation-threshold suppression rule** on every
  aggregated view of survey/eNPS/engagement data (by location, team, role, shift, tenure band, or
  any other grouping dimension): a group with fewer than `aggregation_threshold` respondents (for
  that specific question/dimension combination) shall never render computed statistics (mean,
  distribution, eNPS score, open-text responses) for that group. The suppressed state is rendered
  explicitly (§5 UX) — never a silently blank or zero-value chart that could be misread as "no
  signal."
  - Default `aggregation_threshold` = **5** respondents, tenant-configurable upward only, with a
    **hard floor of 5** that no tenant configuration may set below, enforced at the API/service
    layer, not just in UI (see §9 for rationale). This N=5 floor is also what keeps `EngagementScore`
    out of the EU AI Act high-risk classification described in `_context/engineering-decisions.md`
    §12 — the score is structurally never surfaced at an individually-identifiable level.
- **FR-ENG-014**: The system shall prevent duplicate submission of the same survey occurrence by the
  same user — for `ATTRIBUTED` surveys, via a normal `(survey_run_id, user_id)` uniqueness
  constraint; for `ANONYMOUS` surveys, via a **separately-keyed pseudonymization token**
  (`respondent_token`) that is derived from `(survey_run_id, user_id, tenant secret)` using a
  one-way, tenant-scoped keyed hash, stored only in a narrowly-access-controlled dedupe table
  distinct from `survey_responses`, and never joinable back to `user_id` by application code paths
  that read survey results (LLD §1.3, §8 pseudocode gives the exact mechanism).
- **FR-ENG-015**: Open-text responses on an `ANONYMOUS` survey are subject to the same
  aggregation-threshold rule as numeric aggregates: open-text responses for a group below threshold
  are never surfaced individually or in aggregate (not even as a word cloud or count), because
  free-text is inherently more re-identifying than a numeric rating.
- **FR-ENG-016**: Aggregation-threshold enforcement shall be applied **server-side, at query time**,
  never delegated to the client — the API never returns raw per-respondent or below-threshold
  grouped rows to any client, admin tooling, or export path, including CSV/BI exports (Analytics
  integration must consume only already-threshold-safe aggregates or fact rows that exclude
  identity, per HLD §6).
- **FR-ENG-017**: A user with org-wide or multi-level scope (e.g., a Regional Manager whose scope
  spans 3 stores) viewing a rolled-up aggregate shall be subject to the threshold check **at the
  finest grain requested**, not only at their broadest scope — requesting "Store #12 results" when
  Store #12 has 3 anonymous respondents is suppressed even though the requester's total scope (all
  3 stores, 40 respondents) exceeds the threshold; drilling into an under-threshold subgroup is
  exactly the attack this rule exists to prevent.

### 4.4 eNPS-style scoring computation

- **FR-ENG-018**: For any `ENPS` question, the system shall classify each 0–10 response as
  `PROMOTER` (9–10), `PASSIVE` (7–8), or `DETRACTOR` (0–6), and compute
  `eNPS = (%PROMOTER − %DETRACTOR)`, rounded to the nearest integer, range −100 to +100.
- **FR-ENG-019**: eNPS shall be computed and displayed only for groups meeting the aggregation
  threshold (§4.3); a group below threshold shows the suppressed state, not a partial/estimated
  eNPS.
- **FR-ENG-020**: eNPS trend reporting shall show the score over successive survey occurrences at a
  given rollup level, with response-rate (participation) shown alongside eNPS so a score computed
  from a low-turnout occurrence is visually distinguishable from a high-confidence one (participation
  rate itself is also subject to the aggregation-threshold rule for small groups — see §4.6).

### 4.5 Recognition events

- **FR-ENG-021**: The system shall let any active user post a `RecognitionEvent`: `from_user_id`,
  `to_user_id`, `message` (max 500 chars), `category` (tenant-configurable list, e.g. "Teamwork,"
  "Customer Focus," "Above & Beyond," "Safety Champion"), and optional `visibility`
  (`TEAM_FEED`, `LOCATION_FEED`, `PRIVATE_TO_RECIPIENT`).
- **FR-ENG-022**: A `RecognitionEvent` may optionally carry `suggested_points` (bounded by a
  tenant-configured per-event max and a per-giver rolling-period cap, e.g. max 50 points/event, max
  200 points/giver/week) — this module validates the caps and emits `recognition.event.created`; it
  does not itself credit or track a points balance (owned by Gamification, module 22).
- **FR-ENG-023**: Manager-to-associate recognition is distinguished from peer-to-peer recognition by
  `giver_role_category` (`MANAGER`, `PEER`) computed at creation time from the giver's role relative
  to the recipient's `OrganizationNode`, used for reporting (e.g., "% of associates who received
  manager recognition this quarter") without restricting who can give recognition (both directions
  are peer-to-peer capable; the distinction is descriptive, not access-controlling).
- **FR-ENG-024**: A `RecognitionEvent` targeting a user outside the giver's own location/team scope
  is permitted (cross-location recognition, e.g. a traveling regional trainer recognizing an
  associate at a store they visited) but is flagged `cross_scope = true` for reporting/abuse-review
  purposes (§6).
- **FR-ENG-025**: The recognition feed (`TEAM_FEED`/`LOCATION_FEED` scoped) shall be visible to all
  active users within the relevant scope, showing giver, recipient, message, category, and timestamp
  — recognition is never anonymous (anonymity rules in §4.3 apply exclusively to survey responses,
  not recognition, which is inherently an attributed social action).

### 4.6 Engagement score computation & rollup

- **FR-ENG-026**: The system shall compute an `EngagementScore` per `OrganizationNode` per
  reporting period (default weekly rollup, monthly/quarterly views computed from weekly facts) as a
  weighted composite of: (a) survey participation rate, (b) mean pulse rating(s) normalized to
  0–100, (c) eNPS (rescaled from [−100,100] to [0,100] for composite blending), (d) recognition
  activity rate (recognitions received per active user in period). Weights are tenant-configurable
  with a documented platform default (survey ratings 40%, eNPS 30%, participation 15%, recognition
  activity 15%).
- **FR-ENG-027**: `EngagementScore` rows shall roll up the `OrganizationNode` tree (leaf location →
  district → region → enterprise) as a participant-weighted average of child-node scores, computed
  only from child nodes that themselves meet the aggregation threshold at their level — a
  below-threshold leaf node contributes to its parent's rollup denominator/numerator (so signal
  isn't lost at the enterprise level) but is never separately displayed.
- **FR-ENG-028**: Engagement trend reporting shall show the score over time at any rollup level the
  viewer is authorized for, with period-over-period delta, subject to the same threshold suppression
  as §4.3.
- **FR-ENG-029**: `EngagementScore` computation shall be idempotent and re-runnable for a given
  `(node_id, period)` — a re-run (e.g., after a late-arriving response within a grace window) must
  produce the same result given the same input set, and shall version the computation
  (`computed_at`, `input_response_count`, `scoring_version`) rather than silently overwrite history
  used in prior reports (mirrors P7 Historical Truth for reporting facts already surfaced to users).

## 5. UX States

- **Survey-pending**: frontline "Today" surface shows a pending `Survey` WorkItem exactly like other
  work items (per source-spec §7.3 "Today" pattern), with an estimated completion time (typically
  under 2 minutes) and, for `ANONYMOUS` surveys, an explicit **"Your response is anonymous"** badge
  shown before the user starts, plus a one-line explanation of what "anonymous" guarantees here
  (no name/ID stored, results only shown in groups of `N`+ people) so respondents can trust the
  mechanism rather than take it on faith.
- **Anonymous-response confirmation**: on submission of an `ANONYMOUS` survey, the confirmation
  screen explicitly reiterates "This response was submitted anonymously — it cannot be traced back
  to you" rather than a generic "Thank you" — reinforcing the guarantee at the moment of action.
- **Small-sample-suppressed-results state**: any dashboard/report card for a group below
  `aggregation_threshold` renders a clearly labeled state: *"Not enough responses to show results
  (fewer than N people) — protecting individual anonymity."* This never appears as an empty chart,
  a zero, or a loading spinner that never resolves — it is a distinct, explicit UI state with an
  icon (e.g., a shield) distinguishing it from a true zero-response empty state (§6 also covers "zero
  responses" vs. "below threshold" as materially different states that must not be visually
  conflated).
- **Recognition feed**: reverse-chronological feed scoped to the viewer's location/team, with
  category badges/icons, a "give recognition" action always visible, and a lightweight reaction
  (e.g., "cheer") affordance separate from giving formal recognition.
- **Campaign authoring — impact preview**: before publishing a survey campaign, the admin sees
  estimated recipient count (via Communications' audience preview) and, for `ANONYMOUS` surveys, a
  warning if the resolved audience for any single leaf `OrganizationNode` is smaller than
  `aggregation_threshold` (that node's individual results will always be suppressed — the admin is
  told this at design time, not discovered after the campaign closes).
- **Manager engagement dashboard**: shows the manager's directly-scoped team's engagement score,
  trend, and eNPS **only where threshold is met**; a manager of a small team below threshold sees
  the suppressed state plus their score rolled into the next level up ("Your team's individual score
  isn't shown to protect anonymity, but is included in [District]'s score").
- **Loading/offline/error states**: standard skeleton loaders for dashboards; a survey started
  offline is completed via the standard offline WorkItem/Submission queue (Module 04) and syncs
  idempotently; a survey campaign that closed while a response was queued offline is rejected on
  sync with a clear `SURVEY_CLOSED` reason (response is preserved locally for the user to see it
  wasn't counted, not silently dropped).

## 6. Validation & Edge Cases

- **Exactly-at-threshold edge case**: a group with respondent count **exactly equal to**
  `aggregation_threshold` (e.g., exactly 5 when the threshold is 5) **shall be shown**, not
  suppressed — the rule is "fewer than N," i.e., `count < aggregation_threshold` suppresses,
  `count >= aggregation_threshold` shows. This boundary is unit-tested explicitly (`count = N-1`
  suppressed, `count = N` shown) because off-by-one errors here are a direct anonymity/UX defect.
- **Employee transferred mid-campaign**: if a respondent's `OrganizationNode` assignment changes
  after they submit an anonymous response but before campaign close, the already-stored response is
  **not** retroactively re-attributed to the new node — anonymous responses snapshot the
  respondent's scope context (location/team/role, stored as non-identifying dimension attributes
  alongside the response, not the identity) at submission time. A user who has not yet responded and
  transfers mid-campaign is re-targeted per the new audience resolution on the next re-resolution
  cycle (mirroring Communications' FR-COMMS-015 pattern) — they either gain or lose eligibility for
  that occurrence based on the new targeting rules, and if they gain eligibility late in a short
  campaign window, they still get a fair chance to respond before `close_at`.
- **Duplicate response prevention — offline replay**: a response submitted offline and queued twice
  (e.g., app retry after an ambiguous network failure) is deduplicated using the same
  `client_uuid`-keyed idempotency pattern as ordinary submissions (source-spec §19.4) for
  `ATTRIBUTED` surveys; for `ANONYMOUS` surveys, the `respondent_token` dedupe table additionally
  enforces "one counted response per user per survey occurrence" independent of `client_uuid`, so
  even a client bug that generates two different `client_uuid`s for the same logical submission
  cannot produce two counted anonymous responses.
- **Recognition abuse/spam guardrails**: rate limits per §FR-ENG-022 (points caps) apply
  independently of a separate **volume guardrail** on the recognition feed itself: a per-giver cap
  on recognition *events* per rolling 24h window (tenant-configurable, default 10) to prevent feed
  flooding; recognitions between the same giver→recipient pair beyond a configurable frequency
  (default: more than 3 in 24h) are still recorded (never silently dropped — that would be
  surprising and undermine trust) but flagged `rate_flagged = true` and excluded from
  engagement-score recognition-activity input (so gaming the score by spamming recognition doesn't
  work), with the flag visible to admins reviewing recognition activity reports.
- **Recognition targeting an inactive/deactivated user**: permitted for historical/retrospective
  recognition (e.g., recognizing someone on their last day) but a deactivated `to_user_id` cannot
  receive a points award notification through normal channels; the `recognition.event.created` event
  still fires so Gamification can decide its own deactivated-user points policy.
- **Small-group anonymity vs. legitimate manager visibility tension**: a Store Manager whose entire
  store is below threshold will never see their own store's anonymous survey results in this
  module — this is an intentional, non-configurable-below-the-hard-floor product trade-off (protecting
  associates outweighs a manager's visibility need); the mitigation is rollup visibility at the next
  level and, over multiple survey occurrences, a **trend-only** view (e.g., "your store's
  participation rate over time," which is not itself sensitive) may be shown even when point-in-time
  ratings are suppressed, since participation rate alone across time does not reveal individual
  sentiment — this trend-only carve-out is documented as tenant-configurable, defaulting to on.
- **Attributed survey response visibility scope creep**: an `ATTRIBUTED` survey's individual
  responses are restricted (FR-ENG-012) to a specifically configured viewer set (e.g., "respondent's
  direct manager only") — a broader-scoped admin viewing an aggregate rollup that happens to include
  an attributed survey's data still only sees the threshold-gated aggregate, never a drill-down to
  individual attributed rows, unless they are also in the specifically authorized individual-viewer
  set for that survey.
- **Zero responses vs. below-threshold**: a group with **zero** responses is a materially different
  state from a group with 1..N-1 responses — zero is rendered as "No responses yet" (participation
  problem, potentially actionable — e.g., low awareness or access issue) while 1..N-1 is rendered as
  the anonymity-suppressed state (§5) — conflating these would either leak "at least one person
  responded" information inappropriately in the zero case's framing, or misrepresent a true
  non-participation problem as a privacy protection.

## 7. Non-Functional Requirements

| Requirement | Target | Notes |
|---|---|---|
| Survey response submission latency | p95 < 500 ms server-side write (online); offline-first, queued per Module 04 | Response is a lightweight write (no media by default) |
| Anonymization dedupe-check latency | p95 < 100 ms per response (keyed-hash lookup) | Must not add perceptible latency to a 2-minute pulse survey |
| Aggregation computation freshness | Dashboard aggregates reflect responses received up to 15 minutes prior (near-real-time, not real-time) for open campaigns; final aggregation within 5 minutes of `close_at` for campaign-close reports | Background rollup job, not synchronous per-response recompute (LLD §8) |
| Engagement-score rollup freshness | Weekly rollup computed within 4 hours of period close; ad-hoc re-run available on demand for authorized admins | |
| Aggregation-threshold enforcement correctness | 100% — zero tolerance for a below-threshold group ever rendering computed stats, verified by dedicated authorization/anonymity regression test suite (see §9 QA note) | Treated with the same rigor as tenant-isolation tests (source-spec §24.1) |
| Audience resolution / delivery | Delegated to Communications & Knowledge NFRs (Module 07 §5) — this module adds no separate SLA | |
| Recognition feed latency | p95 < 2s from post to feed visibility for in-scope viewers | |

## 8. Dependencies

| Dependency | Direction | Nature |
|---|---|---|
| Communications & Knowledge (Module 07) | Consumes | Audience-resolution API and multi-channel delivery API for survey campaign targeting/notification — this module never re-implements targeting or delivery (§2.2). |
| Work Orchestration (Module 02) | Consumes / Extends | This module is the functional owner of the `Survey` WorkItem type (source-spec §7.1): defines its `payload` schema, submission semantics, and scheduling recurrence hookup, while Work Orchestration owns the generic WorkItem state machine and scheduling engine primitives. |
| Forms & Workflow Builder (Module 03) | Consumes | Reuses a constrained subset of form component types (rating, single/multi-select, open text) for `SurveyQuestion` rendering; does not use Forms' full template-versioning/branching engine (surveys are simpler and versioned via `PulseSurvey`/`SurveyQuestion` rows directly, not `TemplateVersion`). |
| Analytics & Control Tower (Module 11) | Provides to | Produces `engagement.score.computed`, `survey.campaign.closed`, `enps.computed` facts (already threshold-safe, non-identifying) for enterprise dashboards and correlation analysis (e.g., engagement vs. repeat-failure rate). |
| Gamification & Recognition (Module 22) | Provides to | Emits `recognition.event.created` (with optional `suggested_points`) for Gamification to credit points/badges; this module owns no points ledger. |
| Identity & Organization (Module 01) | Consumes | `OrganizationNode` tree and `UserScope` for targeting, for the anonymity-safe aggregation query's grouping dimensions, and for authorizing who may view attributed individual responses or scoped aggregates. |
| Mobile & Offline (Module 04) | Consumes | Offline queuing/sync for survey responses (queued draft → idempotent sync) and recognition posts, reusing the standard sync protocol (source-spec §19). |

## 9. Resolved Decisions

- **Minimum group size for anonymous aggregation is N = 5 respondents**, tenant-configurable upward,
  with a **hard floor of 5** enforced platform-wide regardless of tenant configuration (a tenant
  cannot set N=1 or N=2 and functionally defeat anonymity). Rationale: N=5 is a common industry
  convention balancing statistical usefulness against re-identification risk in small frontline teams
  (e.g., a 4-person overnight shift); tenants with larger sites may reasonably raise it, but no
  legitimate business need justifies lowering it below a privacy-preserving floor. This is the same
  N=5 figure fixed platform-wide by `_context/engineering-decisions.md` §12 as part of this module's
  EU AI Act high-risk-classification exemption — the two are the same decision, not independently
  configurable.
- **The "trend-only participation rate" carve-out (§6, small-group visibility tension) defaults to
  on.** Participation rate alone (not sentiment) poses materially lower re-identification/sensitivity
  risk than a rating or open-text aggregate, so showing it even for a below-threshold group is a safe
  default; a tenant with a stricter jurisdictional posture (per local workplace-morale-data rules) can
  turn it off. This is a firm platform default, not a placeholder pending review — a tenant electing
  a stricter posture is a configuration change, not a prerequisite to shipping.
- **`EngagementScore` composite weights (§FR-ENG-026) are a platform default that tenants may
  override; the composite is not, and will never be, presented as a validated psychometric
  instrument.** It is marketed and documented to customers as a configurable operational signal only,
  which avoids overclaiming scientific rigor the platform cannot substantiate and avoids the
  regulatory exposure of an unvalidated instrument being relied on for employment decisions.
- **Recognition events are never anonymous, by permanent design (§FR-ENG-025).** An "anonymous
  recognition" feature (e.g., anonymous peer shout-outs) is out of scope for this module and is not
  planned as a future addition to it: social recognition (wants attribution) and survey response
  (wants anonymity) are two different trust models that must not be conflated in one data model. If a
  genuine anonymous-recognition need emerges, it is scoped as its own feature with its own privacy
  review, not an extension of this module's anonymity mechanism.
- **HR/People Analytics has no break-glass path to below-threshold data, under any circumstance,
  including a harassment investigation.** Such a need is served by a completely separate,
  legally-governed HR case-management capability outside this module, never by a backdoor in the
  engagement aggregation query. This is a permanent, non-negotiable design constraint: any backdoor,
  however narrowly scoped or well-intentioned, undermines the anonymity promise made to every
  respondent, including those uninvolved in any investigation, and would make every future anonymous
  response provably less trustworthy the moment its existence became known.
