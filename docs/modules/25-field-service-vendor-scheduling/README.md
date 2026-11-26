# Module 25 — Field Service & Third-Party Vendor Scheduling

> Much frontline work is done by people who are not employees: HVAC contractors, pest control,
> elevator inspectors, appliance repair technicians. Today that coordination is usually an
> unmanaged email-and-phone exercise with no visibility into who was promised what or how the
> vendor actually performs. This module is the system of record for the tenant's external vendors
> and their individual technicians, for scheduling appointments with them (both sides can propose
> time windows), and for tracking their performance against contracted SLAs — including a secure,
> locked-down portal so a vendor technician can see their own appointments and nothing else.
> Store/site managers request and sign off on visits; operations admins manage the vendor
> directory; regional managers compare vendor scorecards.

| | |
|---|---|
| **Module code** | FSVC (`FR-FSVC-###`) |
| **Owning service** | Field Service Service |
| **Tier** | 3 |

## In plain terms
- Keep a directory of vendors and their technicians: which trades they cover (HVAC, pest control,
  plumbing, …), which locations they service, their contract SLA tier, and whether they get
  portal access.
- Raise a service request either standalone (a scheduled pest-control visit, an elevator
  inspection) or as a handoff when an equipment repair (module 16) needs a specialist no internal
  technician can handle.
- Match a vendor automatically — by trade, coverage area, SLA tier, and their own on-time-arrival
  history — or pick one manually from a filtered list.
- Negotiate the appointment in either direction: the vendor proposes windows and the manager
  approves one, or the manager proposes and the vendor confirms or counters. Vendors mark
  en-route/arrived so the on-site contact sees them coming.
- Hold vendors accountable: response-time, on-time-arrival, and first-time-fix metrics roll up
  into scorecards per vendor, and a persistent underperformer can be escalated and raised as a
  managed issue.
- Work with vendors who will never log in: an email/SMS fallback coordinates scheduling for them.

## Key concepts
| Concept | What it means |
|---|---|
| `Vendor` / `VendorTechnician` | The company-level vendor record and the individual person who actually shows up, with their own certifications. |
| `ServiceRequest` | One ask for external work, with its own lifecycle (`REQUESTED → VENDOR_MATCHED → APPOINTMENT_PROPOSED → CONFIRMED → … → SIGNED_OFF`) wrapped around the WorkItem. |
| `ServiceAppointment` | The confirmed visit itself — a `SERVICE_APPOINTMENT` WorkItem assigned to `assignee_type = VENDOR`, the new external assignee type this module introduces. |
| `VendorSLA` | The contracted targets: response time, on-time arrival grace window, first-time-fix rate — reused through Work Orchestration's escalation policies when breached. |
| `VendorScorecard` | The rolling performance rollup (response time p50/p90, on-time arrival rate, first-time-fix rate) that also feeds auto-match ranking. |
| `ASSIGNED_WORK_ITEM_ONLY` scope | The restricted access kind for vendor technicians: they see only the WorkItems assigned to them — nothing else at the location, no org-tree access. |

## How it connects to the rest
- **Work Orchestration (module 02 / EPIC-03)** — owns the `SERVICE_APPOINTMENT` WorkItem's state
  machine, the `VENDOR` assignee type this module introduces as an enum extension (not a parallel
  field), and the escalation machinery reused for SLA breaches. This module never writes
  `work_items.status` directly; it drives its own `ServiceRequest` lifecycle around the WorkItem.
- **Asset & Equipment Maintenance (module 16 / CMMS)** — a `MaintenanceWorkOrder` can be handed
  off here for external dispatch instead of an internal technician; the vendor's part usage and
  asset data stay in module 16.
- **Identity & Organization (module 01 / EPIC-01/02)** — vendor technicians are provisioned as
  real platform users (not a parallel identity system) carrying the new restricted
  `ASSIGNED_WORK_ITEM_ONLY` scope kind; authentication is magic-link/OTP in v1, not tenant SSO.
- **Issues & CAPA (module 06)** and **Evidence (module 05)** — a persistently underperforming
  vendor becomes a managed `Issue`; sign-off photos, signatures, and completion photos are
  Evidence records referenced by ID.
- **Integrations (module 13 / EPIC-13)** and **Analytics (module 11 / EPIC-12/15)** — vendors
  running their own dispatch systems integrate via the Integration Hub's webhook/API pattern;
  vendor performance reporting is consumed by the Control Tower.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-FSVC-001–074 covering the vendor directory, service requests, assignment/matching, appointment scheduling, the restricted vendor portal, sign-off, and SLA scorecards. |
| [high-level-design.md](./high-level-design.md) | Service responsibilities, the split internal/vendor-facing gateway, and the module's most security-sensitive section: external-actor isolation for vendor technicians. |
| [low-level-design.md](./low-level-design.md) | Schema and ER diagram, internal and vendor-portal REST endpoints, sequence diagrams including the scoped-access check, and auto-match/scoping pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Component diagram with the separate vendor gateway, service-request-to-appointment and vendor scoped-access-check sequence diagrams, ER diagram, and the ServiceRequest lifecycle state machine. |
