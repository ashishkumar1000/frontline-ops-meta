# Analytics & Control Tower — Functional Specification

Module corresponds to **EPIC-12 (Control Tower)** and **EPIC-15 (Analytics & ML)**. Source:
source-spec.md §14 (Analytics and Control Tower, all subsections), Capability Map §5 ("Predictive
Risk" row), §26 (Success Metrics and Operating Model), Appendix E EPIC-12 and EPIC-15.

## 1. Purpose and Scope

### 1.1 Purpose

This module answers, for every role from Store Manager to Executive, the seven Executive Questions
(source-spec §14.1): what is wrong, where, how serious, why, what action has already been taken,
what needs human intervention, and whether the intervention is working. It is the analytical and
predictive layer of the Operational Graph — it does not generate operational state, it observes,
aggregates, explains, and forecasts it. It owns the enterprise KPI set (§14.2), the three-tier
Control Tower UI (§14.3), the analytics warehouse/lakehouse, and the predictive risk/anomaly
detection models that turn raw operational history into forward-looking guidance.

### 1.2 In Scope

- Exact computation logic, segmentation dimensions, and freshness tier for every KPI in the §14.2
  KPI set, plus the North-Star Metric (Verified Operational Execution Rate, §26.1) and the §26.2
  Product Health metric families where they surface in a tenant-facing dashboard (as distinct from
  internal platform telemetry, which is out of scope — see §1.3).
- The three Control Tower views — Executive, Regional/District Manager, Store/Site Manager — each
  with persona-appropriate widgets, drill-down, and time-range/segmentation controls.
- Drill-down / root-cause navigation from an enterprise- or region-level KPI anomaly down to
  contributing locations, work items, issues, and (where Visual AI or Agent Runtime evidence
  exists) the underlying evidence records.
- Alerting and threshold configuration: who is notified when a metric breaches a configured
  threshold, at what severity, through which channel (via Notification Hub).
- Predictive risk model outputs: location risk scores, forecasted KPI trajectories, anomaly flags
  on KPI time series, and the recommended-action surface shown to a manager.
- Scheduled and on-demand exports (CSV/XLSX, PDF snapshot, scheduled email/Slack/Teams digest) and
  the metrics query API consumed by BI integrations (Power BI/Tableau/Snowflake-class, per
  source-spec §15) and by the Executive Insights Agent.
- The analytics warehouse/lakehouse: bronze/silver/gold layered ingestion of every domain event and
  CDC stream, and the semantic metrics layer that sits in front of it.

### 1.3 Out of Scope (owned by other modules, referenced here)

- Source-of-truth operational data (WorkItem, Issue, Submission, Evidence, AgentRun, etc.) —
  produced and owned by Work Orchestration (EPIC-03), Forms (EPIC-04), Evidence (EPIC-06),
  Issues/CAPA (EPIC-07), Visual AI (EPIC-10), Agent Runtime (EPIC-11). This module only ever holds
  **derived** facts and aggregates; a correction to a fact must happen at the source and flow
  through CDC, never be edited directly in the warehouse.
- Notification delivery mechanics (push/email/SMS fan-out, retries, preference resolution) —
  Notification Hub. This module only decides *that* an alert should fire and *what* it contains.
  and shows how it renders and lets a manager acknowledge/resolve it in-app.
- Agent reasoning, tool execution, and approval workflow for the Executive Insights Agent itself
  (prompt construction, model calls, autonomy level enforcement) — Agent Runtime (EPIC-11). This
  module exposes the metrics query API the agent calls as a retrieval tool and validates that agent
  queries are scope-authorized like any other caller.
- Internal platform/infra observability (API latency, event lag, DLQ depth, sync success) — owned
  by Observability & Reliability (source-spec §22); this is *platform* telemetry, not *tenant*
  operational analytics, and lives in a separate technical monitoring stack even though some of the
  same event volume overlaps.
- Organization hierarchy and effective-dating mechanics — Identity/Org module (EPIC-01/02). This
  module consumes `OrganizationNode` effective-dated snapshots for rollup attribution but does not
  own hierarchy edits.

## 2. Personas Served

| Persona | Primary Control Tower surface | Interaction |
|---|---|---|
| Executive | Executive Control Tower | Enterprise health score, critical exceptions, at-risk locations, repeated failure patterns, board-level trend export |
| Regional/District Manager | Manager Control Tower (regional scope) | Territory location-health ranking, exception queue for owned scope, drill-down into underperforming stores, risk forecast for territory |
| Store/Site Manager | Manager Workspace — Location Health tab | Own-location KPI trend, open exceptions, verification backlog, comparison to peer/region benchmark |
| Operations Admin | Control Tower configuration console | Defines KPI thresholds, alert routing rules, dashboard widget sets per role, scheduled report subscriptions |
| Executive Insights Agent | Metrics Query API (as consumer) + Control Tower narrative injection (as co-producer) | Retrieves KPI/segment data as a retrieval tool (`get_kpi_snapshot`, `get_location_risk`, `get_repeated_failure_patterns`); writes back narrative annotations attached to a KPI point ("why this moved") surfaced as an explain affordance in the UI |

Enterprise Admin and Auditor/Inspector consume Control Tower views read-only within their scope but
have no module-specific configuration surface beyond what Operations Admin defines.

## 3. KPI Set — Definitions, Computation, Segmentation

All KPIs below are computed in the gold semantic layer (see high-level-design.md §4) and exposed
through the metrics query API (see low-level-design.md §3). Every KPI carries a `metric_definition`
version (see §8, Validation) so historical values remain interpretable after a definition change.

### 3.1 North-Star Metric

**FR-ANLY-001 — Verified Operational Execution Rate (VOER).**
`VOER = count(WorkItems in state VERIFIED within period) / count(WorkItems expected within period)`.

- **Numerator**: WorkItems whose terminal state for the period is `VERIFIED` (i.e. completed *and*
  passed configured evidence/approval criteria per source-spec §7/§17.2), attributed to the period
  by `due_at` (not `verified_at`) so late verifications still count against the period the work was
  expected in, capped by a configurable `verification_grace_period` (default 7 days) after which an
  un-verified item is scored as not-verified for that period even if later verified (and separately
  tracked as a late-verification data point).
- **Denominator**: "work expected" = all WorkItems materialized for the period whose type is
  configured as `counts_toward_voer = true` (excludes, by default, `ANNOUNCEMENT_ACKNOWLEDGEMENT`
  and `TRAINING_ASSIGNMENT`; tenant-configurable), excluding `CANCELLED` items with a
  `cancellation_reason` in the tenant's configured exclusion set (e.g. `LOCATION_CLOSED`,
  `DUPLICATE_SCHEDULE`).
- **Segmentation dimensions** (mandatory, per source-spec §26.1): `location` (and rollups up the
  `OrganizationNode` tree), `workflow_type` (WorkItem type / template), `risk_band` (from
  Visual AI / form risk scoring where present), `evidence_type` (photo/signature/form/sensor),
  `verification_method` (manual approval vs. auto-verified vs. AI-assisted). Additional optional
  cuts: `time_of_day`, `shift`, `assignee_role`.
- **Grain**: computed at `location x workflow_type x day` in the gold fact table, then rolled up.

### 3.2 KPI Table (source-spec §14.2, extended with computation and segmentation)

| Metric | Formula | Segmentation dimensions | Freshness tier | Primary audience |
|---|---|---|---|---|
| Verified Operational Execution Rate | See FR-ANLY-001 | location, workflow_type, risk_band, evidence_type, verification_method | T-1 (daily batch) + near-real-time provisional | Executive |
| On-Time Completion Rate | `count(completed_at <= due_at) / count(completed)` where `completed` = state in {SUBMITTED, APPROVED, VERIFIED} reached at least once | location, workflow_type, assignee_role, shift | Near-real-time (<15 min) | Operations |
| Repeat Failure Rate | `count(Issues where repeat_flag = true) / count(Issues total)`; `repeat_flag` set when a new Issue's `(location_id, failure_category)` matches a prior Issue within a configurable lookback window (default 90 days) — see anomaly/pattern pseudocode in low-level-design.md §5.2 | location, failure_category, root_cause_method | T-1 | Quality/Operations |
| Exception Resolution Time | `avg/p50/p90(resolved_at - opened_at)` for Issues reaching `RESOLVED` in period | location, severity, failure_category | T-1 | Managers |
| Time to Intervention | `avg/p50/p90(first_corrective_action_created_at - issue_opened_at)` | location, severity | T-1 | Executives |
| Automation Rate | `count(eligible WorkItems completed via approved agent autonomy L2/L3 execution) / count(eligible WorkItems)`; "eligible" = WorkItem type flagged `automation_eligible = true` in policy | location, work_item_type, agent | T-1 | Digital/AI |
| Frontline Adoption | `count(distinct active frontline users in period) / count(provisioned frontline users)`; "active" = at least one WorkItem state transition or app session in period | location, role | Daily | Product |
| AI Override Rate | `count(AgentRun recommendations with human_approval in {REJECTED, MODIFIED}) / count(AgentRun recommendations issued)` | agent, autonomy_level, tenant | T-1 | AI Governance |

### 3.3 Derived Composite — Enterprise Health Score

**FR-ANLY-002 — Enterprise Health Score.** A single 0–100 index shown at the top of the Executive
Control Tower (source-spec §14.3: "92.4% verified compliance"). Computed as a weighted composite:
`0.5 * VOER + 0.2 * (1 - normalized Repeat Failure Rate) + 0.2 * On-Time Completion Rate + 0.1 *
(1 - normalized open critical-severity Issue density per location)`. Weights are tenant-configurable
(admin console, Operations Admin only) with the above as system default; every change to weights is
versioned (`health_score_formula_version`) and a re-weighted historical score is *not* silently
backfilled — see §8.1 Validation. Displayed with week-over-week delta (`▼ 1.8% week-over-week`
per source-spec mock) computed against the trailing 7-day average, not a single prior point, to
reduce day-of-week noise.

### 3.4 Supporting Counters Shown on the Control Tower Header

- `critical_exceptions_count` — open Issues with `severity = CRITICAL` in scope, live count
  (near-real-time tier).
- `locations_at_risk_count` — locations whose current predictive risk score (§6) is in the
  `HIGH`/`CRITICAL` band, T-1 tier (risk scores recompute daily; see background jobs in
  low-level-design.md §8).
- `repeated_failure_patterns_count` — distinct `(location_cluster or workflow_type, failure_category)`
  pairs currently flagged as a repeated pattern (§3.2 Repeat Failure Rate mechanism, clustered — see
  FR-ANLY-010), T-1 tier.
- `critical_issues_unacknowledged_count` — critical Issues past their configured acknowledgement SLA
  with no `TRIAGED` transition yet, near-real-time tier (sourced directly from Issues/CAPA events,
  bypassing the warehouse for freshness — see dependencies §9).

## 4. Control Tower Views

### 4.1 Executive Control Tower

**FR-ANLY-003.** Enterprise-wide scope (root `OrganizationNode` or the executive's assigned
top-level scope). Required widgets:
- Enterprise Health Score tile with trend sparkline and week-over-week delta.
- Header counters (§3.4): critical exceptions, locations at risk, repeated failure patterns,
  unacknowledged critical issues — each clickable, opening the corresponding filtered list.
- VOER trend chart (13-week rolling), segmentable by business unit / brand / region.
- Top-10 and Bottom-10 locations by VOER, with sample-size indicator (see §8.3).
- Repeat Failure Pareto (top failure categories by weighted impact — frequency x severity).
- Risk forecast panel: locations predicted to cross into HIGH risk in the next 7/30 days
  (§6), each with the top 2 contributing drivers and a recommended action.
- AI Governance strip: Automation Rate, AI Override Rate, trend.
- "Explain this" affordance on the Health Score and on any KPI point, invoking the Executive
  Insights Agent narrative (§2, agent as co-producer) — always labeled as AI-generated with a
  confidence indicator and a link to the underlying data it cites.
- Scheduled report subscription control (§7).

### 4.2 Regional/District Manager Control Tower

**FR-ANLY-004.** Scoped to the manager's `UserScope` subtree. Required widgets:
- Territory Health Score (same formula, scoped) with delta vs. prior period and vs. enterprise
  average.
- Location Health ranked list/heatmap for all locations in scope, sortable by VOER, Repeat Failure
  Rate, open critical Issues, or risk score; color-banded (green/amber/red) against configurable
  thresholds.
- Exception queue: open Issues in scope, sorted by severity then age, with Time to Intervention and
  Exception Resolution Time shown per row and in aggregate.
- Repeated failure pattern list scoped to territory, each expandable to affected locations.
- Risk forecast for the territory: locations trending toward risk, with driver breakdown.
- Drill-down entry point into any location's Store view (§4.3) without leaving context.

### 4.3 Store/Site Manager — Location Health View

**FR-ANLY-005.** Single-location scope (the manager's assigned location(s)). Required widgets:
- Own-location VOER, On-Time Completion Rate, Repeat Failure Rate trend (13-week), each with peer
  benchmark (region/brand average) overlay.
- Open Issues list for the location with age, severity, SLA countdown.
- Verification backlog: WorkItems `SUBMITTED`/`PENDING_REVIEW` awaiting the manager's own action.
- "What needs my attention" ranked list — a rules + model-scored blend (open critical Issues first,
  then near-SLA-breach items, then items contributing most to the location's risk score) rather
  than a raw chronological list.
- Location risk score with the top contributing drivers (§6) in plain language, not raw model
  coefficients.

### 4.4 Drill-Down / Root-Cause Navigation

**FR-ANLY-006.** From any aggregate KPI point (an enterprise VOER dip, a territory heatmap cell, a
Pareto bar) the user can drill down through a fixed navigation path:
`Enterprise metric -> Business unit / Region -> Location -> WorkItem type / failure category ->
individual WorkItems / Issues -> Evidence`.
Each hop re-issues a scoped metrics/records query (never a client-side filter of an already-fetched
enterprise dataset, to preserve tenant/scope enforcement — see low-level-design.md §3.3). The final
hop links out to the owning module's record view (Work Orchestration WorkItem detail, Issues/CAPA
case view, Evidence viewer) rather than duplicating that UI inside Control Tower. Breadcrumbs
preserve the applied filter/segmentation state at every level so a manager can pivot dimensions
(e.g. switch from "by location" to "by workflow_type") without restarting the drill-down.

### 4.5 Alerting and Threshold Configuration

**FR-ANLY-007.** Operations Admin configures, per tenant and optionally per business unit/region:
- A threshold per KPI (e.g. VOER < 85% at location grain, Repeat Failure Rate > 20%) with severity
  (WARNING/CRITICAL) and evaluation window (instantaneous vs. sustained-for-N-days).
- Recipient rule: role-based (e.g. "location's assigned Store Manager plus their reporting District
  Manager") or explicit user/distribution list.
- Delivery channel(s) via Notification Hub (in-app, push, email; Slack/Teams where Integration Hub
  connector is active).
- Suppression/snooze: an acknowledged alert for a given (metric, scope) does not re-fire for a
  configurable cool-down unless severity escalates.
- A breached threshold also raises `analytics.threshold_breached` on the Event Backbone (new
  canonical event owned by this module — see low-level-design.md §4) so Agent Runtime or other
  modules can subscribe without polling the dashboard API.

## 5. Predictive Risk and Forecasting

**FR-ANLY-008 — Location Risk Score.** A 0–100 score per location, recomputed daily (T-1), blending:
recent VOER trend, open/critical Issue density, repeat-failure pattern membership, days-since-last
verified audit, and (where available) Visual AI compliance-score trend. Displayed as a band
(LOW/MEDIUM/HIGH/CRITICAL) with the top 2–3 contributing drivers stated in plain language (e.g.
"Repeat temperature exceptions (3 in 30 days); overdue safety audit (12 days)") — never only a raw
number, per Product Principle P8 (Explainability).

**FR-ANLY-009 — KPI Forecast.** For VOER, Repeat Failure Rate, and On-Time Completion Rate at
enterprise/region/location grain, the model produces a 7/30-day forward projection with a
confidence band. "Forecast usefulness" (the Capability Map §5 success signal for Predictive Risk)
is operationalized as three measured properties, each tracked as a model-quality metric independent
of the underlying KPI itself:
1. **Lead time** — days between a forecast crossing a risk threshold and the location actually
   crossing that threshold in reality (target: median lead time >= 5 days for a 30-day-horizon
   forecast that is later confirmed correct).
2. **Precision at alert** — of locations flagged HIGH/CRITICAL risk, the fraction that actually
   develop a CRITICAL Issue or drop below the VOER threshold within the forecast horizon.
3. **Action rate** — of forecast-driven alerts shown to a manager, the fraction that result in a
   manager-initiated WorkItem, Issue, or schedule change within 3 days (a forecast nobody acts on
   is not useful regardless of accuracy).
All three are shown on an internal Model Quality panel (Operations Admin / Enterprise Admin only,
not the Executive Control Tower) and used to gate model promotion (see high-level-design.md §7).

**FR-ANLY-010 — Repeated Failure Pattern Detection.** Clusters Issues by
`(location or location-cluster, failure_category, root_cause_theme)` using the lookback-window
matching in §3.2 plus a similarity pass over free-text root-cause notes (embedding-based, reusing
the platform's vector store) to catch same-cause issues logged under different categories. A
cluster becomes a "repeated failure pattern" once it reaches a configurable occurrence threshold
(default 3 within 90 days) at a single location, or reaches a lower threshold (default 5 within 90
days) across >= 3 locations sharing a common attribute (same brand, same equipment model, same
supplier) — surfacing cross-location systemic patterns, not just single-store repeat offenders.

**FR-ANLY-011 — Recommended Action Surface.** Every risk forecast or repeated-failure-pattern card
shown to a manager includes a `recommended_action` object: `{action_type, target, rationale,
confidence, evidence_refs[]}` where `action_type` is one of a fixed set
(`SCHEDULE_REINSPECTION`, `ASSIGN_CORRECTIVE_TASK`, `ASSIGN_TRAINING`, `ESCALATE_TO_REGIONAL`,
`REVIEW_EQUIPMENT`, `NO_ACTION_MONITOR`). The recommendation is generated by the same
deterministic-rule + statistical-model + LLM pattern used platform-wide (source-spec §13.3): a
rules layer maps known driver combinations to an `action_type`, the LLM (Executive Insights Agent
context) drafts the human-readable rationale from the cited evidence, and the manager can accept
(which creates the corresponding WorkItem/Issue via the Work Orchestration / Issues API, logged as
`agent_action_source = control_tower_recommendation`) or dismiss with a reason, which is fed back
as a labeled training example for the recommendation-quality metric (§5, action rate).

## 6. Exports and Scheduled Reports

**FR-ANLY-012.** Any Control Tower view can be exported on demand as PDF (visual snapshot) or
CSV/XLSX (underlying data at the current drill-down grain and filter state). **FR-ANLY-013.** A
user with export permission can create a scheduled report: a saved view + filter + schedule
(daily/weekly/monthly, timezone-aware) + recipient list + delivery channel (email attachment,
Slack/Teams post via Integration Hub, or a landing link into the live dashboard). Scheduled reports
respect the requester's scope at send time (re-evaluated on each run, not frozen at creation, so a
manager who loses scope to a location stops receiving location data in the next run).
**FR-ANLY-014.** All exports and scheduled report deliveries are recorded in `AuditLog` (who
exported what scope/filter, when) since exported data can leave the platform's access-control
boundary.

## 7. UX States

**FR-ANLY-015 — Dashboard loading.** Skeleton widgets render immediately with cached last-known
values (if any exist client-side or in the query cache) faded at reduced opacity while the live
query resolves; a spinner-only blank state is used only on a true first load with no cache.

**FR-ANLY-016 — Partial data while the warehouse lags.** If a KPI's underlying gold-layer partition
for part of the requested period has not yet landed (see CDC-lag failure mode in
high-level-design.md §8), the widget renders the available partial period, visually truncates the
trend line at the last complete data point, and shows an inline label: "Data through {date};
{today}'s figures pending (usually available by {freshness SLA time})." The KPI is never
silently computed over a partial period and presented as if final — a `data_completeness` flag
accompanies every KPI response payload (see low-level-design.md §3).

**FR-ANLY-017 — Stale-data warning banner.** If the freshness SLA for the current view's tier is
breached (e.g. T-1 data older than 30 hours, or near-real-time data older than 30 minutes — see
NFRs §10), a persistent banner reads "Data last updated {timestamp} — refresh delayed" with a
manual refresh action, and affected numeric tiles are visually marked (subdued color + a small
clock icon) rather than silently shown as current.

**FR-ANLY-018 — Drill-down empty state.** When a drill-down hop returns zero contributing records
(e.g. an enterprise-level anomaly whose contributing locations no longer show the condition because
data has since improved, or a filter combination with no matches), show "No contributing records
found for this view at the current filter" plus the applied filter chips and a one-click "broaden
to parent scope" action, rather than a bare empty table.

**FR-ANLY-019 — Forecast confidence display.** Every forecast value is shown as a point estimate
plus a shaded confidence band on the chart, and every risk score/recommended action carries a
discrete confidence label (LOW/MEDIUM/HIGH, mapped from the underlying model's confidence interval
width) — never a bare unqualified number, consistent with Product Principle P8. A forecast whose
underlying model has not retrained within its expected cadence (§ background jobs,
low-level-design.md §8) is marked "Forecast based on model last trained {date}" so a manager can
discount a stale model.

**FR-ANLY-020 — Permission-denied / scope-empty state.** A user whose scope contains zero
locations with data for the selected period (e.g. newly onboarded region, or a scope reorganization
just applied) sees "No data in your current scope for this period" distinct from a genuine zero-value
metric, to avoid a manager reading "no data" as "0% compliance."

## 8. Validation and Edge Cases

### 8.1 Metric Definition Changes Mid-Period

**FR-ANLY-021.** Every KPI formula (including the Enterprise Health Score weighting, §3.3) is a
versioned `metric_definition` row (low-level-design.md §3). Changing a formula, an inclusion/
exclusion rule (e.g. which WorkItem types "count toward VOER"), or a threshold band creates a new
`metric_definition_version`, effective from a configured `effective_from` timestamp — never an
in-place mutation. Historical dashboard queries for a period always resolve against the
`metric_definition_version` that was effective for that period, so a chart spanning a definition
change shows a visible version-boundary marker on the timeline and a tooltip explaining what
changed. A tenant may explicitly request a full historical recompute under the new definition
(admin action, async job, produces a *new* named metric series, e.g. `voer_v2`, rather than
overwriting `voer_v1` history) — recompute is opt-in, never automatic, because the choice of
whether historical comparability or definition currency matters more is a tenant/business decision.

### 8.2 Location Reorg Mid-Period

**FR-ANLY-022.** Because `OrganizationNode` is effective-dated (source-spec §17.1), a location that
moves between regions mid-period is attributed to its *effective-dated parent at the time each
underlying event occurred* (per source-spec Reorg acceptance test §24.1: "Location moves between
regions without changing historical attribution for prior events"). Gold-layer facts join to a
`dim_location` snapshot keyed by `(location_id, valid_from, valid_to)`, not to the current
hierarchy, so a rollup for "Region A, last quarter" is stable even after the location later moves
to Region B. The Control Tower UI, when a manager views a rollup that spans a reorg boundary, shows
a footnote: "Includes N days of history attributed to prior organizational structure."

### 8.3 Small-Sample-Size Statistical Noise

**FR-ANLY-023.** Any KPI or risk score computed over fewer than a configurable minimum sample size
(default: 10 WorkItems or 5 Issues in the period at that grain) is flagged `low_sample_size = true`
in the API response and rendered in the UI with a distinct visual treatment (dashed border / muted
color) plus a tooltip: "Based on {N} records — may not be statistically reliable." Such
low-sample-size locations are excluded by default from Top-10/Bottom-10 ranking widgets (§4.1) to
avoid a single-digit-volume location dominating a ranking by chance, with an explicit toggle to
include them. Anomaly detection (low-level-design.md §5.3) applies a minimum-volume gate before
flagging a location-level anomaly, to avoid false positives from noise at very low transaction
counts.

### 8.4 Real-Time Dashboard vs. T-1 Warehouse Latency

**FR-ANLY-024.** Not every KPI can or should be near-real-time. The freshness tier (§3.2 table,
§10 NFRs) is an explicit product decision per metric: counters that gate escalation/safety behavior
(critical exception count, unacknowledged critical issues) are sourced near-real-time directly from
operational service events (bypassing full warehouse aggregation latency) per §3.4; KPIs that
require complex joins/dedup across many domains (VOER, Repeat Failure Rate) are computed once daily
in the gold layer (T-1) because computing them online on every dashboard load does not scale and
would risk inconsistent numbers between two managers viewing the same metric seconds apart. This
distinction is always visible to the user via a small tier badge on each widget ("Live" vs. "Updated
daily") rather than left implicit.

## 9. Non-Functional Requirements

| Requirement | Target | Notes |
|---|---|---|
| Dashboard initial load (cached/warm) | p95 < 2.5s to first meaningful render | Applies to Executive/Regional/Store Control Tower default view |
| Dashboard initial load (cold, no cache) | p95 < 5s | |
| Drill-down query response | p95 < 1.5s per hop | Each hop is a discrete scoped query, not a client-side re-filter |
| Near-real-time metric freshness | <= 15 minutes end-to-end (event occurrence to dashboard-visible) | Applies to On-Time Completion Rate, critical exception counters |
| T-1 batch metric freshness | Available by 06:00 tenant-local time for the prior full day | Applies to VOER, Repeat Failure Rate, Exception Resolution Time, Automation Rate, AI Override Rate, risk scores, forecasts |
| Forecast/model retrain cadence | Weekly for risk scores, monthly for long-horizon forecast models, or immediately on a detected material drift alert | See background jobs, low-level-design.md §8 |
| Export generation | p95 < 30s for a standard scoped export; async job + notification for exports estimated over 30s | |
| Concurrent dashboard sessions | Design target: 10,000 concurrent tenant dashboard sessions enterprise-wide at platform scale (100k locations) without degrading p95 | |
| Data isolation | Zero cross-tenant data in any aggregate, cache entry, or export — see Security §21.1 baseline, enforced additionally at the warehouse layer (row-level security / tenant-schema isolation) | |

## 10. Dependencies

This module is a **consumer of nearly every other module's domain events** via CDC and the Event
Backbone, and a lightweight producer back into the platform (recommended-action WorkItem/Issue
creation, threshold-breach events). Key dependencies:

- **Work Orchestration (EPIC-03)** — `task.*` events and WorkItem CDC stream feed VOER, On-Time
  Completion Rate, Automation Rate, Frontline Adoption computations.
- **Issues/CAPA (EPIC-07)** — `issue.*` events and Issue CDC stream feed Repeat Failure Rate,
  Exception Resolution Time, Time to Intervention, critical-exception counters, repeated-failure
  pattern detection.
- **Visual AI (EPIC-10)** — compliance-score and violation events feed the risk model's visual-
  compliance driver and location health drill-down evidence links.
- **Agentic Operations / Agent Runtime (EPIC-11)** — `agent.*` events (AgentRun, tool execution,
  approval outcomes) feed Automation Rate, AI Override Rate, and the AI Governance strip; this
  module also exposes the metrics query API back to the Executive Insights Agent as a retrieval
  tool, making this the one module with a true bidirectional relationship with Agent Runtime.
- **Communications, Knowledge and Learning (EPIC-08/09)** — `message.acknowledged` and
  `training.completed` events feed engagement metrics (announcement acknowledgement rate, training
  completion) shown on Product Health panels and feed the repeated-failure-to-training-outcome loop
  (source-spec §11.3) as an outcome-measurement input.
- **Identity/Org (EPIC-01/02)** — `OrganizationNode` and `UserScope` effective-dated snapshots are
  consumed (not owned) for rollup attribution and dashboard scope enforcement.
- **Evidence (EPIC-06)** — Evidence metadata (not raw media) is referenced from drill-down/root-
  cause navigation as `evidence_refs`; raw media is fetched via the Evidence module's signed-URL API
  at UI render time, never duplicated into the warehouse.
- **Notification Hub** — consumes this module's `analytics.threshold_breached` events and scheduled
  report jobs to deliver alerts/reports through the appropriate channel.
- **Integration Hub (EPIC-13)** — the metrics query API is the basis for BI connector exports
  (Power BI/Tableau/Snowflake-class) referenced in source-spec §15.

## 11. Resolved Decisions

- **Lakehouse platform vendor**: Databricks, one vendor globally across every deployment region — it
  unifies batch ETL and ML forecasting workloads (used by this module's pattern-detection/forecast
  features) in a single platform, and a single vendor avoids maintaining region-specific
  warehouse-SQL dialect variants of every metrics-layer query this module defines. This is a firm
  platform choice, not a per-region option, consistent with conventions.md's "Snowflake/BigQuery/
  Databricks-class" framing (the class was named there; this register-adjacent decision picks the
  member of that class).
- **Near-real-time critical counters sourced from operational read replicas/events**: firm
  architecture, not a flagged trade-off — these specific counters bypass the warehouse to meet the
  15-minute SLA, and are kept numerically reconcilable with the T-1 warehouse view of the same
  underlying Issues via the nightly reconciliation job (low-level-design.md §8). A reconciliation
  discrepancy above the job's defined tolerance pages the data-platform on-call per
  engineering-decisions.md §9 alerting rules, rather than being silently absorbed.
- **Repeat Failure Rate / pattern-detection lookback window**: fixed at **90 days**, platform-wide
  default, tenant-configurable per vertical. 90 days balances enough history to distinguish a real
  repeat-failure pattern from noise against staying recent enough that a pattern reflects current
  operating conditions; verticals with materially different audit cadences (e.g. healthcare-adjacent
  quality programs with longer regulatory review cycles) override it at the tenant level rather than
  changing the platform default.
- **Enterprise Health Score formula and weights**: platform default with tenant override (as
  specified in §3.3) — not fully tenant-defined from onboarding. This gives every new tenant a
  working, comparable score on day one (important for the cross-tenant benchmarking use cases this
  KPI feeds) while still letting a tenant with a materially different operating model reweight it.
- **Historical VOER/Repeat Failure Rate backfill for a migrating tenant**: backfilled for the lesser
  of the tenant's available legacy-system export history or **12 months**, at onboarding, run as a
  one-time bulk-load job before the tenant's first live dashboard render. 12 months is chosen because
  it covers a full seasonal cycle for retail/QSR operating patterns, which is what makes Top/Bottom-10
  rankings and trend charts meaningful rather than noise-dominated in month one; a tenant with less
  than 12 months of exportable legacy history backfills what it has and the dashboard clearly labels
  the shorter trailing window until 12 months of native data accumulates.
- **Custom/ad-hoc metric builder for Operations Admins**: out of scope for this baseline module and
  not scheduled on the current roadmap; not a decision this pass reopens. The semantic metrics layer
  (high-level-design.md §4) is deliberately designed so this is additive if and when it is prioritized,
  not a rearchitecture — no further action is needed in this spec package for that to remain true.
