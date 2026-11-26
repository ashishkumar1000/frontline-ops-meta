# Frontline Operations & Agentic Ops Platform — Product Requirements Document (PRD)

Version 3.0 | Status: Build-Ready Master PRD | Supersedes PRD sections 1-15 and Appendices A-E of `source-spec.md` v2.0

This document is the master, product-level PRD for the platform. It sharpens and deepens the vision,
personas, principles, and capability scope defined in `source-spec.md`, and adds the mobile-first
strategy, risks, and non-goals that the baseline left implicit. It does not restate module-level
functional requirements — those live in `/modules/<slug>/functional-spec.md` for each of the 13
modules listed in Section 7. Every entity name, state machine, API convention, and tech-stack
decision referenced here is defined authoritatively in `conventions.md` and must not be
contradicted.

---

## 1. Executive Summary & Problem Statement

Enterprises that operate through many physical locations — retail chains, QSR and hospitality
groups, facilities and logistics networks, manufacturing floors, healthcare-adjacent site
operations — run on a structural mismatch: the standards, brand promises, safety rules, and
compliance obligations are defined centrally, but the work that fulfils them happens at the edge, on
a phone or a clipboard, often without connectivity, and largely invisible to the people who are
accountable for it. The result is a chronic gap between **what leadership believes is happening**
and **what is actually happening at the site level**, discovered only after a customer complaint, a
health inspection, an audit finding, or a safety incident.

Today that gap is patched with a fragmented stack: a checklist/inspection tool, a separate LMS, a
separate messaging tool, a spreadsheet-based issue tracker, and a BI dashboard stitched together by
a data team weeks after the fact. Each tool captures a slice of the truth; none of them close the
loop from **standard → work → evidence → exception → correction → re-verification → learning**. Newer
entrants bolt a chatbot or an image classifier onto a checklist product and market it as "AI," but
without a governed connection to organizational scope, work state, and historical evidence, an AI
recommendation is a guess dressed as insight.

**Why now.** Three forces make this the right moment to build a unified platform rather than another
point tool: (1) frontline labor markets are tight and turnover is structurally high, so the
per-location cost of manual chasing, spreadsheet consolidation, and undocumented tribal knowledge has
become unaffordable at scale; (2) foundation models have made contextual reasoning, natural-language
workflow authoring, and visual compliance checking commercially viable at frontline price points for
the first time, but only if governed by deterministic policy — ungoverned AI in a regulated,
multi-tenant, safety-relevant domain is a liability, not a feature; (3) mobile hardware, on-device
storage, and offline-first sync patterns (CRDT-adjacent conflict resolution, resumable media upload)
have matured enough that "offline is a first-class operating mode" is now an engineering-tractable
promise rather than a marketing claim.

**Our bet: mobile-first, explicitly.** We are not building a web product with a companion app. We are
building a frontline mobile application as the primary product surface for the majority of the user
base (Frontline Associates, Auditors, most Store/Site Managers), with the web Control Tower serving
the administrative and analytical minority. This is a deliberate, load-bearing strategic bet
detailed in Section 5 — it shapes API design, sync architecture, evidence capture, and even how we
staff and sequence engineering work, not just the UI layer.

**Competitive framing.** Incumbents fall into three camps, each incomplete: (a) *checklist/inspection
tools* (strong at form capture, weak at exception management, weak or absent AI governance); (b)
*task/communication tools* (strong at messaging and task assignment, weak at evidence and audit
rigor); (c) *point-solution AI vendors* (a single computer-vision or LLM feature with no operational
system of record underneath it). None of the three own the **Operational Graph** — the linked record
of organization, work, evidence, exceptions, knowledge, and historical interventions that makes an AI
agent's recommendation trustworthy and an auditor's evidence defensible. That graph, not any single
feature, is the durable moat this platform is built to own.

### 1.1 Product Outcomes (carried forward and sharpened from source-spec.md §1.1)

- Replace manual chasing, spreadsheet consolidation, and fragmented exception handling with a single
  governed system of record for frontline work.
- Increase **verified** frontline execution — a claim of "done" is not the product's unit of success;
  a claim backed by evidence and, where required, approval is.
- Reduce repeat failures by structurally connecting observation → root cause → corrective action →
  training → re-verification, rather than leaving that connection to institutional memory.
- Give every management layer, from site manager to executive, a control tower that explains what is
  wrong, where, how severe, why, what has already been done, and what needs a human now.
- Build a safe, incremental path from deterministic rules-based automation to policy-governed
  autonomous agents, with the policy and audit substrate built before autonomy is exposed.
- Preserve an auditable historical record: what happened, under which version of the standard, with
  what evidence, and who (human or agent) acted — durable through reorganizations, template changes,
  and personnel turnover.

### 1.2 Primary Market Shape (unchanged targets, carried from source-spec.md §1.2)

| Customer characteristic | Target state |
|---|---|
| Locations | 50 to 100,000+ operational sites |
| Frontline users | 500 to 1,000,000+ |
| Management layers | 3 to 8 |
| Recurring workflows | 10+ per location |
| Operational evidence | Photos, forms, signatures, documents, sensor data |
| Connectivity | May be intermittent or unavailable at the point of work |
| Governance | Enterprise policies, approvals, SLAs, audit trail |
| AI maturity | From recommendations to controlled autonomous actions |

---

## 2. Product Vision, Strategic Positioning & Category Definition

**Vision.** Become the enterprise operating layer that turns a centrally authored standard into
localized, measurable, continuously improving frontline execution — the system that knows who owns
each location, what work should happen there and when, what evidence that work requires, what policy
governs it, what exceptions currently exist, how prior exceptions were resolved, and what
intervention should happen next, automatically where policy allows and with a human in the loop where
it does not.

**Strategic positioning.** We compete not on any single feature (forms, chat, an image classifier)
but on the completeness and trustworthiness of the closed loop:

```
PLAN -> COMMUNICATE -> ASSIGN -> EXECUTE -> CAPTURE EVIDENCE -> VERIFY ->
DETECT EXCEPTIONS -> CORRECT -> RE-VERIFY -> LEARN -> OPTIMIZE -> AUTOMATE
```

A competitor with a better checklist builder or a flashier chatbot does not threaten this position
unless it also owns organizational scope, evidence provenance, exception lifecycle, and agent
governance simultaneously — replicating the graph, not the feature, is the barrier to entry we are
building.

**Category definition.** *Enterprise Frontline Operations & Agentic Ops Platform*: a software
category spanning task orchestration, inspections and audits, communications, knowledge and
learning, evidence management, issue/CAPA management, computer-vision-assisted compliance, analytics
and control-tower reporting, and governed AI agent execution, unified by one authorization model, one
evidence model, and one audit trail.

**Product promise** (the compounding thesis that should discipline every roadmap decision):

- Every standard becomes work.
- Every completion can become evidence.
- Every deviation can become a managed exception.
- Every repeated exception can become an automated intervention.
- Every intervention can be measured for effectiveness.

Each step in that chain is optional at the tenant's discretion (a small tenant may never touch
autonomous agents), but the platform must make each step available without requiring the customer to
adopt a different vendor to get there. This is what distinguishes a platform from a feature suite:
the exit ramp from "recommendation" to "controlled autonomy" exists on day one of the architecture
even for tenants who will not use it for years.

---

## 3. Market Shape, Buyer/User Personas and Jobs-to-be-Done

The buying motion is enterprise, multi-stakeholder, and long-cycle: an Enterprise Admin or VP
Operations typically sponsors the purchase, Operations Admins and Regional Managers are the
day-to-day power users who determine renewal, and Frontline Associates and Auditors are the
highest-volume users whose adoption determines whether the data in the Control Tower is trustworthy
at all. A platform that is loved by admins but resented by associates fails at scale, because the
entire value proposition depends on frontline capture behavior. This is the single most important
go-to-market fact shaping product priority: **frontline adoption is not a downstream metric, it is
the leading indicator for every other metric in the system.**

### 3.1 Persona Profiles

#### Enterprise Admin
- **Goals:** Stand up and govern the platform for the whole enterprise — identity/SSO, organizational
  hierarchy, tenant-wide policy, integration connections, AI feature enablement and kill switches.
- **Pains:** Fragmented identity systems across brands/subsidiaries; slow, error-prone bulk
  provisioning; no single place to see "who can access what" before an audit forces the question.
- **Primary device:** Desktop/laptop web browser (Admin Console).
- **Connectivity reality:** Always-online, corporate network or VPN.
- **Success criteria:** New tenant/brand onboarded in days not months; zero cross-scope access
  incidents; SCIM-driven provisioning eliminates manual account lifecycle work; can prove access
  posture to a security or compliance reviewer on demand via the Effective Access preview.

#### Operations Admin
- **Goals:** Translate operating standards (brand, safety, quality, promotional) into deployable
  templates, workflows, audits, and campaigns; manage the template/workflow lifecycle; measure
  rollout impact.
- **Pains:** Version chaos ("which checklist is live at which stores"), no impact preview before a
  breaking template change, slow iteration cycles when standards change mid-quarter.
- **Primary device:** Desktop web (Builder), occasional tablet for field validation.
- **Connectivity reality:** Always-online for authoring; needs to validate how a workflow behaves
  offline before publishing broadly.
- **Success criteria:** Publish a template change with a clear impact preview (locations, work items,
  users, active schedules affected) and near-zero unplanned regression; shorter time from standard
  change to compliant frontline execution.

#### Regional/District Manager
- **Goals:** Monitor exception trends and location health across a territory; intervene where a site
  or cluster of sites is drifting from standard; balance visits and remote oversight.
- **Pains:** Reactive discovery of problems (a customer complaint or a corporate visit surfaces an
  issue that has existed for weeks); no reliable way to distinguish a one-off failure from a systemic
  pattern across the territory.
- **Primary device:** Mix of mobile (in transit, during site visits) and tablet/desktop (territory
  review sessions).
- **Connectivity reality:** Frequently on the road; expects the Manager Control Tower to work on
  patchy cellular connections and to sync quickly on reconnect.
- **Success criteria:** Time-to-detection and time-to-intervention for exceptions drop measurably;
  can explain to their own leadership why a location is at risk with evidence, not anecdote.

#### Store/Site Manager
- **Goals:** Run the local operation day to day — approve, reassign, resolve, verify work at their
  site; keep their team informed and compliant without becoming a full-time administrator.
- **Pains:** Constant context-switching between operating the floor and chasing paperwork; unclear
  prioritization when there are more open items than time; approvals stacking up while
  physically occupied on the floor.
- **Primary device:** Mobile phone (primary), occasionally a back-office tablet/PC for scheduling and
  reporting.
- **Connectivity reality:** Variable — back-of-house Wi-Fi may be poor or absent (walk-in coolers,
  basements, remote sites); must be able to approve/reassign/resolve from the floor.
- **Success criteria:** Reduced day-to-day admin overhead per shift; exceptions resolved before they
  escalate; team compliance is visible without manual chasing.

#### Frontline Associate
- **Goals:** Know exactly what to do next, complete it quickly, and move on with their shift — the
  app should get out of the way of the actual job.
- **Pains:** Slow apps, forms that assume connectivity, unclear prioritization ("what's actually
  urgent right now"), friction in capturing evidence (typing instead of scanning/photographing).
- **Primary device:** Mobile phone, frequently a shared/kiosk device at the site rather than a
  personally owned one.
- **Connectivity reality:** Frequently offline or on unreliable Wi-Fi/cellular (stockrooms, coolers,
  basements, rural sites, vehicles); must be fully functional with zero connectivity for a full
  shift.
- **Success criteria:** Sub-few-second task start/resume; can complete 100% of an assigned shift's
  work while fully offline and trust that it will sync; minimal typing, maximum camera/scan/tap.

#### Auditor/Inspector
- **Goals:** Collect defensible, timestamped, geo-tagged evidence during a visit; record findings
  with rigor sufficient to withstand a regulator or brand-standard challenge; drive follow-up to
  closure.
- **Pains:** Evidence that cannot be tied unambiguously to a location, time, and template version;
  no lightweight way to manage follow-up across many sites visited over a multi-week circuit.
  cannot always guarantee connectivity mid-audit (industrial sites, rural locations, warehouses).
- **Primary device:** Mobile phone or ruggedized tablet.
- **Connectivity reality:** Often intermittent or absent at the audit site itself; needs full offline
  parity with the Frontline Associate experience plus richer evidence/annotation tooling.
- **Success criteria:** Zero disputed findings due to missing/ambiguous evidence; every finding
  traceable to exact template version, capture metadata, and reviewer; follow-up tasks close on
  time.

#### Executive
- **Goals:** Understand enterprise-wide operational risk and performance at a glance; know what
  requires their attention versus what is self-correcting; defend operational integrity to the board,
  regulators, or franchise partners.
- **Pains:** Dashboards that show lagging, unexplained numbers with no "why" or "what to do";
  inability to distinguish noise from a systemic, spreading problem until it is large.
- **Primary device:** Tablet or desktop, occasionally mobile for a quick check between meetings.
- **Connectivity reality:** Always-online.
- **Success criteria:** Can answer "what is wrong, where, why, and is the fix working" in one
  session without commissioning a special report; measurable improvement in the Verified Operational
  Execution Rate north-star metric quarter over quarter.

#### AI Agent
- **Goals (as a system persona):** Observe operational events within its authorized scope, retrieve
  relevant context and history, reason within policy boundaries, recommend or execute permitted
  tools, and escalate to a human when confidence or risk exceeds its authorized autonomy level.
- **Pains (as a design constraint):** Cannot be productive or safe without well-scoped context, a
  typed and allowlisted tool surface, and an explicit policy engine — an agent given raw database
  access or unscoped context is a liability, not a persona to design for enablement of.
- **Primary "device":** Agent Runtime (server-side), surfaced to humans via the Agent Inbox/Approval
  surface.
- **Connectivity reality:** Always-online by construction (runs in the backend), but must gracefully
  handle upstream data that originated offline and is still syncing.
- **Success criteria:** High tool-selection correctness and policy adherence, low false-escalation and
  missed-critical-event rates, rising human trust reflected in a falling (but never zero-oversight)
  override rate at higher autonomy levels.

---

## 4. Product Principles

Each principle from `source-spec.md` §4 is retained verbatim as the requirement, with rationale and a
concrete product implication added here.

| # | Principle | Rationale | Concrete product implication |
|---|---|---|---|
| P1 | **Mobile-first** — the frontline experience is optimized for fast action, not data entry. | Frontline users are paid to run the operation, not to operate software; every extra second per interaction multiplies across millions of daily work-item completions enterprise-wide. | Task lists default to "what's next," not "browse everything"; forms default to camera/scan/tap components over free text (see §8.1 form component list); no screen requires more than a few taps to reach the primary action. |
| P2 | **Offline is normal** — a network outage must not stop work. | Frontline connectivity (coolers, basements, rural sites, vehicles, warehouses) is unreliable by physical environment, not by exception; treating it as an edge case guarantees production incidents and lost evidence. | Offline is a first-class, continuously tested operating mode across product, QA, telemetry, storage quotas, and conflict UX — not a fallback path (see Section 5 and module `04-mobile-offline`). |
| P3 | **Everything is scoped** — authorization follows tenant, organizational scope, role, attributes, and effective dates. | A frontline platform spanning thousands of legally and operationally distinct locations, brands, and franchise relationships cannot rely on role alone; scope leakage is both a security incident and a trust-destroying event for enterprise buyers. | Every API, cache key, search document, and AI context assembly step enforces `tenant_id` + `OrganizationNode` scope server-side (see `conventions.md` Multi-tenancy rule); an "Effective Access" preview is a first-class admin UI, not a debugging tool. |
| P4 | **Evidence first** — a claim of completion and a verified completion are distinct states. | "Marked done" and "actually done to standard" are different facts with different business consequences (liability, brand risk, food safety); conflating them is the core failure mode of legacy checklist tools. | The `WorkItem` state machine has explicit `SUBMITTED → PENDING_REVIEW → APPROVED → VERIFIED` states (see `conventions.md`); dashboards report Verified Operational Execution Rate, never raw completion rate, as the primary KPI. |
| P5 | **Policy before AI** — deterministic policy defines what is allowed; AI interprets context inside those boundaries. | Regulated, safety-relevant, multi-tenant operations cannot tolerate a model's probabilistic judgment silently overriding an enterprise rule; governance must be provable, not merely likely. | Agent Runtime always evaluates a deterministic policy engine (OPA/Rego, per `conventions.md`) before and after model reasoning; no LLM output executes a tool without a policy authorization check (see module `10-agentic-operations`). |
| P6 | **Progressive complexity** — frontline users see simple work; administrators see the enterprise machinery. | The same platform must serve a associate who wants three taps and an admin who wants to configure a 40-node org hierarchy; one UI cannot serve both without hiding complexity by role. | Mobile app surfaces are role-filtered to a minimal, task-shaped experience (see §7.4 Work Item UX); the Builder, Control Tower, and Agent Policy console are entirely separate progressive-disclosure surfaces gated by role/scope. |
| P7 | **Historical truth** — completed records are immutable with respect to the standard version used. | Reorganizations, template edits, and policy changes are constant in a multi-year enterprise deployment; if history re-renders against the current template, audit and legal defensibility collapse. | `TemplateVersion` is immutable after publish; a `Submission` always references the exact `template_version_id`, schema hash, rules version, and policy context in force at completion time (see `conventions.md` domain model). |
| P8 | **Explainability** — risk, AI recommendations, and automated actions expose their drivers and evidence. | An unexplained AI action or risk score is not actionable and erodes trust faster than an absent one; a manager or auditor must be able to defend a decision to their own leadership. | Every `AgentRun` records retrieved sources, tool calls, confidence, and policy decision; every Visual AI result carries a structured violation list with bounding boxes and confidence, not a bare score (see module `09-visual-ai`). |
| P9 | **Idempotency everywhere** — retries must be safe for mobile sync, jobs, webhooks, and agent actions. | Mobile networks, distributed workers, and third-party webhook senders all retry; without idempotency, retries silently duplicate work items, notifications, or financial-adjacent actions. | `Idempotency-Key` required on all unsafe mutation endpoints; submission identity is `tenant_id + client_uuid`; duplicate sync submissions return the original server identity with no duplicate insert (see `conventions.md` API conventions). |
| P10 | **Design for change** — business hierarchies, rules, workflows, model providers, and integrations must be versioned and replaceable. | Enterprise customers restructure, swap ERPs, and will demand model-provider flexibility as the AI market moves; hard-coding any of these forces a costly migration later. | Organization hierarchy is arbitrary-depth and effective-dated, not a fixed Enterprise→Region→Store schema; the Agent Runtime model gateway is provider-agnostic (LiteLLM-style abstraction per `conventions.md`); integrations are configuration-first with connection state and versioned connectors. |

---

## 5. Mobile-First Strategy

Mobile-first is not a UI style choice; it is the platform's central engineering and product bet, and
it must be understood as such by every module owner.

### 5.1 Why mobile-first, not mobile-only

Mobile-first means the frontline mobile app is the design and engineering priority for the majority
of the user base and the majority of daily transaction volume, while the web Control Tower and Admin
Console remain full, non-degraded surfaces for the administrative and analytical minority who need
dense data, bulk configuration, and multi-window workflows a phone cannot serve well. We are
explicitly **not** building mobile-only, because:

- Enterprise Admins, Operations Admins, and Executives genuinely need a desktop-class surface for
  hierarchy configuration, template authoring, and cross-territory analytics — cramming that into a
  phone would degrade the product for a persona whose buy-in gates the sale.
- Bulk operations (SCIM provisioning review, template impact preview across thousands of locations,
  integration configuration) require screen real estate and precision input that only a desktop
  affords.

The discipline is: **when a requirement is ambiguous about which surface owns it, resolve in favor of
the mobile experience for anything a Frontline Associate, Auditor, or Store Manager touches during
active work**, and resolve in favor of the web experience for anything an Enterprise Admin, Operations
Admin, or Executive touches during planning/oversight. Regional/District Managers straddle both
by design (Section 3) and both surfaces must serve them without compromise.

### 5.2 Design implications

- **One-handed use.** Primary actions (complete a step, capture evidence, approve/reject, escalate)
  must be reachable by thumb in the bottom two-thirds of the screen; multi-step wizards that assume
  two-handed operation are treated as a design defect, not a style preference.
- **Offline-by-default.** Every screen a Frontline Associate or Auditor can reach must render and
  accept input with zero network connectivity, backed by the local embedded store (WatermelonDB per
  `conventions.md`) — "offline mode" as a special banner state is explicitly disallowed by P2; the
  only allowed network-dependent affordance is an unobtrusive sync-status indicator.
- **Camera-first capture.** Evidence capture defaults to camera, barcode/QR scan, and signature
  capture over free-text entry wherever the form component library (§8.1) supports it; native camera
  pipeline modules (Kotlin/Swift per `conventions.md`) are treated as core product surface, not a
  peripheral integration.
- **Low-bandwidth, resumable sync.** The sync protocol (module `04-mobile-offline`, detailed in the
  platform FSD) must complete a typical reconnect sync well inside the non-functional target even on
  a degraded 2G/3G-equivalent link; large media uploads are chunked and resumable independently of
  the transactional sync payload so a dropped connection never forces a full evidence re-upload.
  Payload design defaults to sending deltas and cursors, never full-table refreshes, to keep
  data-plan cost bounded for the user and for the tenant paying for backend egress.
  offline. Photos and video use tenant-configurable compression profiles balancing evidentiary
  fidelity against upload cost.
- **Battery and data budget.** Background sync, location capture, and push-notification handling are
  scheduled through OS-native background task frameworks (WorkManager/BGTaskScheduler per
  `conventions.md`) rather than always-on polling, and every module's mobile feature spec must state
  its expected battery and data impact before it ships — this is a release gate, not an
  afterthought, consistent with the Definition of Done in the platform-wide QA process.

### 5.3 What mobile-first changes about web admin, APIs, and sync architecture

- **Web admin** is designed around the artifacts mobile produces (submissions, evidence, exceptions)
  rather than the other way around; the Control Tower's job is to make the aggregate of millions of
  mobile-originated events legible, not to be a parallel data-entry surface for the same work.
- **APIs** are designed device-first: the `/api/v1/sync` endpoint (per `conventions.md`) is a
  first-class, heavily optimized contract — cursor-based, idempotent, delta-oriented — not a generic
  CRUD API repurposed for mobile. Any new backend capability is evaluated for "how does this behave
  when the calling device has been offline for 8 hours and is now syncing on a weak connection"
  before it is considered complete.
- **Sync and conflict design** assumes the mobile client, not the server, is frequently the
  source of the most recent operationally relevant fact (a completed inspection, a captured photo);
  server-authoritative timestamping and domain-specific conflict resolution (per `conventions.md`
  state machines) exist precisely so that mobile-first does not become mobile-unsafe.
- **Notification and escalation design** must assume delivery to a device that may be offline for
  hours; SLA timers and escalation ladders (module `02-work-orchestration`) are evaluated
  server-side against wall-clock time, never against "last seen device time," so a manager cannot
  silently miss an SLA breach because their phone was in a basement.

---

## 6. Capability Map

This expands `source-spec.md` §5 with a one-line description and primary user outcome per capability.
Phase columns indicate initial availability tier; a blank cell means the capability is not the primary
focus of that phase (it may still receive incremental hardening).

| Capability | MVP | Phase 2 | Phase 3 | Description | Primary user outcome |
|---|---|---|---|---|---|
| Organization Graph + RBAC | Yes | | | Arbitrary-depth, effective-dated org hierarchy with RBAC/ABAC scope enforcement and SSO/SCIM. | Admins configure and trust access without manual permission spreadsheets; zero cross-scope leakage. |
| Work Orchestration | Yes | | | Universal work-item model covering tasks, audits, approvals, issues, training, campaigns, and more, with rich scheduling and SLA. | Every frontline user always knows what to do next and by when. |
| No-Code Forms | Yes | | | Drag-and-drop form builder with a rich component library, conditional logic, and versioning. | Ops Admins deploy a new standard as a live form in hours, not a dev cycle. |
| Workflow Builder | Yes | | | Event/schedule-triggered automation layered on the work model (draft→pilot→publish lifecycle). | Repetitive operational logic runs without manual triggering. |
| Offline Mobile | Yes | | | Full offline execution of assigned work with idempotent, resumable sync. | Associates and auditors complete 100% of a shift's work regardless of connectivity. |
| Evidence & Media | Yes | | | Photo/video/audio/document/signature/GPS capture with hashing, secure storage, and retention policy. | Every completion claim is backed by defensible, tamper-evident proof. |
| Issues/CAPA | Yes | | | Structured exception lifecycle with root-cause methods and effectiveness verification. | Deviations are owned, resolved, and prevented from recurring, not just logged. |
| Communications | Yes | Enhanced | | Targeted, acknowledgment-tracked announcements across in-app, push, email, and optional SMS. | Leadership can prove that critical information reached and was read by the right audience. |
| Knowledge | Basic | Yes | | Versioned, scope-aware operational knowledge base with authorization-enforced search and AI answers. | Frontline users get a trustworthy answer in seconds instead of calling a manager. |
| Learning | Basic | Yes | | Training assignments linked to operational outcomes, triggered by repeated failures. | Skill gaps are closed and measurably reduce repeat failures. |
| Visual AI | Pilot | Yes | Advanced | Computer-vision-assisted compliance checks (planogram, signage, cleanliness, PPE, asset condition). | Visual compliance is checked at a scale no human audit team could match, with human review where it matters. |
| Agent Runtime | Recommendation | Yes | Controlled autonomy | Policy-governed AI agents that observe, recommend, and (at increasing autonomy) execute permitted actions. | Routine resolution work is automated safely, freeing managers for judgment work. |
| Predictive Risk | | Yes | Advanced | Statistical/ML forecasting of exception likelihood and location risk. | Leaders intervene before a failure occurs, not after. |
| IoT/Assets | | Yes | Advanced | Asset and sensor-driven inspection and preventive-maintenance workflows. | Equipment failures are anticipated and prevented rather than discovered by a customer or safety incident. |

---

## 7. Module Overview

The 13 modules below decompose the platform for detailed specification. Each has its own
`functional-spec.md`, `high-level-design.md`, `low-level-design.md`, and `architecture-diagram.md`
under `/modules/<slug>/`, per the depth expectations in `conventions.md`. This PRD gives only the
product-level purpose and value of each.

### 7.1 Identity & Organization
Owns the `Tenant`, `OrganizationNode`, `User`, `UserScope`, and `Role` entities: arbitrary-depth,
effective-dated organizational hierarchy; SSO (SAML/OIDC) and SCIM provisioning; RBAC combined with
scope- and attribute-based access control; and the "Effective Access" preview that lets an admin see
exactly what a given user can do at a given location. This is the foundational trust layer every
other module depends on — without correct scoping here, no downstream feature (work assignment,
evidence access, AI context retrieval) can be trusted. Its key user value is giving Enterprise Admins
provable, auditable control over who can see and do what, across brands, franchises, and
reorganizations, without manual permission spreadsheets.
(see `/modules/01-identity-organization/` for detailed functional spec, HLD, LLD, and architecture)

### 7.2 Work Orchestration
The central domain of the platform: a universal `WorkItem` abstraction covering tasks, audits,
approvals, issues, training assignments, campaign tasks, site visits, asset inspections, surveys, and
incidents, driven by a rich scheduling engine (calendar, shift-based, event-triggered,
dependency-based, SLA-timed) and a well-defined state machine. Its key user value is that every
persona, from Associate to Executive, sees "what needs to happen, where, by whom, by when" through
one consistent model rather than a different mini-tool per work type, and that SLA breaches escalate
predictably instead of silently expiring.
(see `/modules/02-work-orchestration/` for detailed functional spec, HLD, LLD, and architecture)

### 7.3 Forms & Workflow Builder
Provides the no-code form component library (text, numeric, media, signature, GPS, barcode, lookups,
repeating groups, calculated fields) and logic layer (conditional visibility, validation, branching,
risk scoring), plus the workflow builder that turns events and schedules into governed automation.
Templates follow an immutable, versioned lifecycle (`DRAFT → REVIEW → APPROVED → PUBLISHED →
SUPERSEDED → RETIRED`) with an impact preview before publication. Its key user value is letting
Operations Admins turn a new or changed standard into deployed, versioned frontline work in hours,
with confidence about blast radius before they publish.
(see `/modules/03-forms-workflow-builder/` for detailed functional spec, HLD, LLD, and architecture)

### 7.4 Mobile & Offline
Owns the frontline mobile application shell and its offline-first data layer: the local embedded
database, sync queue, idempotent sync protocol, camera/scan pipeline, push/local notifications, and
resumable media upload. It is the direct engineering expression of the mobile-first strategy in
Section 5 and of Product Principle P2. Its key user value is that Frontline Associates and Auditors
can complete an entire shift or audit circuit with zero connectivity and trust that their work will
reconcile correctly and losslessly on reconnect.
(see `/modules/04-mobile-offline/` for detailed functional spec, HLD, LLD, and architecture)

### 7.5 Evidence Management
Treats evidence (photo, video, audio, document, signature, GPS, timestamp, device metadata,
barcode/QR scan, AI verification result) as a first-class, hashed, encrypted, retention-governed
object independent of the work item it supports. Its key user value is defensibility: an auditor,
regulator, or franchise partner can trust that a piece of evidence has not been altered and can be
traced to exact capture conditions, satisfying Product Principle P4 (evidence first) at the storage
and integrity layer.
(see `/modules/05-evidence-management/` for detailed functional spec, HLD, LLD, and architecture)

### 7.6 Issues & CAPA
Owns the exception lifecycle (`OPEN → TRIAGED → ASSIGNED → INVESTIGATING → CORRECTIVE_ACTION →
VERIFICATION → RESOLVED → CLOSED`, with reopening on failed verification) and the CAPA discipline —
immediate correction, containment, root cause (5 Whys, Fishbone, Pareto, FMEA, or a custom method),
corrective/preventive action, and effectiveness verification. Its key user value is converting a
detected deviation into an owned, time-bound, ultimately measured resolution rather than a
fire-and-forget alert, directly driving down the Repeat Failure Rate KPI.
(see `/modules/06-issues-capa/` for detailed functional spec, HLD, LLD, and architecture)

### 7.7 Communications & Knowledge
Delivers targeted, acknowledgment-tracked announcements (in-app, push, email, optional SMS/enterprise
messaging) with scheduling, expiry, translation, and audience preview, alongside a versioned,
scope-aware Knowledge Hub whose search and AI answers enforce the same authorization model as direct
document access. Its key user value is that leadership can prove critical information reached and
was understood by the right audience, and that a frontline worker can self-serve a trustworthy answer
in seconds instead of escalating to a manager.
(see `/modules/07-communications-knowledge/` for detailed functional spec, HLD, LLD, and architecture)

### 7.8 Learning
Links training directly to operational outcomes rather than treating it as a standalone LMS: a
repeated failure pattern can trigger a targeted microlearning assignment, a knowledge check, and a
follow-up verification workflow (`REPEATED FAILURE → IDENTIFY GAP → ASSIGN MICRO-LEARNING →
KNOWLEDGE CHECK → RE-INSPECTION → MEASURE IMPROVEMENT`). Its key user value is that training spend is
targeted and its effectiveness is measured against a real reduction in the failure it was assigned to
fix, not against completion-of-course vanity metrics.
(see `/modules/08-learning/` for detailed functional spec, HLD, LLD, and architecture)

### 7.9 Visual AI
Applies computer vision to operational photo/video evidence for planogram/shelf compliance,
promotional signage, brand execution, cleanliness, PPE/safety, and asset/vehicle condition, returning
a structured result contract (compliance score, confidence, per-violation bounding boxes, and an
`AI_RESULT_UNREVIEWED` status) rather than an opaque pass/fail. Its key user value is compliance
checking at a scale no human audit team could sustain, while Product Principle P8 (explainability)
and tenant-configured human-review thresholds keep a human in control of consequential
determinations.
(see `/modules/09-visual-ai/` for detailed functional spec, HLD, LLD, and architecture)

### 7.10 Agentic Operations
Hosts the governed Agent Runtime and the initial agent portfolio (Compliance Monitor, Operations
Copilot, Resolution Agent, Training Agent, Communication Agent, Audit Intelligence Agent, Visual
Inspection Agent, Executive Insights Agent, Workflow Builder Agent), each operating at a
policy-defined autonomy level (`L0 Observe` through `L4 Prohibited`) with a typed, allowlisted tool
surface and full run/tool/approval audit trail. Its key user value is that routine detection,
recommendation, and eventually low-risk resolution work is automated safely, under Product Principle
P5 (policy before AI), freeing managers and admins for judgment work that genuinely requires a human.
(see `/modules/10-agentic-operations/` for detailed functional spec, HLD, LLD, and architecture)

### 7.11 Analytics & Control Tower
Answers the seven executive questions (what is wrong, where, how serious, why, what has been done,
what needs intervention, is the intervention working) through role-specific dashboards — Frontline,
Manager, Regional, and Executive Control Tower views — built on the KPI set anchored by the Verified
Operational Execution Rate north-star metric. Its key user value is that every management layer gets
an explanation, not just a number, and can act on it from the same surface.
(see `/modules/11-analytics-control-tower/` for detailed functional spec, HLD, LLD, and architecture)

### 7.12 Integrations & Extensibility
Provides a configuration-first Integration Hub for standard connectors (identity, HR, ERP, data
warehouse, BI, collaboration, POS/inventory/workforce/maintenance systems) plus a REST/webhook/SDK
surface for custom enterprise integrations, with connection state, ownership, secret rotation, rate
limits, monitoring, retry policy, and audit logging on every integration. Its key user value is that
the platform fits into an enterprise's existing systems of record rather than forcing a rip-and-replace,
and that integration failures are observable and recoverable rather than silent.
(see `/modules/12-integrations-extensibility/` for detailed functional spec, HLD, LLD, and architecture)

### 7.13 Security, Privacy & Compliance
Owns tenant isolation enforcement across every layer (API, service, database, object storage, cache,
search, vector retrieval, event routing, workers, analytics, exports, and AI context assembly),
encryption, key management, device/session security, AI-specific data controls (PII classification,
redaction, model/provider allowlisting, no cross-tenant retrieval), and the data lifecycle through
retention and deletion. Its key user value is that enterprise security, legal, and compliance
stakeholders can approve and continue trusting the platform as the system of record for operational
and audit-relevant data, satisfying the acceptance tests in the platform-wide QA gates.
(see `/modules/13-security-compliance/` for detailed functional spec, HLD, LLD, and architecture)

---

## 8. Success Metrics & North-Star Metric

### 8.1 North-Star Metric

**Verified Operational Execution Rate** — verified work completed correctly, divided by work
expected — segmented by location, workflow type, risk category, evidence type, and verification
method. This metric is chosen deliberately over a raw "completion rate" because it encodes Product
Principle P4 directly: a claim of completion that has not been verified against evidence or approval
criteria does not count. It is the single number an Executive persona should be able to state from
memory, and every roadmap investment should be traceable to a plausible mechanism for moving it.

### 8.2 Product Health Metrics (expanded from `source-spec.md` §26.2)

| Area | Metrics | Primary consumer |
|---|---|---|
| Adoption | Weekly active frontline users; active locations; workflow/template activation rate; time-to-first-value for a newly onboarded tenant | Product, Customer Success |
| Execution | On-time completion rate; verified completion rate; overdue rate; average time-to-start for assigned work | Operations, Regional Managers |
| Quality | Repeat failure rate; first-pass verification rate; reopening rate on CAPA | Quality/Operations |
| Exceptions | Time to intervention; time to resolution; SLA breach rate by severity | Managers, Executives |
| Engagement | Announcement acknowledgment rate; knowledge search success/answer-with-source rate; training completion and post-training re-inspection pass rate | Communications/Learning owners |
| AI | Recommendation acceptance rate; override rate; false positive/negative rate by agent; tool-call success rate; autonomous action success rate; cost per resolved exception; latency per agent run | AI Governance |
| Reliability | Mobile sync success rate and p95 sync duration; event lag; notification delivery rate; API p95/p99 latency; error budget burn | Platform/SRE |
| Enterprise | Integration adoption rate; expansion locations added per quarter; admin configuration effort (time-to-publish a template); renewal/NRR drivers | Sales, Customer Success, Finance |

### 8.3 Metric Ownership and Cadence

Each pod defined in the Operating Model (Section 9.4) owns a subset of this table as its customer-
visible outcome measure and reports it at a standing operating review cadence (recommended: weekly
for Adoption/Execution/Reliability, monthly for Quality/Exceptions/AI, quarterly for Enterprise).
Metrics that regress for two consecutive reporting periods trigger a mandatory root-cause review
before further feature work in that pod is prioritized — the platform should apply its own CAPA
discipline to itself.

---

## 9. Implementation Roadmap

This expands `source-spec.md` §25 by tying each phase explicitly to the modules and epics (Appendix
E IDs, referenced in full in `02-Epics-and-Stories.md`) that ship within it.

### 9.1 Foundation (Months 0-4)

Establishes the trust and execution substrate with no compromise, because every later phase depends
on it being correct: tenant/organization graph, SSO/SCIM, RBAC/scope authorization, the core work
model, forms with template versioning, the mobile app shell, the offline local database and sync
protocol, the evidence service, basic notifications, and audit logging.

- Modules shipped at production depth: `01-identity-organization`, `02-work-orchestration`
  (core model only), `03-forms-workflow-builder` (forms only, workflow engine deferred),
  `04-mobile-offline`, `05-evidence-management` (capture + storage, retention policy simplified).
- Epics: EPIC-01 (Tenant & Organization Graph), EPIC-02 (Identity & Authorization), EPIC-03 (Work
  Core), EPIC-04 (Form Platform), EPIC-05 (Mobile Offline), EPIC-06 (Evidence), EPIC-14 partial
  (baseline security/isolation controls, audit logging).
- Exit criterion: a pilot tenant can complete the "offline submission" and "tenant isolation"
  acceptance tests (Section 24 of the platform FSD) end to end.

### 9.2 Operational Core (Months 4-8)

Turns the foundation into a genuinely operable product: the workflow engine, scheduling/SLA
escalation, issues/CAPA, the Manager Control Tower, communications, the Knowledge Hub, basic
learning, the public API/webhooks surface, the Integration Hub, and the analytics foundation.

- Modules shipped: `03-forms-workflow-builder` (workflow engine complete), `06-issues-capa`,
  `07-communications-knowledge` (communications at full depth, knowledge basic per capability map),
  `08-learning` (basic), `11-analytics-control-tower` (foundation/manager tier),
  `12-integrations-extensibility` (Integration Hub v1).
- Epics: EPIC-07 (Issues & CAPA), EPIC-08 (Comms & Knowledge), EPIC-09 (Learning, basic scope),
  EPIC-12 (Control Tower, manager tier), EPIC-13 (Integration Hub), EPIC-15 partial (analytics
  foundation).
- Exit criterion: a pilot tenant can run its full recurring-audit and exception-management program
  on the platform without a spreadsheet fallback.

### 9.3 Intelligence (Months 8-14)

Introduces AI as recommendation-tier, never autonomous, per the release path discipline
(`OFFLINE EVALUATION → SHADOW → RECOMMENDATION → APPROVAL MODE → CONTROLLED AUTONOMY`): Visual AI
pilot, the Operations Copilot, the Compliance Monitor agent, the agent policy engine and approval
inbox, predictive risk modeling, enterprise search/RAG for the Knowledge Hub, and outcome measurement
instrumentation.

- Modules shipped: `09-visual-ai` (pilot), `10-agentic-operations` (policy engine + two agents at
  L0/L1 autonomy), `07-communications-knowledge` (knowledge enhanced to RAG-backed), analytics
  extended with predictive risk.
- Epics: EPIC-10 (Visual AI), EPIC-11 (Agent Runtime, recommendation tier), EPIC-15 (Analytics & ML,
  predictive layer).
- Exit criterion: agent recommendations achieve target precision/recall and false-escalation rates in
  shadow mode before any tenant sees them in the Agent Inbox; AI policy enforcement acceptance test
  passes.

### 9.4 Autonomous Ops (Months 14-24)

Extends the agent portfolio into controlled execution: Resolution, Training, and Campaign agents
operating at L2/L3 autonomy where policy allows, asset/IoT-driven workflows, an agent marketplace and
a template/workflow marketplace, and advanced forecasting/optimization.

- Modules shipped: `10-agentic-operations` (full portfolio, controlled autonomy), `09-visual-ai`
  (advanced use cases), Predictive Risk and IoT/Assets capabilities at "Advanced" tier per the
  capability map.
- Epics: EPIC-11 (Agent Runtime, controlled-autonomy tier), EPIC-15 (advanced forecasting).
- Exit criterion: a tenant runs at least one L2 autonomous resolution flow in production with a
  measured outcome-improvement rate and a human override rate that is falling, not flat or rising.

### 9.5 Operating Model

Cross-functional product pods are aligned to customer outcomes rather than technology layers:
Frontline Execution, Enterprise Governance, Intelligence/AI, Data & Analytics, Integrations, and
Platform Reliability/Security. Each pod owns a customer-visible outcome and the corresponding rows of
the metrics table in Section 8.2, and is expected to map its module ownership to one or more of the
13 modules in Section 7.

---

## 10. Risks & Assumptions

| # | Risk / Assumption | Category | Mitigation |
|---|---|---|---|
| R1 | Frontline adoption fails to reach the threshold needed for Control Tower data to be trustworthy (garbage in, garbage out for every downstream KPI and agent). | Product/Adoption | Treat mobile-first (Section 5) and Product Principle P1 as non-negotiable release gates; instrument and alert on Frontline Adoption as a leading indicator, not a lagging one; pilot with real frontline users before broad rollout. |
| R2 | Offline sync produces silent data loss or duplication at scale (millions of devices, intermittent connectivity). | Technical | Idempotent submission identity (`tenant_id + client_uuid`), domain-specific conflict resolution rather than global last-write-wins, mandatory "offline submission" and "conflict safety" acceptance tests before every release (per `conventions.md` and platform FSD §19/§24). |
| R3 | Multi-tenant isolation defect exposes one enterprise customer's data to another — an existential trust failure for an enterprise SaaS product. | Security | Defense-in-depth scoping at every layer (API, service, DB via RLS, object storage, search, vector store, events, AI context) per `conventions.md`; mandatory authorization regression suite for horizontal/vertical escalation and IDOR; the "tenant isolation" acceptance test is a release-blocking gate, not a best-effort check. |
| R4 | Ungoverned or over-eager AI autonomy causes a harmful or reputationally damaging automated action (e.g., an incorrect customer-facing communication, an unsafe corrective instruction). | AI Governance | Strict adherence to the autonomy-level ladder (L0-L4) and the release path (`OFFLINE EVALUATION → SHADOW → RECOMMENDATION → APPROVAL MODE → CONTROLLED AUTONOMY`); policy-before-AI (P5) enforced by a deterministic policy engine independent of the model; tenant/agent/tool-level kill switches available at all times. |
| R5 | Visual AI produces biased or unreliable results across store formats, lighting conditions, or demographics of frontline environments, causing unfair compliance findings. | AI Quality/Fairness | Confidence thresholds and mandatory human-review states for low-confidence results (`AI_RESULT_UNREVIEWED`); per-tenant, per-use-case configurable thresholds; ongoing precision/recall evaluation segmented by site conditions; an appeals path for disputed AI findings. |
| R6 | Template/workflow version chaos: a breaking change to a live standard causes incorrect work assignment or historical re-interpretation across thousands of sites. | Product/Technical | Immutable `TemplateVersion` after publish, mandatory impact preview before production publication, and the "version immutability" acceptance test (historical submissions render against their original version even after later versions publish). |
| R7 | Enterprise customers churn integrations or identity providers mid-contract, breaking SSO/SCIM or data pipelines. | Market/Commercial | Configuration-first Integration Hub with per-integration connection state, monitoring, and owner; provider-agnostic identity abstraction (SAML/OIDC) and model gateway design (per `conventions.md`) so no single vendor dependency is load-bearing for the core product. |
| R8 | Regulatory or data-residency requirements (health, safety, food-safety, regional privacy law) vary widely by geography and industry vertical, and a one-size architecture cannot satisfy all of them. | Compliance/Legal | Regional data-plane options at the tenant tier (per Document Control in source-spec.md), configurable retention and AI data controls (PII classification, redaction, model/provider allowlists), and per-tenant AI feature disablement by data class. |
| R9 | Competitive response: an incumbent checklist vendor or a well-funded AI-native entrant closes the feature gap faster than we build the full Operational Graph. | Market/Competitive | Sequence the roadmap (Section 9) so the trust substrate (identity, scoping, evidence, audit) ships first and is defensible on its own; treat the graph, not any single feature, as the moat per Section 2, and avoid feature-race distractions that don't deepen it. |
| R10 | Underestimating mobile engineering complexity (offline conflict resolution, resumable media upload, cross-platform camera/scan native modules) causes schedule slip in the Foundation phase that cascades through the entire roadmap. | Execution/Technical | Dedicate a specialized mobile/offline engineering track from day one (not a generalist web team ported to React Native); treat `04-mobile-offline` as the highest-risk module and staff/QA it accordingly; build the sync protocol and conflict semantics before layering feature modules on top. |
| R11 | Assumption: enterprise buyers will accept a phased AI autonomy rollout rather than demanding full automation immediately for competitive reasons. | Market Assumption | Make the recommendation-tier product genuinely valuable on its own (Operations Copilot, Compliance Monitor) so tenants adopt and see ROI before autonomy is even discussed; do not gate core value on autonomy being enabled. |
| R12 | Assumption: the target tech stack (per `conventions.md`) scales to the stated non-functional targets (100k locations, 1M+ users, millions of daily work items) without a major re-architecture. | Technical Assumption | Validate with load testing against the Non-Functional Requirements targets before GA; design tenant-tier sharding (dedicated schema/cluster for large tenants) into the architecture from the Foundation phase rather than retrofitting it later. |

---

## 11. Out of Scope / Non-Goals

To keep the platform's scope coherent with its category definition (Section 2) and avoid diluting
engineering focus, the following are explicit non-goals for this specification and the roadmap in
Section 9. Any of these may become a deliberate future initiative, but only through a new PRD, not by
incremental scope creep inside an existing module.

- **Point-of-sale (POS) or payment processing.** The platform integrates with POS/inventory/
  workforce/maintenance systems (Section 15 of `source-spec.md`, module `12-integrations-extensibility`)
  but does not become a POS, payments, or transaction-processing system itself.
- **General-purpose HRIS/payroll.** The platform consumes identity and org data from HR systems via
  integration and SCIM; it does not replace an HRIS, run payroll, or manage benefits.
- **General-purpose enterprise messaging/collaboration replacement.** Communications (module
  `07-communications-knowledge`) is purpose-built for operational announcements and knowledge, not a
  replacement for Slack/Teams-class general collaboration; the platform integrates with those tools
  rather than competing with them.
- **Fully unsupervised (L4-equivalent) autonomous action on irreversible, high-impact operations.**
  Per the autonomy ladder, some actions are permanently `L4 Prohibited` regardless of AI maturity
  (e.g., irreversible financial commitments, employment actions); this is a durable governance
  boundary, not a temporary Phase 1 limitation.
- **Consumer-facing applications.** The platform is a B2B enterprise operations system; it does not
  build a customer-facing loyalty app, ordering app, or public-facing storefront, even though
  operational data it manages (e.g., cleanliness, compliance) may indirectly affect the customer
  experience those separate systems deliver.
- **Building a foundation model.** The Agent Runtime is provider-agnostic and consumes third-party
  models (per `conventions.md`); training or hosting a proprietary foundation model is out of scope.
- **Native desktop or offline-first web admin.** The web Control Tower/Admin Console assumes reliable
  connectivity (Section 3.1 Enterprise Admin/Operations Admin personas); it is not engineered for the
  offline-first guarantees that apply to the mobile app.
- **Physical IoT hardware manufacturing.** The platform consumes data from third-party IoT
  sensors/assets (Phase 2/3 capability) but does not design, manufacture, or directly firmware-manage
  sensor hardware.
- **Industry-specific regulatory certification as a product feature per vertical (e.g., becoming a
  certified food-safety HACCP authority, OSHA certifying body).** The platform supports compliance
  workflows against customer-defined or industry-standard criteria; it does not itself act as a
  certifying or regulatory body.
- **Replacing a full business-intelligence/data-science platform.** Analytics (module
  `11-analytics-control-tower`) delivers operational KPIs and a control tower; deep, arbitrary
  ad-hoc data science is expected to happen downstream in the customer's own warehouse/BI tooling via
  the CDC/lakehouse export path (per `conventions.md`), not inside this product.

---

## 12. Glossary

Extends Appendix D of `source-spec.md` with additional terms introduced by this document and the
module decomposition.

| Term | Definition |
|---|---|
| Operational Graph | The linked enterprise representation of people, locations, assets, standards, work, evidence, issues, knowledge, and outcomes — the platform's durable differentiator (Section 2). |
| Work Item | Any actionable operational unit assigned to a user, role, location, or automated process; the universal domain object of Work Orchestration (module `02-work-orchestration`). |
| Evidence | Data proving or supporting an operational event or completion (photo, video, audio, document, signature, GPS, timestamp, device metadata, scan, AI verification result). |
| Issue | A managed exception requiring ownership, SLA, resolution, and verification. |
| CAPA | Corrective and Preventive Action workflow for resolving and preventing recurrence of an issue. |
| Agent | A governed AI runtime capable of observation, reasoning, recommendation, and/or permitted tool execution within a policy-defined autonomy level. |
| Autonomy Level | Policy-defined degree of action an agent can take without human approval, from `L0 Observe` to `L4 Prohibited`. |
| Effective-dated | A record whose validity is bounded by time, enabling historical truth to be preserved after organizational change. |
| Verified Completion | Completion supported by configured evidence and/or approval criteria, distinct from a mere completion claim (Product Principle P4). |
| Control Tower | Role-specific enterprise dashboard for monitoring operational health and exceptions (Frontline, Manager, Regional, Executive tiers). |
| Mobile-first | The strategic and design discipline of treating the frontline mobile application as the primary product surface for the majority of users and transaction volume, while maintaining a full, non-degraded web surface for administrative and analytical personas (Section 5). |
| Offline-first | The engineering discipline of treating zero-connectivity operation as a normal, continuously tested mode rather than a degraded fallback (Product Principle P2). |
| Impact Preview | The pre-publication simulation shown to an admin before a template/workflow change goes live, covering affected locations, work items, users, schedules, and reporting dimensions. |
| Tenant Isolation | The set of enforcement mechanisms (API, service, database, storage, search, events, AI context) that guarantee one tenant's data is never accessible to another. |
| Autonomy Ladder / Release Path (AI) | The staged rollout discipline for any AI capability: `OFFLINE EVALUATION → SHADOW → RECOMMENDATION → APPROVAL MODE → CONTROLLED AUTONOMY`. |
| North-Star Metric | The single top-level product success measure — Verified Operational Execution Rate (Section 8.1) — to which all other metrics and roadmap decisions should trace. |
| Pod (Operating Model) | A cross-functional product team aligned to a customer-visible outcome (e.g., Frontline Execution, Intelligence/AI) rather than to a single technology layer. |
