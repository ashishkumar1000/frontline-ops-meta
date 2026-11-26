# Module 16 — Asset & Equipment Maintenance (CMMS)

> A registry of every piece of equipment at every location — coolers, HVAC units, delivery vans,
> POS terminals — plus the program that keeps it running: scheduled preventive maintenance, fast
> reporting when something breaks, work orders assigned to technicians, and spare-parts tracking.
> CMMS stands for Computerized Maintenance Management System, the product category this module
> closes against MaintainX-class competitors; it exists so equipment gets fixed before it fails and
> never fails twice without a root-cause investigation.

| | |
|---|---|
| **Module code** | **CMMS** (FR-CMMS-###) |
| **Owning service** | Asset & Maintenance Service (Asset Registry + PM Scheduler + Parts Inventory) |
| **Tier** | 1 (competitive gap closure — `new-modules-addendum.md` row 16) |

## In plain terms
- Every asset is registered with its type, location, serial number, warranty, criticality, and parent-child relationships (e.g., a compressor inside a walk-in cooler), so anyone can answer "what equipment do we have, where, and in what condition?"
- Preventive maintenance (PM) schedules fire on a calendar ("every 90 days"), on usage ("every 500 operating hours"), or on whichever comes first — mirroring real OEM maintenance contracts.
- Due schedules and breakdown reports automatically create maintenance work orders in the same work system associates already use for checklists and audits, matched to technicians by skill and certification.
- Anyone can report a breakdown; a critical report can immediately take the asset out of service, and duplicate reports for the same asset are caught with a friendly "someone reported this 20 minutes ago" prompt.
- Technicians record parts used, meter readings, and before/after photos in the app; parts stock decrements automatically, and low stock raises a reorder signal to the procurement module.
- Repeated breakdowns (3 ad-hoc repairs in 30 days by default) automatically open an Issue for root-cause analysis, and downtime per asset feeds "preventive intervention rate" dashboards for regional managers.

## Key concepts
| Concept | What it means |
|---|---|
| `Asset` | One piece of equipment: type, location, serial, warranty, criticality, and an operational status (in service / out of service / under maintenance / retired). Promoted from a placeholder in the original spec to a full entity by this module. |
| `AssetType` | The taxonomy (e.g., "Walk-in Cooler," "HVAC Rooftop Unit") with type-specific attribute schemas and default required technician skills. |
| `MaintenanceSchedule` | A preventive-maintenance rule targeting one asset or a whole asset type at a set of locations, on a calendar basis, a meter basis, or both. |
| `MaintenanceWorkOrder` | The actual repair/inspection job — a `WorkItem` of type `MAINTENANCE_WORK_ORDER` that reuses Work Orchestration's canonical state machine verbatim; this module never invents a second state machine. |
| `PartInventoryItem` / `PartUsage` | Per-location spare-parts stock and the record of parts consumed on a work order, which decrements stock atomically. |
| `Technician` / `Vendor` | Internal maintenance staff (with skills/certifications) or external companies; a work order can be assigned to either. |
| Meter reading | A usage value (operating hours, cycles, mileage) captured with each service, used to compute when meter-based maintenance is next due. |

## How it connects to the rest
- **Work Orchestration (Module 02)** — the execution engine. This module creates `MAINTENANCE_WORK_ORDER` WorkItems via Work Orchestration's API and listens for `task.completed`/`task.overdue` events to flip asset status and badges; it never writes `work_items.status` directly. The docs include an explicit non-duplication rule (functional-spec §9) to keep one state machine for all work.
- **Evidence (Module 05)** — meter-reading photos and before/after repair photos are stored as `Evidence` records referenced by ID; only the meter *value* lives in this module's own tables.
- **Issues & CAPA (Module 06)** — recurring-failure patterns auto-create an `Issue` (category EQUIPMENT, source RULE_TRIGGERED) via `create_issue`; root-cause and corrective-action workflows stay in Module 06.
- **Module 17 (IoT Sensor Platform)** — a sensor bound to an `Asset` can supply the "latest meter value" that drives meter-based PM and anomaly scores; this module never ingests raw high-volume sensor streams.
- **Module 19 (Inventory & Procurement), Module 25 (Field Service), Module 12 (Integrations)** — low stock auto-creates a purchase requisition via Module 19; a work order assigned `assignee_type=VENDOR` hands off external dispatch to Module 25; ERP parts/vendor master data syncs through the Integration Hub.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-CMMS-001–061: asset registry, PM schedule authoring, automatic work-order creation, downtime tracking, parts inventory, technician matching, plus the WorkItem non-duplication rule and resolved decisions. |
| [high-level-design.md](./high-level-design.md) | The Asset Registry + PM Scheduler + Parts Inventory service boundaries, why PM scheduling reuses Work Orchestration's Temporal recurrence pattern, data ownership, and failure degradation. |
| [low-level-design.md](./low-level-design.md) | Postgres schema, REST endpoints, sequence diagrams (PM materialization, breakdown-to-repair), meter-based due-date estimation pseudocode, and parts-stock concurrency control. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view, PM-materialization and breakdown-to-repair sequences, the owned-entities ER model, and the asset operational-status state machine. |
