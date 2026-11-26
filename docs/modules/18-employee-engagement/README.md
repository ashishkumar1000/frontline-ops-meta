# Module 18 — Employee Engagement & Voice of Frontline

> Gives frontline workers a governed, two-way voice channel: short recurring pulse surveys they can
> answer anonymously, public peer-and-manager recognition posts, and a computed engagement score per
> location and team. It exists because the people working the floor usually have no safe, structured
> way to say how work is going — and because morale is a leading indicator of turnover and
> operational problems leadership otherwise sees too late.

| | |
|---|---|
| **Module code** | **ENG** (FR-ENG-###) |
| **Owning service** | Engagement Service |
| **Tier** | 2 (competitive gap closure — `new-modules-addendum.md` row 18) |

## In plain terms
- An admin builds a pulse survey from simple question types — ratings, multiple choice, short free text — plus a dedicated eNPS question (the standard 0–10 "how likely are you to recommend working here" measure, scored as promoters minus detractors).
- Frontline associates answer in the mobile app, anonymously or with their name attached, whichever mode the survey promised; anonymity mode can never be switched after the survey opens.
- The trust rule: results are only shown for groups of at least 5 respondents (a hard platform floor no tenant can lower), and drilling into a small subgroup — say a 3-person night shift — shows an explicit "suppressed" state instead of numbers, so nobody can work backwards to who said what.
- Coworkers and managers give each other recognition in a feed ("Teamwork," "Above & Beyond"); recognition can carry suggested points, which the Gamification module decides whether to award.
- A weekly EngagementScore per location blends participation, ratings, eNPS, and recognition activity, rolls up the org tree from store to enterprise, and feeds leadership dashboards — always threshold-suppressed at every level.
- HR gets no backdoor: there is no break-glass path to below-threshold anonymous data, under any circumstance, by permanent design.

## Key concepts
| Concept | What it means |
|---|---|
| `PulseSurvey` | A short recurring survey campaign with a type (pulse / eNPS / custom), an anonymity mode, and a close time; versioned by cloning once scheduled. |
| `SurveyQuestion` | One question, limited to rating, single/multi-select, open text, or the special 0–10 `ENPS` subtype (max one per survey). |
| `SurveyResponse` | One person's answers to one survey occurrence; in anonymous mode, stored with no identifier that can be traced back to the respondent. |
| Aggregation threshold | The minimum group size (default and hard floor: 5 respondents) below which no statistics for that group are ever rendered — enforced server-side, even on exports. |
| `RecognitionEvent` | A public, always-attributed recognition post (giver, recipient, message, category) — recognition is deliberately never anonymous, unlike surveys. |
| `EngagementScore` | The weighted composite metric per org node per period (default weights: ratings 40%, eNPS 30%, participation 15%, recognition 15%), tenant-configurable and versioned on recompute. |
| `PulseSurveyRun` | One occurrence of a recurring survey campaign; each run generates a per-recipient `Survey` WorkItem. |

## How it connects to the rest
- **Communications & Knowledge (Module 07)** — this module reuses Module 07's audience-targeting resolution and push/email/in-app delivery APIs rather than duplicating them; the docs describe the Engagement Service as "a client of Module 07," with no notification or audience engine of its own.
- **Work Orchestration (Module 02)** — this module is the *functional owner* of the existing `Survey` WorkItem type: it defines what a survey WorkItem's payload and submission mean, while Work Orchestration owns the generic state machine and scheduling primitives.
- **Forms & Workflow Builder (Module 03)** — reuses a constrained subset of the Forms component library for question rendering, but the docs deliberately do *not* use `TemplateVersion` for survey versioning — surveys are simpler and version via `PulseSurvey`/`SurveyQuestion` rows directly.
- **Gamification (Module 22)** — emits `recognition.event.created` (with optional `suggested_points`); this module keeps no points ledger, badges, or leaderboards.
- **Analytics & Control Tower (Module 11) and Identity & Organization (Module 01)** — produces already-threshold-safe, non-identifying engagement facts for dashboards and correlation analysis (e.g., engagement vs. repeat failures), and consumes the org tree and user scopes for targeting, grouping, and authorization.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-ENG-001–029: survey design, campaign targeting, the anonymity/aggregation-threshold rules, eNPS computation, recognition events, and engagement scoring, plus firm resolved decisions (no HR backdoor, no anonymous recognition). |
| [high-level-design.md](./high-level-design.md) | The Engagement Service's four bounded concerns, the defense-in-depth anonymity design (no query path to raw responses for dashboards), CDC exclusion of identifying tables, and failure degradation. |
| [low-level-design.md](./low-level-design.md) | Postgres schema (including the isolated dedupe-token table), REST endpoints, publish/response and threshold-gated-read sequences, and pseudocode for anonymity-safe aggregation and eNPS. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view with the anonymization boundary, survey-publish and anonymity-safe-read sequences, the owned-entities ER model, and the campaign lifecycle state machine. |
