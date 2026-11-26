# Module 15 — Time & Attendance

> Lets frontline associates clock in and out of shifts — and start and end breaks — from the same
> mobile app they use for daily work, with a location check and optional photo capture to stop
> "buddy punching" (a coworker clocking in for someone who isn't there). It turns raw punches into
> manager-approved timesheets that export straight to payroll, so hours are accurate and auditable
> without spreadsheets or a separate time-clock device.

| | |
|---|---|
| **Module code** | **TA** (FR-TA-###) |
| **Owning service** | Time & Attendance Service |
| **Tier** | 1 (competitive gap closure — `new-modules-addendum.md` row 15) |

## In plain terms
- An associate taps Clock In / Clock Out / Start Break on the Today screen; the punch works even with no signal, because it queues offline and syncs later — a punch is never lost to a network gap.
- The app checks the punch is inside a geofence (a configurable radius around the work location, default 150 m); outside punches are blocked or allowed-but-flagged, per tenant policy.
- Optional anti-buddy-punching capture: a selfie stored as evidence, or an on-device fingerprint/Face ID check — never a server-side face-match database.
- A nightly job turns punches into timesheet lines with regular and overtime hours, using configurable, jurisdiction-aware overtime rules (daily/weekly thresholds, 7th-consecutive-day premiums, rounding) — not one country's law hardcoded.
- Problems become visible exceptions: missed punch, early clock-in, late clock-out, geofence violation. A missed punch becomes a correction request the manager can approve, reject, or modify — always with a full before/after audit trail.
- The manager reviews and approves the weekly timesheet; approved timesheets lock, export to the payroll system (Workday/ADP/SAP-class) via the Integration Hub, and this module never computes pay or tax itself.

## Key concepts
| Concept | What it means |
|---|---|
| `TimeClockEvent` | One raw punch: clock-in, clock-out, break start, or break end, with GPS result, capture method, and both device and server timestamps. |
| `GeofenceZone` | A named radius around a location; a punch passes if it lands inside any zone for that location. |
| `BreakPolicy` | Tenant rules for meal/rest breaks — mandatory or optional, paid or unpaid, minimum duration, and the shift length that triggers the requirement. |
| `Timesheet` | An employee's hours for one pay period, moving through DRAFT → PENDING_APPROVAL → APPROVED → EXPORTED; values are immutable once approved. |
| `TimesheetLine` | One day within a timesheet: worked minutes, break deductions, overtime, and any exception flags. |
| `CorrectionRequest` | A proposed fix for a missing or wrong punch (with reason code), which a manager approves, rejects, or edits before the affected day can be approved. |
| `OvertimeRule` | Versioned, per-jurisdiction configuration of overtime thresholds and multipliers; each timesheet line snapshots the rule version used, so rule changes never rewrite history. |

## How it connects to the rest
- **Workforce Scheduling (Module 14)** — the upstream "plan." This module reads planned `Shift`/`ShiftAssignment` rows to decide when clock-in is allowed, flag early/late/unscheduled punches, and show planned-vs-actual on the review screen. It never creates or edits shifts; Module 14 owns the plan, this module owns the actual.
- **Mobile & Offline (Module 04)** — punches are captured with the platform's standard offline queue, `client_uuid` idempotency, and sync contract; the server, not the device clock, is the authoritative time source (per source-spec §19.4 "client clock cannot be trusted").
- **Identity & Organization (Module 01)** — employee record, home location, and role/scope decide who may clock in where and who may approve whose timesheet.
- **Evidence (Module 05) and Integration Hub (Module 12)** — anti-buddy-punch photos are stored as standard `Evidence` records; `timesheet.approved`/`timesheet.exported` events and the export payload feed the payroll connector. The addendum is explicit: payroll integration is always an outbound Integration Hub connector — Time & Attendance owns timesheets, not a payroll engine.
- **Work Orchestration (Module 02)** — pattern reuse only: correction approval follows the platform's approve/reject/modify routing conventions, but `CorrectionRequest` is deliberately its own entity (it carries pay-affecting before/after values a generic WorkItem does not model), not a WorkItem subtype.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-TA-001–033: geofenced clock-in/out, multi-location punches, break policy, exception handling, timesheet approval, overtime configuration, plus resolved decisions (e.g., biometrics are device-local only). |
| [high-level-design.md](./high-level-design.md) | Service boundaries, the client-captures/server-reconciles design, key trade-offs (nightly aggregation, why geofence checks run client-side), data ownership, and failure degradation. |
| [low-level-design.md](./low-level-design.md) | Postgres schema, REST endpoints (`/api/v1/time/...`), offline clock-in reconciliation and timesheet-approval sequences, aggregation and geofence-validation pseudocode, concurrency control. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component/container view, the clock-in-with-geofence flow, correction-request-to-unblock sequence, ER model, and the timesheet lifecycle state machine. |
