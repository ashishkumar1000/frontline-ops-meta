# Module Catalog — All 25 Modules

Every module folder in this directory follows the same shape: a plain-language `README.md` (start
there), followed by the four build-ready deep-dive documents — `functional-spec.md`,
`high-level-design.md`, `low-level-design.md`, and `architecture-diagram.md`. The catalog below
gives a one-line description of each module and links to its README.

**Core modules (01–13)** come from the original v2.0 baseline. **Competitive-gap modules (14–25)**
were added after a gap analysis against Wooqer, Zenput, SafetyCulture, YOOBIC, Connecteam,
MaintainX, and Operandio (see the [addendum](../_context/new-modules-addendum.md)). Gap modules do
not build parallel engines — they reuse the platform primitives (`WorkItem`, `Evidence`, `Issue`,
`TemplateVersion`, the shared approval routing) defined by the core.

## Core modules (01–13)

| Module | What it does | README |
|---|---|---|
| **01 — Identity & Organization** | The foundation module: holds each customer company (tenant), its org tree of regions and stores, its users, and the answer to "who can see and do what, where" — single sign-on, automatic provisioning, and effective-dated org structures that keep historical reports correct after reorganizations. | [README](./01-identity-organization/README.md) |
| **02 — Work Orchestration** | The platform's engine room: turns every operating standard into a WorkItem with an owner, due date, and one shared lifecycle — recurring and event-triggered scheduling, individual/role/queue assignment, multi-level approvals, and automatic SLA escalations when work runs late. | [README](./02-work-orchestration/README.md) |
| **03 — No-Code Forms & Workflow Builder** | Lets operations admins build logic-rich digital checklists, audits, and surveys without code ("if the freezer reads above 8°C, require a corrective action"), publish them as immutable hash-verified versions with an impact preview, and have frontline workers fill them online or offline. | [README](./03-forms-workflow-builder/README.md) |
| **04 — Frontline Mobile & Offline** | The mobile app most frontline workers live in — a camera-first, one-handed Today/Next/Manager experience backed by a local database that works fully offline for 14+ days and syncs safely later through an idempotent, conflict-resolving protocol. | [README](./04-mobile-offline/README.md) |
| **05 — Evidence Management** | The proof vault: captures, hash-protects, and links every piece of evidence that work happened (photos, videos, signatures, GPS, scans, AI results) with chain-of-custody records, retention policies, legal holds, and privacy redaction. | [README](./05-evidence-management/README.md) |
| **06 — Issues & CAPA** | When something goes wrong at a site — a failed temperature check, a missed audit item — turns it into a tracked issue with an owner and deadlines, walks the fix through root cause analysis, corrective action, and a verification re-check, and escalates automatically if anything stalls. | [README](./06-issues-capa/README.md) |
| **07 — Communications & Knowledge** | Gets the right information to the right frontline people and proves it arrived — announcements with mandatory, tracked acknowledgement over push/email/SMS/chat in every language, plus a permission-aware SOP library that both people and the AI copilot search with citations. | [README](./07-communications-knowledge/README.md) |
| **08 — Learning** | When the same check keeps failing, automatically assigns targeted microlearning to the person involved, requires them to pass a knowledge-check quiz (works fully offline), then re-inspects the original check to measure whether the failure actually stopped — and manages expiring certifications that can block unsafe work. | [README](./08-learning/README.md) |
| **09 — Visual AI** | Turns photos frontline workers already capture — shelves, floors, safety gear, signage — into scored compliance checks: confident AI findings can pass automatically, anything uncertain goes to a manager review queue with violations drawn on the image, and stores can appeal. | [README](./09-visual-ai/README.md) |
| **10 — Agentic Operations (Agent Runtime)** | The platform's governed AI layer: nine purpose-built agents watch operational events and either act automatically or ask a manager for approval through one auditable, policy-enforced pipeline. | [README](./10-agentic-operations/README.md) |
| **11 — Analytics & Control Tower** | The dashboards-and-predictions layer: turns tasks, audits, and issues into KPIs, alerts, and location risk forecasts, letting executives drill from a company-wide number down to the store and evidence behind it. | [README](./11-analytics-control-tower/README.md) |
| **12 — Integrations & Extensibility** | The platform's connection point to the rest of the company's software — HR, ERP, data warehouses, BI tools, and chat — via a connector catalog, signed webhooks, an API/SDK, and batch file transfers. | [README](./12-integrations-extensibility/README.md) |
| **13 — Security, Privacy & Compliance** | The security rulebook and control room: keeps every customer's data fully separated and encrypted, records an untamperable audit trail, governs how AI touches data, and handles privacy requests and incidents. | [README](./13-security-compliance/README.md) |

## Competitive-gap modules (14–25)

| Tier | Module | What it does | README |
|---|---|---|---|
| 1 | **14 — Workforce Scheduling & Labor Management** | Builds and publishes the weekly roster of who works which shift where, and lets frontline staff set availability, request time off, and swap or pick up shifts with policy-checked manager approval. | [README](./14-workforce-scheduling/README.md) |
| 1 | **15 — Time & Attendance** | Lets associates clock in and out — breaks included — from the same mobile app, with geofence and anti-buddy-punching checks, turning punches into manager-approved, payroll-ready timesheets. | [README](./15-time-attendance/README.md) |
| 1 | **16 — Asset & Equipment Maintenance (CMMS)** | Tracks every piece of equipment, its condition, and its spare parts, scheduling preventive maintenance and dispatching repair work orders to technicians or vendors before and when things break. | [README](./16-asset-maintenance-cmms/README.md) |
| 2 | **17 — IoT Sensor Platform** | Streams readings from physical sensors (temperature, humidity, door, vibration) into the platform and turns rule breaches — like a cooler running warm too long — into Issues with the sensor data attached as tamper-proof evidence. | [README](./17-iot-sensor-platform/README.md) |
| 2 | **18 — Employee Engagement & Voice of Frontline** | Gives frontline workers pulse surveys they can answer anonymously, public peer and manager recognition, and a location-level engagement score for leadership dashboards. | [README](./18-employee-engagement/README.md) |
| 2 | **19 — Inventory, Procurement & Supplier Management** | Gives frontline operations a governed record of stock and spending — barcode-assisted stock counts with variance flagging, cost-threshold-routed purchase approvals, delivery receiving with price-variance checks, and supplier scorecards. | [README](./19-inventory-procurement/README.md) |
| 2 | **20 — EHS — Safety Incident & Risk Register** | A regulatory safety layer: records workplace injuries, near-misses, and hazards, classifies them against jurisdiction rules (e.g. OSHA-style recordability), tracks mandatory reporting deadlines with escalating alerts, and retains records as long as the law requires. | [README](./20-ehs-safety-risk/README.md) |
| 3 | **21 — Food Safety & HACCP** | Packages the platform's existing forms, sensors, and issue tracking into a HACCP food-safety program: critical control points (like cooler temperatures) are checked manually or watched by IoT sensors, and any breach forces a corrective action such as "discard product" before the check can close. | [README](./21-food-safety-haccp/README.md) |
| 3 | **22 — Gamification & Recognition** | An optional motivational layer that turns work other modules already verify into badges, an append-only points ledger, opt-in-safe leaderboards, and learning-pathway progress — deliberately removable without touching any core operation. | [README](./22-gamification/README.md) |
| 3 | **23 — New-Location Launch Management** | Turns opening, remodeling, or relaunching a physical location from a spreadsheet exercise into a templated project whose checklist items become ordinary tracked work — phase progress, days-to-launch countdown, at-risk milestone flags, and a portfolio view across concurrent openings. | [README](./23-store-launch-management/README.md) |
| 3 | **24 — App Marketplace & Template Catalog** | A catalog of pre-built "apps" — bundles of form templates, workflows, and connector configs — that tenants browse and one-click install as their own editable copies, with a governed publisher-review pipeline and opt-in updates that never overwrite tenant customizations. | [README](./24-app-marketplace/README.md) |
| 3 | **25 — Field Service & Third-Party Vendor Scheduling** | Manages work done by non-employees: a directory of vendors and their technicians, two-way appointment scheduling, a locked-down portal where technicians see only their own assignments, and SLA scorecards that feed future vendor selection. | [README](./25-field-service-vendor-scheduling/README.md) |

## How the modules fit together

A few patterns recur across every module — knowing them up front makes the individual READMEs easier to read:

- **Everything executable is a WorkItem.** Forms, audits, maintenance orders, launch tasks, stock
  counts — all ride module 02's one canonical WorkItem lifecycle. Gap modules own domain vocabulary;
  Work Orchestration owns execution. (Deliberate, documented exceptions exist, e.g. the EHS
  `SafetyIncident` regulatory lifecycle layered beside the WorkItem one.)
- **Identity (01) gates everything.** Every module is a downstream consumer of tenants, org trees,
  users, and scoping; module 01 carries the platform's highest availability target for that reason.
- **Evidence (05) is never mutated.** Media is hash-protected with chain of custody; AI verdicts and
  redactions become *new linked records*, never overwrites of the original.
- **One approval mechanism.** Purchase orders, vendor work, and form publishing all reuse Work
  Orchestration's multi-level `ApprovalRoute` — there is deliberately no second approval engine.
- **Two schedulers, one rule.** Workforce rostering (14) and checklist scheduling (02) are separate
  services that integrate only via the `shift.published` / `shift.cancelled` event contract.
- **Policy before AI.** Agents (10) and Visual AI (09) act through the same governed APIs a human
  would use; low-confidence or high-impact actions always hold for human approval, platform-wide.
- **Server is the source of truth.** Client clocks are never trusted; offline writes are keyed on
  client-generated UUIDs and sync idempotently.
- **Payroll is an outbound connector.** No module owns a payroll engine; timesheets leave via
  Integration Hub (12).

## Related indexes

- [`docs/README.md`](../README.md) — the file-by-file index of every document in the package
- [`docs/_context/README.md`](../_context/README.md) — the shared reference layer (conventions,
  binding engineering decisions, source baseline, modules addendum)
- [`02-Epics-and-Stories.md`](../02-Epics-and-Stories.md) — the engineering backlog mapping epics to
  modules (note: epic numbers do not align 1:1 with folder numbers)
