# New Modules Addendum — Competitive Gap Closure (Modules 14–25)

These 12 modules are **not** in the original v2.0 PRD/FSD baseline (`source-spec.md`). They were
identified from a competitive gap analysis against Wooqer, Zenput, SafetyCulture, YOOBIC,
Connecteam, MaintainX and Operandio, and are being added as first-class modules with the same
build-ready depth as modules 01–13. Every module doc you write MUST still read
`_context/source-spec.md` and `_context/conventions.md` first and stay fully consistent with them
(tech stack, entity-naming style, API/event conventions, diagram conventions, multi-tenancy rule) —
this addendum only adds the NEW scope and tells you how it plugs into the 13 existing modules.
Do not invent a different tech stack or a different API convention style than conventions.md.

Reuse the existing platform primitives wherever a new module's need overlaps one already defined:
`WorkItem`/`WorkItemType` (Work Orchestration) for anything schedulable/assignable/completable,
`Evidence` for any captured photo/reading, `Issue` (Issues & CAPA) for anything that becomes a
managed exception, `OrganizationNode`/`UserScope` (Identity & Organization) for all scoping,
`TemplateVersion` (Forms & Workflow Builder) for any versioned configuration schema, and the
canonical event envelope / `/api/v1/` conventions for everything. A new module should feel like a
natural extension of the same platform, not a bolted-on product.

## Module Map

| # | Slug | Name | Tier | Primary new entities | Key integrations |
|---|---|---|---|---|---|
| 14 | `14-workforce-scheduling` | Workforce Scheduling & Labor Management | 1 | `Shift`, `ShiftTemplate`, `AvailabilityWindow`, `ShiftAssignment`, `SwapRequest`, `LaborForecast` | Identity/Org (role/location eligibility), Work Orchestration (shift-based WorkItem schedules already reference "shift instances" — this module is now the authoritative source of those), Time & Attendance (consumes shift plan), Analytics (labor cost/coverage KPIs) |
| 15 | `15-time-attendance` | Time & Attendance | 1 | `TimeClockEvent`, `Timesheet`, `TimesheetLine`, `GeofenceZone`, `BreakPolicy` | Workforce Scheduling (compares actual punches to planned shifts), Mobile & Offline (clock-in/out must work offline, geofenced), Identity/Org (employee record), Integrations (payroll export) |
| 16 | `16-asset-maintenance-cmms` | Asset & Equipment Maintenance (CMMS) | 1 | `Asset` (promote from placeholder to full entity), `AssetType`, `MaintenanceSchedule`, `MaintenanceWorkOrder` (a `WorkItem` type, not a parallel state machine), `PartInventoryItem`, `PartUsage`, `Vendor`/`Technician` | Work Orchestration (`MAINTENANCE_WORK_ORDER` new WorkItem type reusing the canonical state machine), Evidence (before/after photos, meter readings), Issues & CAPA (asset failure can auto-create an Issue), Integrations (ERP parts/vendor sync) |
| 17 | `17-iot-sensor-platform` | IoT Sensor Platform | 2 | `Sensor`, `SensorReading` (high-volume time-series), `SensorAlertRule`, `SensorDeviceBinding` | Evidence (a sensor reading can be attached as evidence), Issues & CAPA (breached alert rule auto-creates an Issue, same pattern as Appendix B's rule spec), Asset & Equipment Maintenance (sensor bound to an Asset feeds predictive maintenance), Analytics (time-series feed) |
| 18 | `18-employee-engagement` | Employee Engagement & Voice of Frontline | 2 | `PulseSurvey`, `SurveyQuestion`, `SurveyResponse`, `RecognitionEvent`, `EngagementScore` | Communications & Knowledge (shares audience-targeting/delivery infra), Work Orchestration (`Survey` WorkItem type already exists — this module is its functional owner), Analytics (eNPS/engagement KPIs), Identity/Org (anonymity/aggregation rules tied to org scope) |
| 19 | `19-inventory-procurement` | Inventory, Procurement & Supplier Management | 2 | `SKU`, `StockCount`, `StockCountLine`, `PurchaseOrder`, `PurchaseOrderLine`, `Supplier`, `SupplierScorecard` | Forms & Workflow Builder (stock-count is a template-backed WorkItem), Work Orchestration (`STOCK_COUNT`/`PURCHASE_APPROVAL` WorkItem types — the latter reuses the multi-level Approval Routing built for Work Orchestration), Integrations (ERP/POS sync), Analytics (shrinkage/stock-out KPIs) |
| 20 | `20-ehs-safety-risk` | EHS — Safety Incident & Risk Register | 2 | `SafetyIncident`, `RiskAssessment`, `HazardRegisterEntry`, `SafetyObservation`, `RegulatoryReport` | Issues & CAPA (a SafetyIncident can spawn an Issue for corrective action, but keeps its own regulatory-retention record distinct from generic Issues), Evidence, Learning (safety training linkage), Analytics (TRIR/LTIR-style safety KPIs) |
| 21 | `21-food-safety-haccp` | Food Safety & HACCP | 3 | `HACCPPlan`, `CriticalControlPoint`, `TemperatureLog`, `FoodSafetyCorrectiveAction` | IoT Sensor Platform (temperature sensors), Forms & Workflow Builder (CCP check is a template-backed WorkItem), Issues & CAPA (CCP breach auto-creates an Issue), Evidence |
| 22 | `22-gamification` | Gamification & Recognition | 3 | `Badge`, `Achievement`, `PointsLedger`, `Leaderboard`, `Pathway` | Learning (badges/pathways layer on `Course`/`Certification`), Employee Engagement (recognition events can award points), Analytics (leaderboard data), Work Orchestration (completion streaks) |
| 23 | `23-store-launch-management` | New-Location Launch Management | 3 | `LaunchProject`, `LaunchTemplate`, `LaunchMilestone`, `LaunchTask` | Work Orchestration (a launch is a structured `Campaign` of dependent WorkItems — this module is a configuration/orchestration layer on top of Work Orchestration, not a new execution engine), Identity/Org (new `OrganizationNode` provisioning), Integrations (new-store data feeds) |
| 24 | `24-app-marketplace` | App Marketplace & Template Catalog | 3 | `AppListing`, `AppTemplate`, `AppInstallation`, `Publisher` | Forms & Workflow Builder (an "app" is a curated bundle of `TemplateVersion`s + a Workflow), Integrations & Extensibility (a marketplace app may bundle a connector config), Identity/Org (per-tenant install scope) |
| 25 | `25-field-service-vendor-scheduling` | Field Service & Third-Party Vendor Scheduling | 3 | `ServiceRequest`, `VendorTechnician`, `ServiceAppointment`, `VendorSLA` | Asset & Equipment Maintenance (a `MaintenanceWorkOrder` can be dispatched to an external vendor instead of an internal technician), Integrations (vendor-facing portal/API), Work Orchestration (appointment is a WorkItem assigned to an external `assignee_type=VENDOR`, a new assignee type this module introduces) |

## Numbering & file convention (unchanged from modules 01–13)

Each module gets `docs/modules/<slug>/functional-spec.md`, `high-level-design.md`,
`low-level-design.md`, `architecture-diagram.md`, at the same depth and using the same section
structure as modules 01–13 (see any existing module for the template, e.g.
`docs/modules/06-issues-capa/`). Use `FR-<SHORTCODE>-###` numbered requirements
(shortcodes: WFS=Workforce Scheduling, TA=Time & Attendance, CMMS=Asset Maintenance, IOT=IoT
Platform, ENG=Employee Engagement, INV=Inventory/Procurement, EHS=Safety/Risk, HACCP=Food Safety,
GAM=Gamification, LAUNCH=Store Launch, MKT=App Marketplace, FSVC=Field Service).

## New platform-wide conventions these modules introduce (keep consistent across all 12)

- **New `WorkItem.assignee_type` value: `VENDOR`** (external, non-employee actor) — introduced by
  Field Service, but referenced by Asset Maintenance too. Document it as an extension of the
  existing `assignee_type` enum, not a parallel field.
- **New `WorkItem` types**: `MAINTENANCE_WORK_ORDER`, `STOCK_COUNT`, `PURCHASE_APPROVAL`,
  `SAFETY_OBSERVATION`, `LAUNCH_TASK`, `SERVICE_APPOINTMENT` — all reuse the canonical WorkItem
  state machine from `conventions.md`; none of these modules invents a second state machine for
  "work," only for their own module-specific supporting entities (e.g. `SafetyIncident` has its own
  lifecycle distinct from `WorkItem`/`Issue`, because regulatory incidents have reporting deadlines
  and retention rules generic Issues don't).
- **Sensor/time-series data (Module 17)** should be modeled as a high-volume, time-partitioned table
  (or a dedicated time-series store if you justify it in the HLD, e.g. TimescaleDB extension on the
  existing Postgres fleet) — do not model millions of readings/day as ordinary Postgres rows without
  partitioning.
- **Payroll/HR integration (Module 15)** is always via the Integration Hub (module 12) as an
  outbound connector — Time & Attendance owns the timesheet data, it does not own a payroll engine.
