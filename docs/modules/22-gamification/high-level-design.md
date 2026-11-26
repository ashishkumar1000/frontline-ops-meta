# Gamification & Recognition — High-Level Design

Module: `22-gamification`

## 1. Module Responsibilities

The **Gamification Service** is a small, purely **event-consumer** service. Unlike most core modules,
it is almost never called synchronously by other modules — it is not in the request path of a course
completion, an audit submission, or a recognition post. Its responsibilities:

1. **Badge Rule Evaluation** — subscribe to achievement-worthy events from Learning, Work
   Orchestration, and Employee Engagement; maintain per-user rolling counters/state per `BadgeRule`;
   issue `BadgeAward`s when conditions are met, idempotently.
2. **Points Ledger** — an append-only, source-attributed transaction log of points earned and
   reversed, derived balance computation, and manual-adjustment support for admins.
3. **Leaderboard Materialization** — precompute ranked views per scope/time-window from the points
   ledger and badge counts, respecting tenant-wide and per-employee opt-out.
4. **Pathway Progress Tracking** — project a learner's course/badge completions onto an ordered
   `Pathway` definition and award a capstone credential on completion.

It deliberately has **no significant outbound synchronous calls** into other modules' write paths: it
never blocks a course completion, an audit submission, or a recognition post to compute a badge. Its
only outbound calls are read-only (fetching `OrganizationNode`/`UserScope` for leaderboard scoping)
and to the platform Notification Hub for celebratory notifications. This reflects its Tier-3,
lighter-engagement-layer positioning per `new-modules-addendum.md`: it should be safe to disable or
degrade this entire service without any other module's core operational function being affected.

## 2. Component Diagram

```mermaid
flowchart TD
    subgraph UpstreamModules [Upstream Event Producers]
        Learning[Learning — Module 08\ntraining.completed]
        WorkOrch[Work Orchestration — Module 02\ntask.completed / audit.submitted / streak signal]
        Engagement[Employee Engagement — Module 18\nrecognition.event.created]
        IssuesCapa[Issues & CAPA — Module 06\nfraud/invalidation signal]
        IdentityOrg[Identity & Organization — Module 01\nuser.deactivated / role_changed / transfer]
    end

    Kafka[[Event Backbone — Kafka]]

    subgraph GamificationService [Gamification Service]
        Consumer[Event Consumer\n(idempotent, per-topic offsets)]
        RuleEvaluator[Badge Rule Evaluator]
        LedgerWriter[Points Ledger Writer]
        PathwayProjector[Pathway Progress Projector]
        AntiGaming[[Anti-Gaming Guard\nrate limits, reciprocal throttle]]
        Materializer[Leaderboard Materializer\n(scheduled batch)]
        Reconciler[Badge-Award Backfill\nReconciliation Job]
    end

    subgraph OwnedData [Owned Data — Postgres schema: gamification]
        DB1[(badges / badge_rules)]
        DB2[(badge_awards / badge_revocations)]
        DB3[(points_ledger_entries)]
        DB4[(leaderboards — materialized)]
        DB5[(pathways / pathway_steps)]
        DB6[(pathway_progress)]
        DB7[(user_gamification_preferences)]
    end

    subgraph Clients
        Mobile[Mobile App — Frontline]
        Web[Web Admin]
        BFF[API Gateway / BFF]
    end

    subgraph Downstream
        NotifHub[Notification Hub]
        Analytics[Analytics — Module 11]
        IdOrgRead[Identity & Org — read-only scope lookups]
    end

    Learning -- event --> Kafka
    WorkOrch -- event --> Kafka
    Engagement -- event --> Kafka
    IssuesCapa -- event --> Kafka
    IdentityOrg -- event --> Kafka
    Kafka --> Consumer

    Consumer --> RuleEvaluator
    Consumer --> LedgerWriter
    Consumer --> PathwayProjector
    RuleEvaluator --> AntiGaming
    LedgerWriter --> AntiGaming
    RuleEvaluator --> DB1
    RuleEvaluator --> DB2
    LedgerWriter --> DB3
    PathwayProjector --> DB5
    PathwayProjector --> DB6
    RuleEvaluator -. badge.awarded -.-> PathwayProjector

    Materializer --> DB3
    Materializer --> DB2
    Materializer --> DB7
    Materializer --> DB4
    Materializer --> IdOrgRead

    Reconciler --> Kafka
    Reconciler --> DB2
    Reconciler --> DB3

    RuleEvaluator -- badge.awarded --> NotifHub
    RuleEvaluator -- badge.awarded / points.credited / points.reversed --> Kafka
    Kafka --> Analytics

    Mobile --> BFF
    Web --> BFF
    BFF -- GET my badges/points/pathway --> DB3
    BFF -- GET my badges/points/pathway --> DB2
    BFF -- GET my badges/points/pathway --> DB6
    BFF -- GET leaderboard --> DB4
    BFF -- opt-out --> DB7
    BFF -- define badge rule / pathway --> DB1
    BFF -- define badge rule / pathway --> DB5
```

Narratively: `Learning`, `Work Orchestration`, `Employee Engagement`, `Issues & CAPA`, and
`Identity & Organization` never know Gamification exists — they publish their own canonical domain
events for their own reasons, and Gamification is simply one more subscriber. The service's only two
"real-time-ish" write paths (Badge Rule Evaluator, Points Ledger Writer) are both driven purely by
consumed events; the only scheduled/batch component (Leaderboard Materializer) is the one that
performs any meaningfully expensive computation, and it runs off the already-durable ledger/award
tables, never off raw upstream event replay.

## 3. Data Ownership

| Entity | Owned here | Notes |
|---|---|---|
| `Badge`, `BadgeRule`/`BadgeRuleVersion` | Yes | Definition and versioned award conditions. |
| `BadgeAward`, `BadgeRevocation` | Yes | Immutable award record + separate revocation record (append, never edit-in-place). |
| `PointsLedgerEntry` | Yes | Append-only transaction log; balance is a derived read, not a stored mutable field. |
| `Leaderboard` (materialized) | Yes | Precomputed ranked snapshots per scope/window; rebuilt, not incrementally patched in place, on each materialization cycle. |
| `Pathway`, `PathwayStep` | Yes | Ordered course/badge sequence definitions, versioned like `Badge`. |
| `PathwayProgress` | Yes | Per-user projection derived from consumed completion/award events. |
| `UserGamificationPreference` | Yes | Per-employee leaderboard opt-out and notification preferences for this module. |
| `Course`, `CourseVersion`, `Certification`, `LearningAssignment` | No (referenced) | Owned by Learning (module 08); Gamification reacts to their completion events only, never reads or writes their tables. |
| `WorkItem`, task-completion/streak facts | No (referenced) | Owned by Work Orchestration; consumed as events. |
| `RecognitionEvent` | No (referenced) | Owned by Employee Engagement; consumed as an event, never displayed or stored by Gamification beyond the `source_event_id` reference on a `PointsLedgerEntry`. |
| `OrganizationNode`, `UserScope`, `User` | No (referenced) | Owned by Identity & Organization; read-only for leaderboard scope resolution. |
| `Issue` | No (referenced) | Owned by Issues & CAPA; consumed as the trigger for points/badge reversal. |

## 4. External Interfaces

### Consumed (events — primary integration surface)

| Event | Producer | Purpose |
|---|---|---|
| `training.completed` | Learning (08) | Course-completion points credit; pathway-step advancement; input to `LEARNING`-category badge rules. |
| `audit.submitted` (with violation-count payload) | Work Orchestration (02) | Input to `COMPLIANCE`-category badge rules (e.g., zero-violation streaks). |
| `task.completed` / a streak-classification event | Work Orchestration (02) / Analytics (11) | Points/badge input for `CONSISTENCY`-category (streak) rules; Gamification consumes an already-quality-qualified streak signal rather than deriving one itself (functional-spec §7, §12). |
| `recognition.event.created` | Employee Engagement (18) | Optional points credit (`suggested_points`) and input to `RECOGNITION`-category badge rules; carries `rate_flagged` which Gamification honors and independently re-checks. |
| `issue.created` (fraud/invalidation-class, referencing a prior `audit.submitted`/`training.completed` event) | Issues & CAPA (06) / Learning (08) | Triggers points/badge reversal (functional-spec FR-GAM-064). |
| `user.deactivated`, `user.role_changed`, org-transfer signal | Identity & Organization (01) | Freezes in-flight badge counters/pathway progress; drives leaderboard-membership recalculation on next materialization. |

### Consumed (synchronous, read-only)

| Interface | Owner | Purpose |
|---|---|---|
| `GET /internal/v1/org/scope-resolve` | Identity & Organization | Resolve a user's current `location_id`/`team_id` for leaderboard scoping at materialization time. |

### Produced (events)

| Event | Consumer | Purpose |
|---|---|---|
| `badge.awarded` | Notification Hub, Analytics, Pathway Projector (internal) | Drives the celebratory notification and pathway-step advancement when a step is badge-bound. |
| `points.credited`, `points.reversed` | Analytics | Engagement-vs-outcome correlation facts. |
| `pathway.completed` | Notification Hub, Analytics | Capstone-credential milestone signal. |
| `leaderboard.materialized` | Analytics (optional) | Freshness/participation facts, not the leaderboard content itself (which is queried, not streamed). |

**No event, API, or export in this list is tagged or scoped for HR/performance-review consumption**
(functional-spec FR-GAM-027). The Analytics-facing events above carry engagement-correlation facts
only (e.g. "does leaderboard participation correlate with verified execution rate" at a
location/cohort level); this module implements no per-employee "performance summary" export and no
API scope an HR/performance system could integrate against, by design.

### Produced (REST, via BFF — see low-level-design.md §2)

- Badge rule / pathway admin CRUD, "my badges/points/pathway" read APIs, leaderboard read API,
  leaderboard opt-out API.

## 5. Scalability

- **Leaderboards are precomputed, never a live query.** At platform scale (100k locations, 1M+ users,
  source-spec §23), a tenant-wide "all employees ranked by points" query computed on-demand would be a
  full-table sort over a potentially multi-million-row ledger on every dashboard view — unacceptable.
  Instead, the **Leaderboard Materializer** runs on a schedule (NFR-GAM-02: every 15 min for
  weekly/monthly scopes, hourly for all-time) and writes a bounded `leaderboards` table
  (top-N rows + each requester's own rank, not every user's row for every scope) per `(tenant_id,
  scope_type, scope_id, time_window, metric)` combination. Read paths only ever hit this precomputed
  table plus a thin cache layer (low-level-design.md §6), never the raw ledger.
- **Materialization is scoped incrementally, not globally recomputed per cycle.** Only scopes with
  ledger activity since the last cycle are recomputed (`dirty scope` tracking keyed off
  `PointsLedgerEntry`/`BadgeAward` writes since last materialization timestamp), so a large tenant with
  activity concentrated in a subset of locations does not pay the cost of recomputing every location's
  leaderboard every cycle.
- **Badge Rule Evaluation is a streaming, per-user-keyed consumer**, partitioned by `tenant_id:user_id`
  (consistent with the platform's Kafka partitioning convention, conventions.md) so evaluation for
  different users scales horizontally with consumer parallelism and per-user rolling-window state
  updates are strictly ordered per user (no cross-partition coordination needed for a single user's
  rolling counters).
- **Rolling-window counters are maintained incrementally** (a sliding count backed by a small per-
  `(user_id, badge_rule_id)` window-bucket structure — see low-level-design.md), not recomputed by
  re-scanning historical events on every new event, so evaluation cost per event is O(1) amortized
  rather than O(window size).
- **Points-ledger reads (a user's own balance/history) scale trivially** — bounded per-user row counts,
  standard indexed reads; no special scaling treatment needed beyond the platform's standard read-
  replica/caching conventions.

## 6. Multi-Tenancy

Standard tenant isolation applies (`tenant_id` on every table + RLS, per conventions.md). Module-
specific notes:

- Every `BadgeRule`, `Pathway`, and `PointsRule` (point-value configuration) is tenant-scoped and
  independently configurable — there is no shared/global badge catalog across tenants in MVP (a
  cross-tenant template marketplace for badge/pathway definitions is a plausible Phase 2 extension of
  Module 24's App Marketplace pattern, not addressed further here).
- Leaderboard scoping additionally enforces the **same two-gate pattern** Employee Engagement
  established (module 18 HLD §6): RBAC/`UserScope` governs *which* leaderboard scope a viewer may
  request (a manager can only request their own location/team's leaderboard, an Enterprise Admin can
  request any), and the **small-group suppression floor** (functional-spec FR-GAM-025) is a second,
  independent gate on top of that — broad RBAC scope never overrides a too-small group's ranked-view
  suppression.
- Per-employee `leaderboard_opt_out` is tenant-independent per user record but the **tenant-wide
  `leaderboards_enabled` toggle** (functional-spec FR-GAM-022) is checked first and short-circuits
  everything else — a disabled tenant never materializes leaderboards at all, saving the compute
  entirely rather than materializing-then-hiding.

## 7. Failure Modes & Degradation

| Failure | Behavior |
|---|---|
| Missed event (consumer downtime, dead-letter, or a rare Kafka delivery gap) | The **Badge-Award Backfill Reconciliation Job** (low-level-design.md §8) periodically re-queries upstream modules' own event logs/outbox tables (via a bounded, paginated internal API, not a live stream re-subscription) for a lookback window (default 7 days) and re-evaluates badge rules idempotently against any events whose `event_id` has no corresponding processed-marker — this is the primary safety net for the "did we miss a badge" failure mode, run nightly regardless of whether an outage was detected, not only reactively. |
| Leaderboard Materializer job failure mid-run | Each `(scope, window)` cell is materialized independently and atomically (write-then-swap, not row-by-row mutation of a live table); a partial failure leaves the previous materialized snapshot serving reads (stale-but-consistent) rather than a half-updated or empty leaderboard — the freshness indicator (NFR-GAM-02) surfaces the staleness honestly. |
| Points Ledger Writer cannot reach the database (rare — same DB as the rest of the service) | The event remains unacknowledged on its Kafka partition and is retried per standard consumer backoff; because credit is idempotent on `(user_id, source_event_id, source_type)`, a retried write after a transient failure is always safe. |
| Anti-Gaming Guard component unavailable | The Badge Rule Evaluator and Points Ledger Writer **fail closed for crediting, not for the underlying event's acknowledgment** — i.e., the consumer still acknowledges/commits offset progress (it does not block the whole pipeline), but a would-be credit that cannot be rate-checked is queued into a short-lived retry buffer rather than credited without a check, bounded to a few minutes before it is either checked or explicitly logged as `anti_gaming_check_skipped: true` for later admin audit — chosen over blocking the pipeline indefinitely on a non-critical guard component. |
| Notification Hub unavailable at badge-award time | `BadgeAward` and the points credit are unaffected (they are already durably written before the notification call); the celebratory notification is retried via the Notification Hub's own retry/dedupe semantics, and the in-app "unseen award" flag (functional-spec §9) ensures the associate still sees the celebration on next app open even if the push notification itself was lost. |
| Upstream module (Learning/Work Orchestration/Employee Engagement) publishes malformed or unexpected event payload | Consumer routes the message to a dead-letter topic (platform-standard pattern, source-spec §22.3) rather than crashing the consumer group or silently dropping it; a monitoring alert fires on dead-letter volume for this module per the platform's standard DLQ observability convention. |
| Streak-classification signal (build-sequencing dependency resolved in functional-spec §12) not yet available from Work Orchestration/Analytics at a given tenant's rollout | `CONSISTENCY`-category badge rules and streak-sourced points simply have no qualifying events to evaluate — they degrade to "no streak badges awarded yet," not an error state; this is a deliberate soft dependency so Gamification's other categories (compliance, learning, recognition) can ship independently of that signal's availability. |

## 8. Technology Choices (module-specific, justified)

- **Kafka (existing event backbone)** as the sole real-time integration mechanism, with Gamification's
  own consumer group per topic — no new messaging technology, and critically, **no synchronous
  request/response coupling** into any upstream module's write path, which is the architectural
  expression of this module's "does not own... it subscribes" scope boundary (functional-spec §1.3).
- **Event-sourced points ledger (append-only `points_ledger_entries`, derived balance) rather than a
  mutable balance column.** This is chosen specifically for **auditability and safe reversibility**:
  when an upstream record is later invalidated (an audit found fraudulent), the only correct way to
  "undo" a credit without corrupting history is to append a compensating reversal entry, never to
  decrement a balance in place (which would destroy the evidence of what was reversed and why). This
  mirrors the platform's own historical-truth principle (source-spec P7) and the Points Ledger's design
  is intentionally closer to a financial ledger than to a simple game-score counter.
- **Postgres (existing platform datastore), standard row-partitioned by `tenant_id` + RLS** — no
  specialized store is justified. Ledger/award/leaderboard volumes are modest relative to, e.g.,
  Module 17's IoT time-series volume: even at 1M users platform-wide, per-user gamification activity
  is a handful of events per day at most, several orders of magnitude below sensor-reading throughput.
- **Materialized leaderboard tables refreshed by a scheduled batch worker**, not a live materialized
  view or an on-demand aggregation query, chosen for the scalability reasons in §5 — this is the one
  place this module accepts eventual consistency (bounded staleness, explicitly surfaced to the user)
  in exchange for keeping read latency flat regardless of tenant size.
- **Redis (existing platform cache)** in front of the materialized leaderboard table and a user's own
  points-balance read, for the same reasons as every other module's hot-read caching (low-level-
  design.md §6) — no new caching technology.
- **Temporal.io (existing platform workflow engine)** for the Leaderboard Materializer's scheduled runs
  and the Badge-Award Backfill Reconciliation Job, consistent with the platform's standard choice for
  durable, versioned scheduled execution (conventions.md) — not a bespoke cron.
