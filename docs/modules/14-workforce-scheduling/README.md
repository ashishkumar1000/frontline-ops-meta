# Module 14 — Workforce Scheduling & Labor Management

> Builds and publishes the roster of who works which shift, at which location, in which role —
> and lets frontline staff set availability, request time off, and swap shifts without a text-message
> free-for-all. It solves the weekly "who is covering the floor?" problem that store and site
> managers otherwise run in spreadsheets, and warns them early when a time slot is understaffed.

| | |
|---|---|
| **Module code** | **WFS** (FR-WFS-###) |
| **Owning service** | Scheduling Service |
| **Tier** | 1 (competitive gap closure — `new-modules-addendum.md` row 14) |

## In plain terms
- A manager builds the week's schedule from reusable shift templates (e.g., "Opener, 07:00–15:00, 2 people"), edits it on a drag-and-drop grid, and publishes it — associates see their shifts the moment it publishes.
- Associates set recurring availability ("I can't work Tuesdays"), request time off for manager approval, and claim open shifts from a self-service pickup list.
- Associates swap shifts with eligible peers; the swap applies atomically to both people's schedules, and a manager approves only when tenant policy requires it (the default requires approval for cross-location swaps).
- The system detects coverage gaps — a time slot needing more staff than currently assigned — within 60 seconds of any change, and suggests a fillable candidate when one exists.
- A simple, rule-based labor forecast suggests headcount per day and role from historical patterns; a human always confirms the schedule (a forecast never auto-publishes).
- Labor-law rules (minimum rest between shifts, maximum weekly hours, minor-work limits) are configurable per jurisdiction, not hardcoded: hard rules block the action, softer rules warn and require an audit-logged manager override.

## Key concepts
| Concept | What it means |
|---|---|
| `ShiftTemplate` | A reusable shift recipe — role, location, time window, break rules, required headcount — that expands into concrete shifts for a period. |
| `Shift` | One concrete shift instance for one location and date. Draft shifts are manager-only; published shifts become visible to staff and drive downstream task creation. |
| `AvailabilityWindow` | When an employee can (or cannot) work — recurring patterns, one-off overrides, and manager-approved time off. |
| `ShiftAssignment` | The record of who is assigned to a shift, current and historical; exactly one assignee per shift. |
| `SwapRequest` | An associate's request to swap shifts with a peer or drop a shift for someone to pick up, with peer acceptance and policy-based manager approval. |
| `CoverageGap` | A computed shortfall: required headcount per (location, role, time slot) minus assigned headcount. |
| `LaborForecast` | A rule-based suggestion of how many people each location/date/role needs, derived from historical demand signals. |
| `LaborPolicy` | Tenant- and jurisdiction-scoped labor-law rules (rest periods, hour caps) evaluated at assignment, swap, and publish time. |

## How it connects to the rest
- **Work Orchestration (Module 02)** — the most important relationship. Publishing a shift emits `shift.published`; Work Orchestration's `SHIFT_BASED` schedule mode subscribes to that event to materialize shift-anchored `WorkItem`s (checklists, audits). The docs are emphatic that these are two different schedulers that must never be conflated: "the Scheduling Service schedules the roster; the Scheduling Engine schedules the checklist." They integrate only via events, never by reading each other's tables.
- **Identity & Organization (Module 01)** — consumes `User`, `Role`, `UserScope`, and `OrganizationNode` to decide who is eligible to work which role at which location, including multi-location employees and cross-location double-booking checks.
- **Time & Attendance (Module 15)** — consumes this module. Time & Attendance reads `Shift`/`ShiftAssignment` rows via a narrow read API as the "planned" side of its planned-vs-actual variance; this module never reads attendance data.
- **Integration Hub (Module 12) and Analytics (Module 11)** — optional POS/traffic connectors feed the labor forecast; shift/swap/coverage events feed labor-cost and coverage KPIs. Notification Hub (via events) delivers publish, swap, and gap alerts.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-WFS-001–099: shift templates, the draft-to-published lifecycle with a changes-since-last-publish diff, availability and time off, assignment/auto-fill, swaps, coverage gaps, forecasting, and the labor-law policy engine. |
| [high-level-design.md](./high-level-design.md) | The Scheduling Service's components, its strict one-way event contract with Work Orchestration, data ownership, scaling (partitioning, rolling-window template generation), and failure degradation. |
| [low-level-design.md](./low-level-design.md) | Postgres schema, REST endpoints (`/api/v1/shifts`, `/api/v1/swap-requests`, …), sequence diagrams, auto-fill ranking and rest-period policy pseudocode, caching and concurrency control. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component diagram, publish-to-Work-Orchestration and swap-request sequences, the ER model, and the `Shift` and `SwapRequest` state machines. |
