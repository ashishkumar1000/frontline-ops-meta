# Module 21 — Food Safety & HACCP

> This module turns the platform's existing form, sensor, and issue machinery into a
> food-safety compliance program for restaurant/QSR and food-retail tenants. It is built around
> HACCP (Hazard Analysis Critical Control Points — the seven-principle food-safety method used by
> the FDA Food Code and Codex Alimentarius): a QA admin defines the plan and its critical control
> points (CCPs), associates take temperature checks, sensors watch coolers continuously, and any
> out-of-range reading forces a food-safety corrective action like "discard product" before the
> check can be closed. It is used daily by frontline associates and store managers, and
> administered by QA / Food Safety admins.

| | |
|---|---|
| **Module code** | HACCP (`FR-HACCP-###`) |
| **Owning service** | Food Safety & HACCP Service (explicitly a thin configuration/orchestration layer) |
| **Tier** | 3 |

## In plain terms
- Author a HACCP plan the way a QA team already thinks about it: process flow (Receiving → Cold
  Storage → Prep → Cooking → …), hazard analysis per step, then the derived critical control
  points with limits like "cold holding ≤ 41°F."
- Monitor each control point either manually (a scheduled temperature check on the associate's
  Today list) or automatically (an IoT sensor watching the cooler) — the admin picks one binding
  per CCP and the module generates the form or the sensor rule.
- Force corrective action on a breach: an out-of-range reading blocks submission until a
  food-safety action (discard product, reject delivery, re-cook to temperature, recalibrate
  equipment) is recorded — with a photo required for the high-stakes discard/reject actions.
- Sign off at shift level: managers review and sign off a location's monitoring records, and
  non-conforming product decisions (hold/release) live in the same flow.
- Export audit-ready records: a regulator- or auditor-facing export of the plan version, CCP
  monitoring history, breaches, and corrective actions for any date range.

## Key concepts
| Concept | What it means |
|---|---|
| `HACCPPlan` | The versioned food-safety program document (process flow, hazard analysis, CCP list, limits, corrective actions, verification) — versioned like a `TemplateVersion`, immutable once published. |
| `CriticalControlPoint` (CCP) | One process step where a critical limit must be monitored (e.g. cooler temperature), with a manual form binding or an automated sensor binding. |
| Critical limit | The structured threshold a CCP must stay within (temperature, time, pH, water activity). |
| `TemperatureLog` | The combined manual + automated monitoring record — modeled as a read-side view (`ccp_monitoring_records`) over Forms submissions and IoT readings, never a duplicated table. |
| `FoodSafetyCorrectiveAction` | The food-safety-worded action taken on a breach (e.g. `DISCARD_PRODUCT`); it satisfies the underlying `Issue`'s CAPA step rather than running a parallel workflow. |
| `HACCPShiftReview` | The manager's shift-level sign-off over a location's monitoring records. |

## How it connects to the rest
- **IoT Sensor Platform (module 17)** — automated CCPs are `SensorAlertRule`s authored through
  module 17's API; readings and breach events flow from 17, and a sensor going offline triggers
  this module's fallback (spawn a manual check). No sensor hardware is required to use the module.
- **Forms & Workflow Builder (module 03)** and **Work Orchestration (module 02)** — manual CCP
  checks are template-backed WorkItems scheduled through Work Orchestration's existing
  scheduling primitives; this module authors the template content but never renders or stores
  submissions.
- **Issues & CAPA (module 06)** — every CCP breach (manual or automated) creates its `Issue`
  through the same platform-wide path; the `FoodSafetyCorrectiveAction` picker is a front end over
  the Issue's `CORRECTIVE_ACTION` CAPA step, not a second exception engine.
- **Evidence (module 05)** — discard/reject photos are Evidence records attached via module 05's
  API.
- **Module 20 (EHS) and Module 19 (Inventory)** — a food-safety event that is also a workplace
  injury (e.g. a burn during discard) is cross-linked to EHS, not merged; cold-chain receiving is
  a CCP here, while the full PO receiving workflow stays in module 19.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-HACCP-001–062 covering plan authoring, CCP definition, manual and automated monitoring, breach handling, verification, and audit export. |
| [high-level-design.md](./high-level-design.md) | The "configuration layer, not a new execution engine" stance, the ownership table, and the rationale for making `TemperatureLog` a read model rather than a duplicated table. |
| [low-level-design.md](./low-level-design.md) | Schema (plans, CCPs, monitoring-records read model, corrective actions, shift reviews), REST endpoints, sequence diagrams, and compliance-rate/fallback pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Component diagram with the manual and automated paths converging on one Issues & CAPA surface, golden-path and manual-check sequence diagrams, and the CCP monitoring-record state machine. |
