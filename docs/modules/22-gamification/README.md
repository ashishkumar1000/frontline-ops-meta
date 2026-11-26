# Module 22 — Gamification & Recognition

> This module makes good frontline work feel rewarding and visible: badges for things like ten
> perfect audits or a 30-day task streak, a points ledger fed by real completed work, team
> leaderboards, and visible progress through learning pathways. It changes nothing about what
> "correct execution" means anywhere in the platform — it just celebrates the completions other
> modules already verify. Associates earn badges and points as a byproduct of normal work;
> operations admins configure the rules; leaders watch engagement trends.

| | |
|---|---|
| **Module code** | GAM (`FR-GAM-###`) |
| **Owning service** | Gamification Service |
| **Tier** | 3 |

## In plain terms
- Earn badges automatically from real work events — e.g. "PERFECT_AUDITOR" for 10 zero-violation
  audits in a rolling 90 days — with no separate "gamification workflow" to learn.
- Collect points in an honest, append-only ledger: every earn and every reversal stays visible in
  the history, so the record is complete rather than curated.
- Appear (or not) on leaderboards: scoped to location, team, or tenant over weekly/monthly/all-time
  windows, with an unconditional per-employee opt-out and a tenant-wide off switch.
- Progress through learning pathways: an ordered sequence of courses and badges culminating in a
  capstone credential like "Certified Shift Lead."
- Trust the numbers: anti-gaming safeguards rate-limit how fast points can accrue, and if an
  underlying record is later invalidated (e.g. a fraudulent audit), the points and badges it earned
  are reversed with a visible, non-punitive treatment.

## Key concepts
| Concept | What it means |
|---|---|
| `Badge` / `BadgeRule` | A named achievement plus its versioned, declarative award condition (source event class, count threshold, rolling window, zero-defect qualifier). |
| `BadgeAward` | The immutable point-in-time record of who earned which badge via which rule version and which source events — revocations are separate records, never edits. |
| `PointsLedgerEntry` | One append-only points transaction (earn or reversal); the balance is a derived sum, never a mutable column. |
| `Leaderboard` | A precomputed, materialized ranked view per scope/time window/metric — rebuilt by a batch job, never computed per request. |
| `Pathway` / `PathwayProgress` | An ordered course/badge sequence toward a capstone credential, and the per-employee progress projection against it. |
| Anti-gaming guard | Rate limits and plausibility checks that blunt event replay, self-dealing recognition loops, and points farming. |

## How it connects to the rest
- **Pure event consumer by design** — Learning (module 08), Work Orchestration (module 02),
  Employee Engagement (module 18), and Issues & CAPA (module 06) publish their own events for
  their own reasons and never know Gamification exists; this module never polls them or blocks
  their write paths, and the whole service can be disabled without affecting any core operation.
- **Learning (module 08)** — badges and pathways layer on `Course`/`Certification` completion
  events; this module never defines what "completed a course" means or stores course content.
- **Employee Engagement (module 18)** — recognition events are an optional points trigger;
  the recognition posts themselves, and the ad-hoc manager/peer recognition experience, stay in
  module 18 (discretionary manager badges are deliberately Phase 2 to avoid blurring the two).
- **Issues & CAPA (module 06) and Evidence (module 05)** — invalidation/fraud signals trigger
  points/badge reversal; this module reacts, it never investigates fraud itself.
- **Identity & Organization (module 01) and Analytics (module 11)** — org scope drives leaderboard
  scoping (with an aggregation-threshold pattern reused conceptually from module 18); Analytics owns
  any correlation between gamification data and operational KPIs.
- **Core primitives** — the module reuses the canonical event envelope and never re-owns `WorkItem`
  or `Evidence` records; it keeps only `source_event_id` references.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-GAM-001–067 with [MVP]/[P2] tags covering badge rules, points ledger, leaderboards, pathways, anti-gaming, and reversal handling. |
| [high-level-design.md](./high-level-design.md) | The event-consumer service posture, component diagram, data ownership, and the "safe to disable" Tier-3 positioning. |
| [low-level-design.md](./low-level-design.md) | Postgres schema (badges, rules, counters, awards, ledger, leaderboards, pathways, preferences), REST endpoints, sequence diagrams, and idempotent rule-evaluation pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Component diagram of the event-driven service, rule-evaluation and leaderboard-materialization sequence diagrams, and the owned-data ER view. |
