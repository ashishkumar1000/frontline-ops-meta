# 02 — Epics and Stories

This document decomposes the 15 build-ready epics defined in `source-spec.md` Appendix E into
detailed user stories. It is the traceability anchor referenced by every module's
`functional-spec.md` as `EPIC-xx.S-yy`. Personas are taken verbatim from source-spec.md §3.
Priority aligns to the PRD Capability Map (source-spec.md §5) and the Implementation Roadmap
(source-spec.md §25). State machines, entities and API conventions referenced below follow
`conventions.md` exactly.

Legend — Priority: **MVP** (Foundation/Operational Core, roadmap phases 25.1–25.2), **Phase 2**
(Intelligence, phase 25.3), **Phase 3** (Autonomous Ops, phase 25.4). Size: **S** (≤3 days),
**M** (~1 sprint), **L** (~2 sprints), **XL** (multi-sprint, likely split at implementation time).

---

## EPIC-01 — Tenant & Organization Graph

**Module:** `01-identity-organization`
**Goal:** Give every tenant an arbitrary-depth, effective-dated organization graph that supports
corporate/franchise/brand/business-unit dimensions, so that all downstream work, access and
reporting can be scoped and attributed correctly over time.

### EPIC-01.S-01 — Provision a new tenant
As an **Enterprise Admin**, I want to provision a new tenant with an isolation boundary, plan, and
data residency setting, so that our organization gets a dedicated, policy-governed workspace.
- Given a valid provisioning request, when the tenant is created, then a `Tenant` row is created with `id`, `status=ACTIVE`, `plan`, `residency`, and default `policies`.
- Given a tenant is created, when any table is queried, then `tenant_id` is enforced server-side on every path (API, cache, search, object storage, events).
- Given a residency setting of e.g. `EU`, when data is persisted, then it is written only to the region-pinned data plane.
- Given tenant creation, then a default root `OrganizationNode` and default `Role` set are seeded automatically.
Priority: MVP | Size: M | Dependencies: none (foundational)

### EPIC-01.S-02 — Build an arbitrary-depth organization hierarchy
As an **Enterprise Admin**, I want to create and nest `OrganizationNode`s of any type and depth
(not a hard-coded Enterprise→Country→Region→Store chain), so that our real-world structure is
represented accurately.
- Given a tenant, when an admin creates a node, then it stores `id, tenant_id, parent_id, type, path, valid_from/valid_to`.
- Given a node with children, when the tree is rendered, then depth is unbounded and `path` (materialized path) supports efficient subtree queries.
- Given a node type list (e.g. Enterprise, Country, Region, District, Store, Department), when an admin defines custom types, then the system accepts them without a schema migration.
- Given an attempt to create a cycle (child becomes ancestor of its own parent), when saved, then the API returns `422` with code `INVALID_HIERARCHY`.
Priority: MVP | Size: L | Dependencies: EPIC-01.S-01

### EPIC-01.S-03 — Effective-dated hierarchy changes
As an **Enterprise Admin**, I want hierarchy moves (e.g., a store reassigned to a new region) to be
effective-dated, so that historical reports still attribute past events to the structure in force
at that time.
- Given a node move scheduled for a future `valid_from`, when the change date arrives, then the old edge is closed with `valid_to` and a new edge opens with matching `valid_from`.
- Given a report for a date range spanning the move, when generated, then events before the move roll up to the old parent and events after roll up to the new parent.
- Given the "Reorganization" acceptance test (source-spec §24.1), when a store moves between regions, then historical attribution for prior events is unchanged.
- Given an attempted retroactive edit to a closed historical edge, when submitted, then the API rejects it with `IMMUTABLE_HISTORY`.
Priority: MVP | Size: L | Dependencies: EPIC-01.S-02

### EPIC-01.S-04 — Model brand, franchise and business-unit dimensions
As an **Enterprise Admin**, I want to tag nodes with corporate/franchise/brand/business-unit
relationships orthogonal to the reporting tree, so that franchise owners and brand leaders can view
their own slice.
- Given a node, when a `business_unit` or `brand` attribute is set, then it is queryable independent of the parent/child path.
- Given a franchise relationship, when a franchisee admin logs in, then they see only nodes tagged to their franchise entity, scoped by `UserScope`.
- Given multiple brands under one corporate parent, when a brand-level report is run, then results aggregate correctly across non-contiguous subtrees sharing the brand tag.
Priority: MVP | Size: M | Dependencies: EPIC-01.S-02

### EPIC-01.S-05 — Bulk import and validate organization structure
As an **Enterprise Admin**, I want to bulk-import locations and hierarchy via CSV/API, so that
onboarding thousands of sites does not require manual entry.
- Given a CSV upload with parent references by external ID, when validated, then row-level errors (missing parent, duplicate external ID, cycle) are reported before commit.
- Given a valid file, when imported, then nodes are created transactionally with an import batch ID for rollback.
- Given 100,000+ rows, when imported, then the job runs asynchronously with progress and a completion notification.
Priority: MVP | Size: M | Dependencies: EPIC-01.S-02

### EPIC-01.S-06 — Visual hierarchy configuration UI
As an **Enterprise Admin**, I want a visual tree editor for nodes, scopes, roles and assignments, so
that I can configure the org graph without raw data entry.
- Given the hierarchy screen, when an admin expands a node, then children load lazily (paginated) for trees with 100k+ nodes.
- Given drag-and-drop of a node to a new parent, when confirmed, then the system prompts for an effective date and previews affected users/work items.
- Given search-by-name/external-ID, when used, then the tree auto-scrolls and highlights the match.
Priority: MVP | Size: L | Dependencies: EPIC-01.S-02, EPIC-01.S-03

### EPIC-01.S-07 — Node lifecycle (activate/suspend/close)
As an **Operations Admin**, I want to open, temporarily suspend, and permanently close locations, so
that seasonal closures and site closings are reflected without deleting history.
- Given a node set to `CLOSED` with an effective date, when work is scheduled after that date, then no new `WorkItem`s are generated for that node.
- Given a `SUSPENDED` node (e.g., remodel), when queried, then it is excluded from active dashboards but retained in historical reports.
- Given a closed node, when historical submissions are viewed, then they remain fully accessible and unaffected.
Priority: MVP | Size: S | Dependencies: EPIC-01.S-02

### EPIC-01.S-08 — Organization graph API and events
As an **AI Agent**, I want a typed `get_location()` tool and org-graph read API scoped to my
authorized context, so that I can retrieve accurate location context without direct DB access.
- Given a tool call `get_location(location_id)`, when the caller's context does not include that node's subtree, then the call is denied and logged.
- Given a node create/update/move, when persisted, then a canonical event (`org_node.created`, `org_node.moved`) is published in the standard envelope (source-spec §18.4).
- Given the REST API, when `GET /api/v1/org-nodes?cursor=...` is called, then results are cursor-paginated and tenant/scope filtered server-side.
Priority: MVP | Size: M | Dependencies: EPIC-01.S-02, EPIC-11 (Agent Runtime context builder)

### EPIC-01.S-09 — Org graph impact preview for structural changes
As an **Enterprise Admin**, I want to preview the impact of a hierarchy change (affected users,
active schedules, in-flight work) before committing it, so that I avoid breaking active operations.
- Given a proposed node move or close, when previewed, then the system lists affected `WorkItem`s, `UserScope`s, and active schedules.
- Given the preview shows in-flight critical work, when the admin proceeds, then a confirmation with explicit acknowledgement is required.
Priority: Phase 2 | Size: M | Dependencies: EPIC-01.S-03, EPIC-03

---

## EPIC-02 — Identity & Authorization

**Module:** `01-identity-organization`
**Goal:** Provide enterprise-grade SSO/SCIM identity and a RBAC + scope + attribute authorization
model with a verifiable "Effective Access" preview, so access is provably correct at every scope
and never leaks across tenants.

### EPIC-02.S-01 — SAML 2.0 SSO login
As an **Enterprise Admin**, I want to configure SAML 2.0 SSO for our IdP, so that our employees
authenticate with corporate credentials.
- Given a configured SAML metadata exchange, when a user authenticates via IdP, then a session is issued mapping the SAML `NameID` to a `User.identity_ref`.
- Given SSO is enforced, when a user attempts password login, then it is rejected per tenant policy.
- Given an IdP certificate rotation, when the admin uploads new metadata, then existing sessions are unaffected and new logins validate against the new cert.
Priority: MVP | Size: L | Dependencies: EPIC-01.S-01

### EPIC-02.S-02 — OIDC SSO login
As an **Enterprise Admin**, I want to configure OIDC as an alternative to SAML, so that IdPs that
only support OIDC (e.g., Okta, Azure AD OIDC apps) can integrate.
- Given a configured OIDC client, when a user completes the auth-code flow, then an ID token is validated (issuer, audience, signature, expiry) before session issuance.
- Given token refresh, when the access token expires, then the mobile/web client silently refreshes without re-prompting login.
Priority: MVP | Size: M | Dependencies: EPIC-02.S-01

### EPIC-02.S-03 — SCIM provisioning and deprovisioning
As an **Enterprise Admin**, I want SCIM-based automated user provisioning/deprovisioning from our
identity provider, so that joiners/leavers/role changes propagate without manual admin work.
- Given a SCIM `POST /Users` request, when received, then a `User` is created/linked and default `UserScope` applied per group mapping.
- Given a SCIM deactivation, when processed, then the user's status becomes `INACTIVE` within 5 minutes and they can no longer create new work, but historical actions remain attributable (per acceptance test in source-spec §24.1).
- Given a SCIM group membership change, when synced, then `UserScope` rows are added/removed accordingly, effective-dated at sync time.
Priority: MVP | Size: L | Dependencies: EPIC-02.S-01, EPIC-02.S-02

### EPIC-02.S-04 — Define roles and permission sets
As an **Enterprise Admin**, I want to define roles composed of granular permissions (read, create,
approve, reassign, export, administer per object type), so that access matches least-privilege
principles.
- Given a role definition, when saved, then it is versioned and existing `UserScope` assignments reference the role by ID, not a copy of permissions.
- Given a permission matrix UI, when an admin edits a role, then a diff of affected users is shown before save.
- Given a built-in role (e.g., Store Manager), when cloned, then the clone is editable without altering the original.
Priority: MVP | Size: M | Dependencies: EPIC-01.S-01

### EPIC-02.S-05 — Scope-based role assignment with inheritance
As an **Enterprise Admin**, I want to assign a role to a user at a specific `OrganizationNode` with
inheritance to child nodes by default, so that a Regional Manager automatically covers all stores in
the region without per-store assignment.
- Given a `UserScope` at a region node, when a user requests access to a child store, then access is granted via inherited scope.
- Given an explicit deny/exception at a child node, when evaluated, then the deny overrides the inherited grant for that subtree only.
- Given a `valid_from`/`valid_to` on `UserScope`, when the current date is outside that range, then access is not granted.
Priority: MVP | Size: L | Dependencies: EPIC-02.S-04, EPIC-01.S-02

### EPIC-02.S-06 — Attribute-based policy overlay (ABAC)
As an **Enterprise Admin**, I want attribute-based rules (e.g., only users with a "Food Safety
Certified" attribute may approve temperature-exception corrective actions), so that access reflects
real qualifications, not just role/scope.
- Given an ABAC rule referencing a user attribute, when evaluated, then RBAC+scope grants are further filtered by the attribute condition.
- Given a policy authored in OPA/Rego (per conventions.md), when deployed, then it is versioned and testable in isolation via a policy simulation endpoint.
Priority: Phase 2 | Size: L | Dependencies: EPIC-02.S-05

### EPIC-02.S-07 — Effective Access preview
As an **Enterprise Admin**, I want to select any user and any location and see exactly what they can
read, create, approve, reassign, export, and administer, so that I can verify authorization
correctness before an audit or incident.
- Given a user+location pair, when previewed, then the UI lists the resolved permission set with the contributing role(s), scope path, and any overriding deny.
- Given a user with multiple overlapping scopes, when previewed, then the union/override resolution is shown, not just the first match.
- Given the preview is exported, then it produces an audit-log entry recording who ran the check and when.
Priority: MVP | Size: M | Dependencies: EPIC-02.S-05, EPIC-02.S-06

### EPIC-02.S-08 — Authorization regression test suite
As an **Enterprise Admin** (via Engineering), I want an automated regression suite covering
horizontal/vertical privilege escalation, IDOR, stale permissions, and cross-tenant access, so that
authorization regressions are caught before release.
- Given a suite run, when a request attempts to access another tenant's object by guessing an ID, then it is denied with `SCOPE_DENIED` and logged.
- Given a demoted user whose cached permission has not expired, when they attempt a now-disallowed action, then the server-side check (not client cache) denies it.
- Given CI, when a PR touches an authorization path, then the suite is required to pass before merge.
Priority: MVP | Size: L | Dependencies: EPIC-02.S-05, EPIC-02.S-06

### EPIC-02.S-09 — Session and device controls
As an **Enterprise Admin**, I want to configure session timeout, concurrent session limits, and
device-level restrictions (e.g., MDM-enrolled only), so that enterprise security policy is enforced.
- Given an idle timeout policy, when exceeded, then the session is invalidated and the user is re-prompted for auth.
- Given a policy limiting concurrent sessions, when a new session exceeds the limit, then the oldest is revoked with a notification.
- Given MFA is required by tenant policy, when a user without MFA attempts login, then they are forced into MFA enrollment.
Priority: MVP | Size: M | Dependencies: EPIC-02.S-01

### EPIC-02.S-10 — Delegated / temporary access
As a **Store/Site Manager**, I want to delegate my approval authority to a covering manager for a
defined time window, so that operations continue during my absence without permanent scope changes.
- Given a delegation with `valid_from/valid_to`, when the window is active, then the delegate can perform the delegated actions and this is visible in the Effective Access preview.
- Given the window ends, then access reverts automatically with no manual cleanup.
- Given an approval performed by a delegate, when audited, then the `AuditLog` records both the delegate and the delegating principal.
Priority: Phase 2 | Size: M | Dependencies: EPIC-02.S-05, EPIC-02.S-07

---

## EPIC-03 — Work Core

**Module:** `02-work-orchestration`
**Goal:** Provide the universal `WorkItem` abstraction covering all work types, scheduling models,
dependencies, and SLA/escalation, so any operational activity — task, audit, approval, training,
campaign — is governed by one consistent lifecycle.

### EPIC-03.S-01 — Create and assign a work item
As a **Store/Site Manager**, I want to create a work item of any type (Task, Audit, Approval,
Training Assignment, etc.) and assign it to a user, role, or location, so that frontline work is
tracked centrally.
- Given a work item is created, then it is persisted with `id, type, location_id, assignment, status=DRAFT, due_at` per the domain model.
- Given an assignment to a role (not a named user), when the work becomes `AVAILABLE`, then it appears in the queue of any user holding that role at the location.
- Given a work item without a location, when created, then the API rejects it with `422` unless the type explicitly allows location-less work (e.g., enterprise-wide announcement ack).
Priority: MVP | Size: M | Dependencies: EPIC-01, EPIC-02

### EPIC-03.S-02 — Work item state machine enforcement
As a **Frontline Associate**, I want the system to enforce the canonical work state machine
(`DRAFT→SCHEDULED→AVAILABLE→IN_PROGRESS→SUBMITTED→PENDING_REVIEW→APPROVED→VERIFIED`, with
`BLOCKED`, `REJECTED`, `OVERDUE→ESCALATED` branches), so that status is always consistent and
trustworthy.
- Given a work item in `AVAILABLE`, when I start it, then it transitions to `IN_PROGRESS` with a timestamp and actor recorded.
- Given an illegal transition attempt (e.g., `DRAFT`→`VERIFIED`), when submitted, then the API returns `409 CONFLICT` with the current valid transitions.
- Given a `PENDING_REVIEW` item rejected by a reviewer, then it transitions to `REJECTED` then back to `IN_PROGRESS` with the rejection reason attached.
Priority: MVP | Size: L | Dependencies: EPIC-03.S-01

### EPIC-03.S-02b — Optimistic concurrency on work item updates
As an **AI Agent**, I want work item mutations to require a version/ETag check, so that concurrent
updates from mobile, web, and agents never silently overwrite each other.
- Given two concurrent updates with the same base version, when the second is submitted, then it returns `409 Conflict` with a machine-readable `conflict` payload (per conventions.md).
- Given the "Conflict safety" acceptance test (source-spec §24.1), when two managers update the same task concurrently, then a deterministic conflict is surfaced, never a silent overwrite.
Priority: MVP | Size: M | Dependencies: EPIC-03.S-02

### EPIC-03.S-03 — Calendar-based recurrence
As an **Operations Admin**, I want to define calendar-based recurring work (daily/weekly/monthly),
so that standard operating tasks regenerate automatically.
- Given a recurrence rule (RFC 5545-style RRULE), when the schedule runs, then new `WorkItem`s are generated for each occurrence at the configured lead time.
- Given a recurrence spanning a hierarchy change, when locations move (EPIC-01.S-03), then future occurrences follow the new structure while past occurrences remain attributed to the old one.
- Given a Temporal.io-backed schedule (per conventions.md), when the worker restarts, then no occurrences are duplicated or dropped.
Priority: MVP | Size: L | Dependencies: EPIC-03.S-01, EPIC-01.S-03

### EPIC-03.S-04 — Opening/closing relative and shift-based schedules
As an **Operations Admin**, I want to schedule work relative to a location's opening/closing time or
shift boundaries, so that operational checklists trigger at the right real-world moment per site.
- Given a store's opening time from its operating-hours reference data, when the relative schedule fires, then the work item's `due_at` is computed relative to that time, honoring the location's timezone.
- Given a shift-based schedule, when a new shift starts (per workforce integration or manual shift record), then a shift-opening work item is generated.
- Given a location's hours change, when a schedule next evaluates, then it uses the updated hours without manual template edits.
Priority: MVP | Size: L | Dependencies: EPIC-03.S-03

### EPIC-03.S-05 — Event-triggered and dependency-based schedules
As an **Operations Admin**, I want work items to be triggered by domain events (e.g.,
`issue.created`) or by completion of a prerequisite work item, so that workflows model real
operational sequencing.
- Given a dependency-based rule ("Task B available only after Task A is VERIFIED"), when Task A is verified, then Task B transitions from `DRAFT`/`SCHEDULED` to `AVAILABLE`.
- Given an event-triggered rule subscribed to `issue.created` with severity=CRITICAL, when such an event is published, then a follow-up work item is created within the configured SLA.
- Given a circular dependency configuration, when validated at publish time, then the system rejects it with a clear error.
Priority: MVP | Size: L | Dependencies: EPIC-03.S-01, EPIC-07

### EPIC-03.S-06 — Role-triggered and location-triggered schedules
As an **Operations Admin**, I want work to auto-generate when a new user is assigned a role or a
new location is activated, so that onboarding work is never missed.
- Given a new `UserScope` granting the "Store Manager" role, when created, then an onboarding work item is auto-assigned to that user.
- Given a new location activation (EPIC-01.S-07), when it occurs, then all applicable standing work item templates are instantiated for that location.
Priority: Phase 2 | Size: M | Dependencies: EPIC-03.S-03, EPIC-02.S-05

### EPIC-03.S-07 — SLA timers and escalation ladders
As a **Regional/District Manager**, I want SLA timers with configurable multi-step escalation
ladders, so that overdue critical work automatically escalates to me and then to my leadership.
- Given a work item approaching its SLA deadline, when the warning threshold is reached, then a reminder notification fires to the assignee.
- Given the SLA deadline passes unresolved, then the item transitions `AVAILABLE/IN_PROGRESS → OVERDUE → ESCALATED` and the next ladder rung (e.g., District Manager) is notified.
- Given the "Escalation" acceptance test (source-spec §24.1), when a critical issue is unacknowledged, then it follows the configured SLA ladder and stops/escalates per policy — never escalates indefinitely past the configured ceiling.
- Given escalation ladders are implemented on Temporal.io durable timers (per conventions.md), when the service restarts mid-timer, then no escalation step is lost or duplicated.
Priority: MVP | Size: L | Dependencies: EPIC-03.S-02

### EPIC-03.S-08 — Seasonal / campaign schedules
As an **Operations Admin**, I want to launch a time-bound campaign (e.g., holiday planogram
compliance) across a selected set of locations, so that time-limited initiatives are tracked
distinctly from standing operations.
- Given a campaign definition with a location filter and date range, when launched, then Campaign Task work items are generated only for matching locations within the window.
- Given a campaign dashboard, when viewed, then completion/verification rates are shown scoped to that campaign only, separate from standing KPIs.
Priority: MVP | Size: M | Dependencies: EPIC-03.S-01, EPIC-12

### EPIC-03.S-09 — Frontline "Today" work queue
As a **Frontline Associate**, I want a prioritized "Today" view showing urgent/due/overdue counts
and a ranked next-actions list, so that I always know what to do next without hunting through menus.
- Given assigned work items due today, when the app loads, then counts for urgent/due/overdue render from local cache within the mobile startup performance target (source-spec §23).
- Given multiple eligible items, when ranked, then the ordering considers due time, severity, and SLA proximity.
- Given the device is offline, when the Today view loads, then it renders from the local WatermelonDB cache with no network dependency (per conventions.md mobile stack).
Priority: MVP | Size: M | Dependencies: EPIC-03.S-01, EPIC-05

### EPIC-03.S-10 — Manager exceptions/approvals/team view
As a **Store/Site Manager**, I want a workspace showing Exceptions, Approvals, Team status, and
Location health at a glance, so that I can run the location without switching between many screens.
- Given pending approvals, when viewed, then each shows evidence, requester, and SLA remaining.
- Given team status, when viewed, then it shows each direct report's open/overdue work counts.
- Given the manager taps an exception, then they can reassign, escalate, or resolve directly from the workspace.
Priority: MVP | Size: L | Dependencies: EPIC-03.S-07, EPIC-07

### EPIC-03.S-11 — Reassignment and bulk work operations
As a **Regional/District Manager**, I want to reassign or bulk-update work items across a set of
locations (e.g., extend due date for a supply delay), so that I can respond to operational
disruptions without editing each item.
- Given a bulk reassignment, when applied, then each affected `WorkItem` records the change in its history and emits `task.assigned` per item.
- Given a bulk operation on 10,000+ items, when submitted, then it runs asynchronously with a progress indicator and a completion summary (succeeded/failed counts).
Priority: Phase 2 | Size: M | Dependencies: EPIC-03.S-01, EPIC-02.S-05

### EPIC-03.S-12 — Work item audit history
As an **Auditor/Inspector**, I want a full chronological history of every status change, assignment,
and comment on a work item, so that I can reconstruct exactly what happened for compliance review.
- Given a work item, when its history is viewed, then every transition is listed with actor, timestamp (server-authoritative), and before/after state.
- Given the item was completed under a prior `TemplateVersion`, when viewed later, then it still renders against the exact version used (Version Immutability acceptance test, source-spec §24.1).
Priority: MVP | Size: M | Dependencies: EPIC-03.S-02, EPIC-04

---

## EPIC-04 — Form Platform

**Module:** `03-forms-workflow-builder`
**Goal:** Deliver a no-code form/workflow builder with rich field types, conditional logic, and
immutable versioning, so operational standards can be authored, safely changed, and reliably
replayed against historical submissions.

### EPIC-04.S-01 — Build a form with core field types
As an **Operations Admin**, I want to build a form using the full component palette (text, number,
currency, percentage, single/multi-select, date/time, yes/no, rating, matrix, barcode/QR, GPS,
signature, photo/video/audio, document upload, lookups, repeating groups, calculated fields), so
that I can model any operational checklist without engineering help.
- Given the builder canvas, when a field is dragged in, then it is configurable (label, required, help text, default value) without code.
- Given a barcode/QR field, when configured, then it specifies expected symbology and an optional lookup validation against reference data.
- Given a repeating group, when configured, then it supports a min/max occurrence count and per-instance validation.
- Given a calculated field, when its input fields change, then it recomputes client-side in real time and again server-side on submission for integrity.
Priority: MVP | Size: XL | Dependencies: EPIC-01

### EPIC-04.S-02 — Conditional visibility and mandatory logic
As an **Operations Admin**, I want fields/sections to show, hide, or become mandatory based on
other answers, so that forms adapt dynamically and irrelevant questions are skipped.
- Given a visibility rule referencing another field's value, when that value changes, then the dependent field shows/hides immediately, both online and on a cached offline form.
- Given a conditional-mandatory rule (e.g., "corrective action required if temperature > 8°C"), when the trigger condition is met, then submission is blocked until the field is completed.
- Given nested conditional rules, when evaluated, then evaluation order is deterministic and documented (topological by dependency graph).
Priority: MVP | Size: L | Dependencies: EPIC-04.S-01

### EPIC-04.S-03 — Cross-field and cross-record validation
As an **Operations Admin**, I want validation rules spanning multiple fields or referencing other
records (e.g., asset lookup, prior submission), so that data integrity issues are caught at entry,
not after the fact.
- Given a cross-field rule (e.g., end time must be after start time), when violated, then a field-level error is shown before submission is allowed.
- Given a cross-record validation (e.g., asset serial must exist in asset registry), when the lookup fails offline, then the value is accepted provisionally and flagged for server-side re-validation on sync.
Priority: MVP | Size: L | Dependencies: EPIC-04.S-02

### EPIC-04.S-04 — Risk scoring and branching logic
As an **Operations Admin**, I want to configure risk scoring rules and branching paths based on
answers, so that high-risk responses automatically trigger a different downstream path (e.g.,
mandatory corrective action workflow).
- Given a scoring rule set (e.g., Appendix B rule spec pattern), when a submission is evaluated, then a computed risk score and level (LOW/MEDIUM/HIGH/CRITICAL) is attached to the submission.
- Given a HIGH risk score, when computed, then branching logic can route the submission into a mandatory follow-up section or trigger `create_issue` per rule action.
- Given a rule referencing `set_risk`, `require_field`, `create_issue`, `notify_role` actions (Appendix B), when a submission matches the trigger, then all configured actions fire idempotently exactly once.
Priority: MVP | Size: L | Dependencies: EPIC-04.S-02, EPIC-07

### EPIC-04.S-05 — Template versioning and publish lifecycle
As an **Operations Admin**, I want templates to move through
`DRAFT→REVIEW→APPROVED→PUBLISHED→SUPERSEDED→RETIRED` with immutable published versions, so that
historical submissions always reference the exact schema and rules used.
- Given a template is published, when a new version is later published, then the prior `TemplateVersion` becomes `SUPERSEDED` but remains readable and immutable.
- Given a `Submission`, when created, then it stores `template_version_id`, `schema_hash`, rules version, and scoring version at time of submission.
- Given the Version Immutability acceptance test (source-spec §24.1), when a historical submission is rendered after later versions publish, then it renders exactly against the original template version.
- Given an attempt to edit a `PUBLISHED` version directly, then the API rejects it; only a new draft version can be created.
Priority: MVP | Size: L | Dependencies: EPIC-04.S-01

### EPIC-04.S-06 — Impact preview before publish
As an **Operations Admin**, I want to preview affected locations, work items, users, active
schedules, downstream workflows, and reporting dimensions before publishing a template change, so
that I don't break live operations.
- Given a draft version ready to publish, when previewed, then the system lists count and sample of affected active schedules and in-flight work items.
- Given the preview flags a breaking field removal referenced by an active dashboard, then the admin is warned before publish with an explicit override step.
Priority: MVP | Size: M | Dependencies: EPIC-04.S-05

### EPIC-04.S-07 — Simulated execution ("test run") before production publish
As an **Operations Admin**, I want to run a simulated execution of a template with representative
inputs before publishing to production, so that logic errors are caught pre-launch.
- Given a `TEST` or `PILOT` stage template, when representative sample data is submitted, then computed scores, branching, and validation behave exactly as production would, without creating real `WorkItem`s.
- Given a simulation run, when completed, then a report of triggered rules/actions is generated for admin review.
Priority: MVP | Size: M | Dependencies: EPIC-04.S-04, EPIC-04.S-06

### EPIC-04.S-08 — Reference data lookups
As an **Operations Admin**, I want form fields to look up person/location/asset/product reference
data, so that data entry is fast and consistent rather than free text.
- Given a location lookup field, when typed, then it searches within the user's scoped locations only.
- Given an asset lookup, when the asset registry is updated, then subsequent lookups reflect the update without a form republish.
- Given the lookup field is used offline, when no network is available, then it searches the locally cached bounded reference dataset (per source-spec §9.2).
Priority: MVP | Size: M | Dependencies: EPIC-04.S-01, EPIC-05.S-01

### EPIC-04.S-09 — Multi-language form translation
As an **Operations Admin**, I want to provide translations of a template's labels/help text for
each supported locale, so that frontline associates see the form in their preferred language.
- Given a template with translations configured, when a user's device locale matches a supported language, then labels/help text render translated while field keys/schema remain unchanged.
- Given an untranslated new field is added, then it falls back to the default language and is flagged for translation in the builder.
Priority: Phase 2 | Size: M | Dependencies: EPIC-04.S-01

### EPIC-04.S-10 — Submission review and approval
As a **Store/Site Manager**, I want to review a submitted form, approve or reject it with comments,
so that verification is a distinct, evidenced step from mere completion.
- Given a `SUBMITTED` submission requiring review, when approved, then the work item transitions to `APPROVED`/`VERIFIED` per configured verification criteria.
- Given a rejection with required comment, then the work item returns to `IN_PROGRESS` and the original submitter is notified with the reason.
- Given "Verified Completion" (source-spec Appendix D), when a submission requires evidence and/or approval, then `VERIFIED` status is only reachable once those criteria are satisfied — a claim of completion alone is insufficient.
Priority: MVP | Size: M | Dependencies: EPIC-04.S-05, EPIC-03.S-02

### EPIC-04.S-11 — Workflow Builder Agent (NL-to-draft-workflow)
As an **Operations Admin**, I want to describe a workflow in natural language and receive a draft
template/workflow for review, so that building common standards is faster.
- Given a natural-language prompt, when submitted, then the Workflow Builder Agent produces a `DRAFT` template with fields/logic populated for admin review — never auto-published.
- Given the agent's suggestion, when accepted, then the resulting draft is indistinguishable in structure from a manually built one and follows the same versioning lifecycle.
Priority: Phase 2 | Size: L | Dependencies: EPIC-04.S-01, EPIC-11

---

## EPIC-05 — Mobile Offline

**Module:** `04-mobile-offline`
**Goal:** Make offline a first-class, fully supported operating mode for the frontline app — local
caching, capture, queuing, idempotent sync, and conflict handling — so connectivity loss never stops
frontline work.

### EPIC-05.S-01 — Local data cache bootstrap
As a **Frontline Associate**, I want the app to cache my assigned work, permissions, locations,
published templates, and required knowledge locally on login, so that I can work immediately even
before or without a network connection.
- Given first login with connectivity, when bootstrap completes, then WatermelonDB tables (`users, permissions, locations, templates, template_versions, work_items`) are populated per conventions.md's local schema.
- Given a storage quota is configured, when the cache approaches the limit, then least-recently-needed non-pending data is evicted first, never pending submissions/evidence.
- Given the bootstrap fails midway (app killed), when relaunched, then it resumes rather than restarting from zero.
Priority: MVP | Size: L | Dependencies: EPIC-03, EPIC-04, EPIC-02

### EPIC-05.S-02 — Fast app start and resume
As a **Frontline Associate**, I want the app to start fast and resume instantly from background, so
that I can act quickly between customer interactions.
- Given the app is cold-started, when measured, then the home ("Today") screen renders from local cache within the mobile startup performance target (source-spec §23), independent of network availability.
- Given the app resumes from background after a sync completed elsewhere, then the UI reflects updated state without a full reload.
Priority: MVP | Size: M | Dependencies: EPIC-05.S-01

### EPIC-05.S-03 — Camera-first evidence capture
As a **Frontline Associate**, I want to capture photo/video/audio evidence directly within a form
field using a camera-first UI, so that evidence capture is fast, one-tap, and doesn't require
leaving the app.
- Given a photo evidence field, when tapped, then the native camera opens directly (no gallery-first flow) via the RN native camera module (per conventions.md).
- Given a captured photo, when saved, then GPS, timestamp, and device metadata are attached automatically per source-spec §10.1.
- Given low storage on device, when capturing video, then the app warns before capture rather than failing silently mid-capture.
- Given the capture is done offline, then the media is queued locally with a pending-upload state, visible to the user.
Priority: MVP | Size: L | Dependencies: EPIC-05.S-01, EPIC-06

### EPIC-05.S-04 — Barcode / QR scanning
As a **Frontline Associate**, I want to scan a barcode/QR code to populate a field or look up an
asset/product, so that data entry is instant and error-free.
- Given a barcode field, when the scanner opens, then it auto-detects common symbologies (UPC, EAN, Code128, QR) and populates the field on successful scan without manual confirmation for high-confidence reads.
- Given a scan matches a reference-data lookup, when offline, then it resolves against the locally cached bounded reference dataset.
- Given repeated scan failures (poor lighting/damaged label), then a manual entry fallback is offered after 3 failed attempts.
Priority: MVP | Size: M | Dependencies: EPIC-05.S-01, EPIC-04.S-08

### EPIC-05.S-05 — Offline draft submission and local queueing
As a **Frontline Associate**, I want to complete an entire audit/form while fully offline and have
it queue locally for sync, so that connectivity is never a blocker to doing my job.
- Given zero connectivity, when a form is completed and submitted, then it is written to the local `submissions` table with a `client_uuid` and queued in `sync_queue`.
- Given the "Offline submission" acceptance test (source-spec §24.1), when I complete an audit in airplane mode, kill/restart the app, and reconnect, then exactly one server submission is created.
- Given the app is force-closed mid-form, when reopened, then the in-progress draft is restored from local storage with no data loss.
Priority: MVP | Size: L | Dependencies: EPIC-05.S-01, EPIC-04

### EPIC-05.S-06 — Idempotent background sync
As a **Frontline Associate**, I want queued changes to sync automatically in the background when
connectivity returns, so that I never have to manually manage uploads.
- Given queued changes, when connectivity returns, then `POST /api/v1/sync` is called with `device_id`, `sync_token`, and the batched `changes[]` per the Sync Endpoint contract (source-spec §18.2).
- Given a duplicate `client_uuid` is resent (retry after ambiguous network failure), then the server returns `200` with the original server identity and creates no duplicate record (per source-spec §19.4).
- Given large media, when synced, then it uploads separately via resumable upload semantics, decoupled from the submission-metadata sync call.
- Given sync fails, when retried, then exponential backoff with jitter is used, and background execution uses WorkManager/BGTaskScheduler per conventions.md.
- Given a successful sync response, when processed, then the client persists `next_sync_token` as the new `sync_cursor` for the next incremental pull.
Priority: MVP | Size: XL | Dependencies: EPIC-05.S-05

### EPIC-05.S-07 — Conflict detection and resolution UX
As a **Frontline Associate**, I want a clear conflict-resolution UI when my offline edit conflicts
with a server-side change, so that I don't lose work and understand what happened.
- Given the sync response includes a `conflicts[]` entry, when displayed, then the user sees both the local and server values with a choice to keep local, accept server, or merge (field-level, where safe, per conventions.md).
- Given a finalized/immutable submission, when a conflicting edit is attempted, then no merge UI is offered — the server version is authoritative and the local draft is preserved separately for reference.
- Given the Conflict safety acceptance test (source-spec §24.1), when two users concurrently update the same mutable task, then a deterministic version conflict is returned, never a silent overwrite.
Priority: MVP | Size: L | Dependencies: EPIC-05.S-06

### EPIC-05.S-08 — Push and local notifications
As a **Frontline Associate**, I want push notifications for new/urgent assignments and local
notifications for reminders even without connectivity, so that I don't miss time-sensitive work.
- Given a new critical work item assignment, when published server-side, then FCM/APNs push is delivered via the Notification Hub within the platform's delivery SLA.
- Given a scheduled local reminder (e.g., "task due in 15 min"), when the device has no connectivity, then the local notification still fires from device-scheduled state.
- Given notification preferences, when a user opts out of a non-critical category, then only that category is suppressed; SLA-critical/escalation notifications cannot be fully disabled by the user.
Priority: MVP | Size: M | Dependencies: EPIC-05.S-01, EPIC-03.S-07

### EPIC-05.S-09 — One-handed interaction and accessibility
As a **Frontline Associate**, I want core actions (start task, capture evidence, submit) reachable
one-handed and compatible with screen readers/dynamic text, so that the app is usable in real
working conditions and by users with disabilities.
- Given a one-handed reachability review, when primary actions are placed, then they fall within the bottom-two-thirds thumb zone on standard device sizes.
- Given a screen reader is active, when navigating a form, then all interactive elements have accessible labels and logical focus order (WCAG-aligned, per source-spec §23).
- Given dynamic text size is increased at the OS level, when the app renders, then layouts reflow without clipping or overlap.
Priority: MVP | Size: M | Dependencies: EPIC-05.S-01

### EPIC-05.S-10 — App upgrade resilience
As a **Frontline Associate**, I want an app upgrade to never lose my pending offline submissions or
queued evidence, so that a forced update mid-shift doesn't cost me work.
- Given a pending sync queue, when the app is upgraded, then local DB migrations preserve `submissions`, `evidence`, and `sync_queue` tables intact.
- Given a schema-breaking template change ships in a new app version, when an old cached template version is still queued for submission, then it is honored for in-flight submissions and only new work uses the updated schema.
- Given a mandatory-update policy, when the server flags the installed version as unsupported, then the user is blocked from new actions but allowed to complete and sync already-in-progress offline work first, where safe.
Priority: MVP | Size: L | Dependencies: EPIC-05.S-06

### EPIC-05.S-11 — Device-level security controls
As an **Enterprise Admin**, I want mobile builds signed and device/session security enforced (PIN
lock, biometric unlock, remote wipe on deprovision), so that lost/stolen devices don't expose
tenant data.
- Given app-level PIN/biometric lock is enabled by policy, when the app is backgrounded beyond a timeout, then it requires re-authentication to resume.
- Given a device is reported lost, when an admin triggers remote data wipe, then local WatermelonDB contents are purged on next app launch/heartbeat.
- Given mobile builds, when distributed, then they are signed and integrity-verified per source-spec §21.2.
Priority: MVP | Size: M | Dependencies: EPIC-02.S-09

### EPIC-05.S-12 — Sync telemetry and offline-mode instrumentation
As an **Enterprise Admin**, I want offline usage and sync success/failure treated as first-class
telemetry (not error logs), so that offline is measured as a supported mode per source-spec §9.2.
- Given a sync attempt, when completed or failed, then latency, payload size, and outcome are recorded against the Sync Recovery NFR target (source-spec §23: 95% of standard reconnect syncs < 30s).
- Given a dashboard for offline health, when viewed, then it surfaces sync success rate, average queue depth, and time-to-sync distribution.
Priority: MVP | Size: M | Dependencies: EPIC-05.S-06, EPIC-12

---

## EPIC-06 — Evidence

**Module:** `05-evidence-management`
**Goal:** Treat evidence as a first-class, hashed, securely stored, and retention-governed object so
that verified completion is provable and defensible under audit.

### EPIC-06.S-01 — Capture and attach evidence to a submission
As a **Frontline Associate**, I want photo/video/audio/document/signature evidence I capture to be
attached to the exact form field and submission, so that verification has a clear evidentiary trail.
- Given evidence captured in a field, when the submission is saved, then an `Evidence` record is created with `id, object_id, hash, capture metadata` linked to the field and `Submission`.
- Given GPS/timestamp/device metadata/barcode scan results, when captured, then all are stored as evidence metadata per source-spec §10.1.
Priority: MVP | Size: L | Dependencies: EPIC-05.S-03, EPIC-04

### EPIC-06.S-02 — Content hashing and integrity verification
As an **Auditor/Inspector**, I want every evidence file cryptographically hashed at capture, so that
tampering after the fact can be detected.
- Given a captured file, when uploaded, then a SHA-256 hash is computed client-side at capture time and re-verified server-side after upload.
- Given a hash mismatch between client-computed and server-computed values, then the upload is rejected and flagged for investigation.
Priority: MVP | Size: M | Dependencies: EPIC-06.S-01

### EPIC-06.S-03 — Secure, tenant-isolated object storage
As an **Enterprise Admin**, I want evidence stored in per-tenant, KMS-encrypted object storage with
signed-URL access, so that no tenant can ever access another tenant's evidence.
- Given evidence upload, when stored, then the object key is prefixed by `tenant_id` and encrypted at rest via KMS (per conventions.md).
- Given a read request, when authorized, then access is via a short-lived signed URL, never a public/static link.
- Given the Tenant isolation acceptance test (source-spec §24.1), when Tenant A attempts to retrieve Tenant B's evidence file by ID, then it is denied.
Priority: MVP | Size: L | Dependencies: EPIC-06.S-01, EPIC-14

### EPIC-06.S-04 — Resumable large-media upload
As a **Frontline Associate**, I want large video/photo evidence to upload resumably in the
background, so that a dropped connection mid-upload doesn't force a full re-capture or re-upload.
- Given an interrupted upload, when connectivity resumes, then the upload continues from the last successfully received chunk rather than restarting.
- Given an upload completes, then the local pending-evidence record is marked uploaded and removed from the retry queue.
Priority: MVP | Size: L | Dependencies: EPIC-06.S-01, EPIC-05.S-06

### EPIC-06.S-05 — Evidence retention and lifecycle policy
As an **Enterprise Admin**, I want configurable retention policies (active → archived → optional
retention hold → delete/destroy) per tenant/data class, so that evidence handling meets legal and
contractual obligations.
- Given a retention policy of N days, when evidence ages past N days and no legal hold applies, then it transitions `ACTIVE→ARCHIVED` and eventually is eligible for deletion per source-spec §21.4.
- Given a legal hold is applied to evidence linked to an open issue/litigation, then deletion is blocked regardless of the age-based policy.
- Given evidence is deleted per policy, then the `Evidence` metadata row is tombstoned (not silently erased) preserving audit continuity, while the underlying object is destroyed.
Priority: MVP | Size: L | Dependencies: EPIC-06.S-03

### EPIC-06.S-06 — Evidence viewer with annotations
As a **Store/Site Manager**, I want to view evidence (photos with zoom, video playback, signature
image, documents) inline during review, with the ability to annotate/comment, so that approval
decisions are informed and documented.
- Given a photo evidence item, when viewed, then pinch-zoom and rotation are supported without downloading the full-resolution original unnecessarily (progressive/thumbnail loading).
- Given a reviewer adds an annotation/comment on evidence, then it is stored and visible in the submission's audit history.
Priority: MVP | Size: M | Dependencies: EPIC-06.S-01, EPIC-04.S-10

### EPIC-06.S-07 — AI verification result attachment
As an **AI Agent**, I want to attach a structured AI verification result (e.g., Visual AI compliance
score) to a piece of evidence, so that automated analysis is traceable back to the exact evidence it
analyzed.
- Given an evidence item processed by Visual AI, when the result is returned, then it is stored per the Result Contract (source-spec §12.2: `compliance_score, confidence, status, violations[]`) and linked to the `Evidence` record.
- Given the AI result status is `AI_RESULT_UNREVIEWED`, then it is never treated as an irreversible compliance fact until human review policy resolves it (see EPIC-10).
Priority: Phase 2 | Size: M | Dependencies: EPIC-06.S-01, EPIC-10

### EPIC-06.S-08 — Evidence export and chain-of-custody report
As an **Auditor/Inspector**, I want to export a bundle of evidence with a chain-of-custody report
(who captured, when, hash, device, any subsequent access), so that I can produce defensible audit
packages for regulators or customers.
- Given an export request for a submission's evidence, when generated, then it includes original files, hashes, capture metadata, and an access log.
- Given the export is downloaded, then the export event itself is recorded in `AuditLog` with actor and timestamp.
Priority: Phase 2 | Size: M | Dependencies: EPIC-06.S-02, EPIC-14

### EPIC-06.S-09 — Signature capture as evidence
As a **Frontline Associate**, I want to capture a signature (mine or a customer's) directly in a
form, so that sign-off evidence is captured without paper.
- Given a signature field, when signed on-screen, then the signature is rasterized to an image, hashed, and stored as `Evidence` with capturer identity and timestamp.
- Given the signature field is required for submission, then submission is blocked until a signature is present.
Priority: MVP | Size: S | Dependencies: EPIC-06.S-01, EPIC-04.S-01

---

## EPIC-07 — Issues & CAPA

**Module:** `06-issues-capa`
**Goal:** Manage the full exception lifecycle from detection through root-cause analysis, corrective
and preventive action, and effectiveness verification, connecting deviations to durable fixes.

### EPIC-07.S-01 — Create an issue (manual or rule-triggered)
As a **Store/Site Manager**, I want to open an issue manually or have one auto-created by a form
rule (e.g., temperature exception), so that every deviation gets a governed record.
- Given a form rule action `create_issue: true` fires (Appendix B pattern), when evaluated, then an `Issue` is created in `OPEN` status with `severity, source, owner, SLA` populated and linked back to the triggering `Submission`.
- Given a manually created issue, when saved, then it requires at minimum a severity and description before leaving `OPEN`.
- Given an issue is created, then a canonical `issue.created` event is published per source-spec §18.3.
Priority: MVP | Size: M | Dependencies: EPIC-04.S-04, EPIC-03

### EPIC-07.S-02 — Issue lifecycle state machine
As a **Regional/District Manager**, I want the issue to move through
`OPEN→TRIAGED→ASSIGNED→INVESTIGATING→CORRECTIVE_ACTION→VERIFICATION→RESOLVED→CLOSED` with a
`REOPENED` path on failed verification, so that resolution is rigorous and repeat failures are
caught.
- Given an issue in `VERIFICATION`, when the effectiveness check fails, then it transitions to `REOPENED→INVESTIGATING` automatically, per conventions.md state machine.
- Given an illegal transition attempt, then the API rejects it with the current valid next states.
- Given an issue is `CLOSED`, when a later related failure occurs, then analytics can link it as a "repeat failure" against the closed issue for the Repeat Failure Rate KPI (source-spec §14.2).
Priority: MVP | Size: L | Dependencies: EPIC-07.S-01

### EPIC-07.S-03 — Triage and assignment
As a **Store/Site Manager**, I want to triage a new issue (confirm severity, assign owner) so that
accountability is established quickly.
- Given an `OPEN` issue, when triaged, then it moves to `TRIAGED` with a confirmed/adjusted severity.
- Given triage assigns an owner, then it moves to `ASSIGNED` and the owner is notified with SLA countdown.
- Given no triage occurs within the configured SLA window, then the issue auto-escalates per EPIC-03.S-07's escalation ladder mechanism.
Priority: MVP | Size: M | Dependencies: EPIC-07.S-02, EPIC-03.S-07

### EPIC-07.S-04 — Immediate correction and containment
As a **Frontline Associate**, I want to record the immediate correction and containment action taken
at the moment an issue is found, so that risk is controlled before full root-cause analysis.
- Given an issue moves to `INVESTIGATING`, when an immediate correction is logged, then it is stored as a distinct CAPA sub-record (`immediate_correction`) separate from the eventual corrective action.
- Given containment is marked complete, then the issue can proceed to root cause without blocking on full resolution.
Priority: MVP | Size: M | Dependencies: EPIC-07.S-02

### EPIC-07.S-05 — Root cause analysis (5 Whys, Fishbone, Pareto, FMEA, custom)
As a **Store/Site Manager**, I want to run a structured root cause analysis using 5 Whys, Fishbone,
Pareto, FMEA, or a custom enterprise method, so that the fix addresses the actual cause, not just
the symptom.
- Given the RCA method selection, when 5 Whys is chosen, then the UI guides sequential "why" capture and stores the causal chain.
- Given Fishbone is chosen, then categories (People/Process/Equipment/Environment/etc.) are captured with contributing factors per category.
- Given a custom enterprise method is configured by an admin, then it appears as a selectable RCA method with its own structured fields.
- Given RCA is completed, then the issue's `INVESTIGATING` stage cannot advance to `CORRECTIVE_ACTION` without a recorded root cause (configurable policy).
Priority: MVP | Size: L | Dependencies: EPIC-07.S-04

### EPIC-07.S-06 — Corrective and preventive action (CAPA) plan
As a **Regional/District Manager**, I want to define both corrective (fix this instance) and
preventive (stop recurrence) actions with owners and due dates, so that the same failure doesn't
keep happening.
- Given a CAPA plan, when saved, then it stores discrete `corrective_action` and `preventive_action` records, each with owner, due date, and status.
- Given a preventive action creates a new standing `WorkItem` or template change, then it links back to the originating `Issue` for traceability.
Priority: MVP | Size: L | Dependencies: EPIC-07.S-05, EPIC-03

### EPIC-07.S-07 — Effectiveness verification and re-inspection
As an **Auditor/Inspector**, I want to schedule and record a verification/re-inspection to confirm
the corrective action actually worked, so that resolution isn't just self-reported.
- Given a CAPA plan is complete, when a verification work item is scheduled and submitted, then the issue moves `CORRECTIVE_ACTION→VERIFICATION`.
- Given the verification submission passes acceptance criteria, then the issue moves to `RESOLVED`.
- Given the verification fails, then the issue moves to `REOPENED→INVESTIGATING` with the failure evidence attached.
Priority: MVP | Size: L | Dependencies: EPIC-07.S-02, EPIC-06

### EPIC-07.S-08 — Issue closure and sign-off
As a **Store/Site Manager**, I want to formally close a resolved issue with a sign-off, so that
there is a clear, auditable end to the exception lifecycle.
- Given an issue in `RESOLVED`, when closed, then it requires a closing comment and moves to `CLOSED`, becoming immutable except for authorized reopen.
- Given a `CLOSED` issue, when an authorized user reopens it (e.g., new related evidence surfaces), then it transitions to `REOPENED` with a mandatory reason.
Priority: MVP | Size: S | Dependencies: EPIC-07.S-07

### EPIC-07.S-09 — Issue dashboard and queue for managers
As a **Store/Site Manager**, I want a filterable issue queue (by severity, status, SLA remaining,
location), so that I can prioritize my team's exception-handling work.
- Given the queue, when filtered by "overdue SLA", then only issues past their SLA threshold display, sorted by time overdue.
- Given an issue row, when clicked, then the manager can triage/assign/comment directly without navigating away.
Priority: MVP | Size: M | Dependencies: EPIC-07.S-02, EPIC-12

### EPIC-07.S-10 — Repeat failure pattern detection
As a **Regional/District Manager**, I want the system to flag when a similar issue recurs at the
same or related locations, so that I can escalate systemic problems rather than treating each one in
isolation.
- Given a new issue matches prior closed issues by category/location/template within a lookback window, when created, then it is flagged as a "repeat failure" and linked to prior occurrences.
- Given 3+ repeat occurrences within a rolling window, then the issue severity is auto-elevated per configurable policy and a Compliance Monitor Agent recommendation may be generated (see EPIC-11).
Priority: Phase 2 | Size: M | Dependencies: EPIC-07.S-02, EPIC-15

### EPIC-07.S-11 — CAPA linkage to training
As a **Store/Site Manager**, I want a CAPA preventive action to be able to directly assign
microlearning to the responsible associate(s), so that skill/knowledge gaps are closed as part of
resolution.
- Given a preventive action of type "training", when saved, then a `LearningAssignment` is created and linked to the `Issue`.
- Given the training is completed and a knowledge check passed, then the linked issue's preventive-action status updates to complete automatically.
Priority: Phase 2 | Size: M | Dependencies: EPIC-07.S-06, EPIC-09

---

## EPIC-08 — Comms & Knowledge

**Module:** `07-communications-knowledge`
**Goal:** Deliver targeted, acknowledgeable communications and an authorization-aware Knowledge Hub
with search, so the right information reaches the right audience and can be proven read.

### EPIC-08.S-01 — Compose and target an announcement
As an **Operations Admin**, I want to compose an announcement and target it to specific roles,
locations, or organizational segments, so that only the relevant audience receives it.
- Given a targeting rule (role + location subtree + brand tag), when previewed, then an audience preview shows the resolved recipient count and sample list before publish.
- Given the announcement is published, then it is delivered via in-app feed, and optionally push/email/SMS per configured channels (source-spec §11.1).
Priority: MVP | Size: L | Dependencies: EPIC-02, EPIC-01

### EPIC-08.S-02 — Mandatory acknowledgement with read receipts
As an **Operations Admin**, I want to require acknowledgement of critical announcements and track
read receipts, so that I can prove policy communication compliance.
- Given an announcement marked "mandatory acknowledgement," when a targeted user opens it, then they must tap "Acknowledge" before it is dismissible, and this creates a `message.acknowledged` event.
- Given a compliance report, when viewed, then acknowledgement rate is shown per location/role with drill-down to non-acknowledgers.
- Given a user has not acknowledged within a configured window, then a reminder notification is sent, and non-acknowledgement can optionally escalate to their manager.
Priority: MVP | Size: M | Dependencies: EPIC-08.S-01

### EPIC-08.S-03 — Scheduled publication and expiry
As an **Operations Admin**, I want to schedule an announcement's publish time and set an expiry, so
that time-sensitive content appears and disappears automatically.
- Given a future `publish_at`, when that time arrives, then the announcement becomes visible without manual action.
- Given an `expires_at` is reached, then the announcement is removed from active feeds but remains in the historical audit trail.
Priority: MVP | Size: S | Dependencies: EPIC-08.S-01

### EPIC-08.S-04 — Announcement translation
As a **Frontline Associate**, I want to read announcements in my preferred language, so that
critical information is understood correctly.
- Given a message with translations configured or machine-translated, when viewed, then it renders in the user's locale with an indicator if machine-translated.
- Given a machine translation is used, then the original-language source remains one tap away.
Priority: Phase 2 | Size: M | Dependencies: EPIC-08.S-01

### EPIC-08.S-05 — Multi-channel delivery (push, email, SMS/enterprise messaging)
As an **Enterprise Admin**, I want announcements to optionally deliver via push, email, and
SMS/Teams/Slack-class enterprise messaging, so that reach matches how each workforce actually
communicates.
- Given a channel is enabled per tenant, when an announcement publishes, then delivery attempts are made per user's channel preference and delivery status is tracked per channel (per Notification Hub responsibilities, source-spec §16.3).
- Given a channel delivery fails, then it retries per the Notification Hub's retry policy and falls back to in-app feed as the guaranteed channel.
Priority: MVP (in-app/push) / Phase 2 (enhanced SMS/enterprise messaging) | Size: L | Dependencies: EPIC-08.S-01, EPIC-13

### EPIC-08.S-06 — Knowledge document authoring and versioning
As an **Operations Admin**, I want to author Knowledge Hub documents with versions, effective dates,
audience scope, language, and ownership, so that approved knowledge is governed like any other
operational standard.
- Given a `KnowledgeDocument` is published, when a new version is later published, then the prior version is retained with its own effective date range.
- Given an audience scope is set, then only users within that scope can view the document, enforced identically to direct-access authorization.
Priority: Phase 2 (Basic in MVP per Capability Map) | Size: L | Dependencies: EPIC-02, EPIC-01

### EPIC-08.S-07 — Knowledge search with authorization enforcement
As a **Frontline Associate**, I want to search the Knowledge Hub and get results scoped to what I'm
authorized to see, so that search never leaks restricted content.
- Given a search query, when executed, then results are filtered server-side by the same authorization model as direct document access (no over-fetch-then-filter client-side).
- Given the Knowledge answer/source success KPI, then each result links to the exact document version and shows its effective date.
- Given OpenSearch is the backing index (per conventions.md), when a document is updated, then the index reflects the change within a bounded freshness SLA.
Priority: Phase 2 | Size: L | Dependencies: EPIC-08.S-06

### EPIC-08.S-08 — AI-assisted knowledge answers (Operations Copilot input)
As a **Frontline Associate**, I want to ask a natural-language question and get an answer grounded
in approved Knowledge Hub documents with citations, so that I get a fast, trustworthy answer instead
of searching manually.
- Given a question, when answered by the Operations Copilot agent, then the response cites source document(s) and version, and never surfaces content the user isn't authorized to see (no cross-tenant/cross-scope retrieval).
- Given no sufficiently confident grounded answer exists, then the agent declines and offers to route to a human/search instead of hallucinating.
Priority: Phase 2 | Size: L | Dependencies: EPIC-08.S-07, EPIC-11

### EPIC-08.S-09 — Communication Agent draft assistance
As an **Operations Admin**, I want an agent to draft a targeted communication from a prompt or
triggering event (e.g., repeated failure pattern), so that communications go out faster with
consistent tone.
- Given a triggering event and target audience, when the Communication Agent runs, then it produces a draft announcement for human review — never auto-publishes (L1 Recommend autonomy per source-spec §20.3).
- Given the admin edits and approves the draft, then it publishes through the normal EPIC-08.S-01 pipeline with the agent recorded as originator in `AgentRun`.
Priority: Phase 2 | Size: M | Dependencies: EPIC-08.S-01, EPIC-11

### EPIC-08.S-10 — Document access analytics
As an **Operations Admin**, I want to see which knowledge documents are viewed, searched for but not
found, or never accessed, so that I can improve content coverage and quality.
- Given a reporting period, when viewed, then top search queries with no satisfactory result are surfaced as content gaps.
- Given a document has zero views in 90 days, then it is flagged for review/retirement.
Priority: Phase 2 | Size: M | Dependencies: EPIC-08.S-07, EPIC-15

---

## EPIC-09 — Learning

**Module:** `08-learning`
**Goal:** Link training to operational outcomes so that repeated failures trigger targeted
microlearning, verified by knowledge checks and re-inspection, closing the loop from failure to
measured improvement.

### EPIC-09.S-01 — Course and content authoring
As an **Operations Admin**, I want to author or upload a course (video, document, slide-based
microlearning) with a linked knowledge check, so that training content is structured and
assessable.
- Given a course is created, then it stores content reference, estimated duration, and an optional linked quiz/knowledge check.
- Given a course is published, then it follows a version lifecycle so historical completions remain attributable to the exact version taken.
Priority: Phase 2 (Basic in MVP per Capability Map) | Size: L | Dependencies: EPIC-01, EPIC-02

### EPIC-09.S-02 — Assign training to a user, role, or location
As a **Store/Site Manager**, I want to assign a course to an individual, a role, or all associates
at a location, so that training reaches the right people.
- Given a role/location-based assignment, when a new user joins that role/location, then the course is automatically assigned to them (standing assignment).
- Given an assignment is created, then a `LearningAssignment` record and `training.assigned` event are generated per source-spec §18.3.
Priority: Phase 2 (Basic in MVP) | Size: M | Dependencies: EPIC-09.S-01, EPIC-02.S-05

### EPIC-09.S-03 — Complete training and knowledge check
As a **Frontline Associate**, I want to complete assigned training and its knowledge check on
mobile, including offline where content was pre-cached, so that I can train during downtime without
needing connectivity.
- Given a course is pre-cached for offline availability, when completed offline, then completion and quiz results queue for sync like any other submission (EPIC-05.S-06 semantics).
- Given a knowledge check score below the passing threshold, then the course is marked incomplete and can be retaken per policy (e.g., max attempts, cooldown).
- Given completion, then `training.completed` event is published and the `LearningAssignment` status updates.
Priority: Phase 2 (Basic in MVP) | Size: L | Dependencies: EPIC-09.S-02, EPIC-05

### EPIC-09.S-04 — Certification tracking and expiry
As a **Regional/District Manager**, I want to track certifications (e.g., food safety) with expiry
dates and automatic renewal reminders, so that compliance lapses are caught before they become
audit findings.
- Given a certification nearing expiry, when the reminder threshold is reached, then the holder and their manager are notified.
- Given a certification expires, when a work item requires it (ABAC condition per EPIC-02.S-06), then the associate is blocked from that action until renewed.
Priority: Phase 2 | Size: M | Dependencies: EPIC-09.S-03, EPIC-02.S-06

### EPIC-09.S-05 — Failure-triggered microlearning assignment
As a **Compliance Monitor** (**AI Agent**), I want a repeated operational failure to automatically
trigger a targeted microlearning assignment to the responsible associate(s), so that the skill gap
is addressed, not just the symptom.
- Given the loop `REPEATED FAILURE→IDENTIFY GAP→ASSIGN MICRO-LEARNING→KNOWLEDGE CHECK→RE-INSPECTION→MEASURE IMPROVEMENT` (source-spec §11.3), when a repeat-failure pattern is detected (EPIC-07.S-10), then the agent proposes a mapped microlearning assignment (L1 Recommend; a manager approves before assignment unless tenant policy grants L2).
- Given the assignment is approved/executed, then it links back to the originating `Issue` for outcome traceability.
Priority: Phase 2 | Size: L | Dependencies: EPIC-07.S-10, EPIC-09.S-02, EPIC-11

### EPIC-09.S-06 — Re-inspection after training
As an **Auditor/Inspector**, I want a re-inspection automatically scheduled after failure-triggered
training completes, so that improvement is verified, not assumed.
- Given the linked knowledge check passes, then a re-inspection `WorkItem` is auto-scheduled at the affected location/associate.
- Given the re-inspection is submitted, then its result is linked to the original issue and training assignment for the outcome-improvement measurement.
Priority: Phase 2 | Size: M | Dependencies: EPIC-09.S-05, EPIC-07.S-07

### EPIC-09.S-07 — Training-to-outcome measurement
As an **Executive**, I want to see whether training interventions actually improved subsequent
performance (e.g., reduced repeat failures), so that we can measure Training Agent effectiveness and
overall program ROI.
- Given a set of completed training-linked re-inspections, when aggregated, then a before/after failure-rate comparison is shown per training program.
- Given the Training-to-outcome improvement success signal (source-spec §5), then this metric is available at course, location, and enterprise level.
Priority: Phase 2 | Size: M | Dependencies: EPIC-09.S-06, EPIC-15

### EPIC-09.S-08 — Learning dashboard for managers
As a **Store/Site Manager**, I want to see my team's training completion status and overdue
assignments, so that I can follow up before compliance deadlines.
- Given the dashboard, when filtered by "overdue," then it lists associates with incomplete required training past due date.
- Given a manager nudges an associate directly from the dashboard, then a reminder notification is sent.
Priority: Phase 2 | Size: M | Dependencies: EPIC-09.S-02, EPIC-12

---

## EPIC-10 — Visual AI

**Module:** `09-visual-ai`
**Goal:** Provide an image-analysis pipeline that produces a structured, confidence-scored result
with bounding boxes and a configurable human-review workflow, so visual compliance checks scale
without becoming an unreviewed black box.

### EPIC-10.S-01 — Submit evidence image to Visual AI pipeline
As a **Frontline Associate**, I want a photo I capture for a compliance check (e.g., planogram) to
be automatically analyzed, so that I get immediate feedback on likely issues before leaving the
location.
- Given an evidence photo tagged for a Visual AI use case (planogram, signage, brand execution,
  cleanliness, PPE/safety, asset condition, vehicle/site condition — source-spec §12.1), when
  uploaded, then it is queued to the Visual Inspection pipeline automatically.
- Given the device is offline at capture time, then the image queues for analysis once uploaded during sync, without blocking submission.
Priority: MVP (Pilot per Capability Map) | Size: L | Dependencies: EPIC-06, EPIC-05

### EPIC-10.S-02 — Structured result contract with bounding boxes
As an **AI Agent** (Visual Inspection Agent), I want to return a structured result with
`compliance_score, confidence, status, violations[]` including bounding boxes, so that downstream
systems and reviewers can act on precise, structured findings.
- Given an analysis completes, then the result matches the contract in source-spec §12.2 exactly, including `bounding_box` coordinates per violation.
- Given the result, then `status` defaults to `AI_RESULT_UNREVIEWED` and is never auto-promoted to a final compliance fact without passing the tenant's review policy.
Priority: MVP (Pilot) | Size: L | Dependencies: EPIC-10.S-01

### EPIC-10.S-03 — Confidence-threshold-based human review routing
As an **Operations Admin**, I want to configure a confidence threshold per use case below which
results require human review, so that low-confidence AI output never silently becomes a compliance
fact.
- Given a result confidence below the configured threshold, when produced, then it enters a "pending human review" queue instead of auto-finalizing.
- Given the Visual AI review acceptance test (source-spec §24.1), when a low-confidence result is produced, then it enters the configured human-review state rather than auto-failing the location.
- Given a reviewer confirms/overrides the result, then the final status and reviewer identity are recorded for audit and model evaluation.
Priority: MVP (Pilot) | Size: M | Dependencies: EPIC-10.S-02

### EPIC-10.S-04 — Bounding-box review UI
As a **Store/Site Manager**, I want to view detected violations overlaid as bounding boxes on the
original image and confirm/reject each, so that review is fast and precise rather than re-inspecting
the whole image.
- Given a result with violations, when the review screen opens, then each bounding box is rendered on the image with type, severity, and confidence.
- Given a reviewer rejects a specific violation, then only that violation is marked overridden; others remain unaffected.
Priority: MVP (Pilot) | Size: M | Dependencies: EPIC-10.S-03

### EPIC-10.S-05 — Appeals and dispute workflow
As a **Frontline Associate**, I want to appeal a Visual AI finding I believe is incorrect, so that
disputed findings get human adjudication before affecting my location's compliance record.
- Given an appeal is filed, when submitted, then the finding's status changes to "under appeal" and is excluded from finalized compliance scoring until adjudicated.
- Given an adjudicator resolves the appeal, then the outcome (upheld/overturned) is recorded and used as model-evaluation feedback.
Priority: Phase 2 | Size: M | Dependencies: EPIC-10.S-04

### EPIC-10.S-06 — Model/version traceability
As an **Enterprise Admin**, I want every Visual AI result to record which model/version produced it,
so that I can audit accuracy over time and manage model changes responsibly.
- Given a result is produced, then it stores model provider, model version, and prompt/pipeline version.
- Given a model version is upgraded, when historical results are viewed, then they still display the model version that actually produced them (immutable, per P7 Historical truth).
Priority: MVP (Pilot) | Size: S | Dependencies: EPIC-10.S-02

### EPIC-10.S-07 — Per-tenant use-case and action policy configuration
As an **Enterprise Admin**, I want to configure per-tenant which Visual AI use cases are enabled and
what action each result status triggers (e.g., auto-create issue on HIGH severity + confirmed), so
that automation matches our risk tolerance.
- Given a tenant disables a use case (e.g., PPE/safety) entirely, then no images are routed to that model for this tenant.
- Given a confirmed HIGH-severity violation and tenant policy `create_issue: true`, then an `Issue` is auto-created linked to the evidence and result.
Priority: MVP (Pilot) | Size: M | Dependencies: EPIC-10.S-03, EPIC-07

### EPIC-10.S-08 — Visual AI accuracy dashboard (precision/recall)
As an **Enterprise Admin**, I want to see precision/recall and override rate per use case and
location, so that I can judge whether a Visual AI model is ready to move from Pilot to Advanced
autonomy.
- Given confirmed vs. overridden results over a period, when aggregated, then precision/recall and human-override rate are computed per use case, matching the source-spec §5 success signal.
- Given accuracy drops below a configured threshold, then an alert notifies the AI governance owner and the use case can be auto-suspended per kill-switch policy (EPIC-11.S-08).
Priority: Phase 2 | Size: M | Dependencies: EPIC-10.S-06, EPIC-15

### EPIC-10.S-09 — Advanced multi-frame / video visual inspection
As an **Operations Admin**, I want Visual AI to analyze short video/multi-angle capture (e.g.,
walking a shelf aisle), so that compliance checks aren't limited to single static photos.
- Given a video/multi-frame capture, when processed, then violations are aggregated across frames with per-frame bounding boxes retained for review.
- Given duplicate detections of the same violation across frames, then they are de-duplicated in the summarized result.
Priority: Phase 3 | Size: XL | Dependencies: EPIC-10.S-02, EPIC-10.S-08

---

## EPIC-11 — Agent Runtime

**Module:** `10-agentic-operations`
**Goal:** Provide a governed, provider-agnostic agent runtime — context, policy, typed tools,
approvals, evaluation, and full audit — so AI can move safely from recommendation to controlled
autonomy without silently overriding enterprise policy.

### EPIC-11.S-01 — Context Builder for agent runs
As an **AI Agent**, I want a Context Builder that assembles only authorized tenant/location/user/
work context for my run, so that I never see or reason over data outside my scope.
- Given an agent run is triggered, when context is assembled, then it is filtered by the invoking event's tenant and location scope before being passed to the model (no post-hoc filtering of raw query results).
- Given a context hash, when the run is recorded, then `input_context_hash` is stored on the `AgentRun` for reproducibility/audit per source-spec §20.4.
Priority: Phase 2 | Size: L | Dependencies: EPIC-01, EPIC-02

### EPIC-11.S-02 — Policy engine (OPA/Rego) for agent actions
As an **Enterprise Admin**, I want a declarative policy engine to gate every agent tool call against
tenant + agent-level policy, so that deterministic rules always take precedence over model output
(P5 — Policy before AI).
- Given a policy defines `allowed_tools`, `blocked_tools`, and `approval_required_for` conditions (Appendix C pattern), when an agent attempts a tool call, then the policy engine evaluates it before execution.
- Given the AI policy enforcement acceptance test (source-spec §24.1), when an agent attempts to call a prohibited tool, then the call is blocked, logged, and surfaced — never silently dropped.
- Given a policy change, then it is versioned and testable via a policy simulation endpoint before deployment.
Priority: Phase 2 | Size: L | Dependencies: EPIC-11.S-01

### EPIC-11.S-03 — Typed, allowlisted tool contracts
As an **AI Agent**, I want access only to typed, allowlisted tools (`get_submission`,
`get_location`, `get_recent_failures`, `get_active_policy`, `search_knowledge`, `create_task`,
`assign_task`, `create_issue`, `send_notification`, `request_approval`, `schedule_training`,
`get_task_status`, `schedule_reinspection`), so that I cannot perform unrestricted database access
or arbitrary actions.
- Given a tool call, when executed, then it is validated against a typed schema (args and return shape) before and after execution.
- Given a tool is not in the agent's allowlist, then the call is rejected at the runtime layer before reaching the tool implementation.
- Given a tool execution, then `AgentToolExecution{run_id, tool, args_hash, result_hash, outcome}` is recorded per the domain model.
Priority: Phase 2 | Size: L | Dependencies: EPIC-11.S-02

### EPIC-11.S-04 — Autonomy-level enforcement (L0–L4)
As an **Enterprise Admin**, I want each agent/tool pairing to carry an explicit autonomy level
(L0 Observe … L4 Prohibited per source-spec §20.3), so that risk of automated action matches
organizational trust and maturity.
- Given a tool/action classified L2 (execute low-risk), when invoked within policy, then it executes without human approval and is logged.
- Given an action classified L3 (execute with approval) or where `approval_required_for` matches (e.g., `severity=CRITICAL`), then execution pauses pending human approval (see EPIC-11.S-06).
- Given an action classified L4 (prohibited), then it can never execute regardless of policy configuration — enforced at the runtime layer, not just policy config.
Priority: Phase 2 (L0-L2) / Phase 3 (L3 controlled autonomy expansion) | Size: L | Dependencies: EPIC-11.S-02, EPIC-11.S-03

### EPIC-11.S-05 — Agent recommendation surface (L1 Recommend)
As a **Store/Site Manager**, I want to see agent-generated recommendations (e.g., Resolution Agent's
proposed corrective task) with rationale, confidence, and evidence, so that I can decide quickly
with full context.
- Given a recommendation, when displayed, then it shows the policy basis, confidence, impacted objects, and source evidence per source-spec §13.4.
- Given the manager acts, then they may approve, reject, or modify the recommendation, and the choice is recorded for the AI Override Rate KPI (source-spec §14.2).
Priority: Phase 2 | Size: M | Dependencies: EPIC-11.S-04

### EPIC-11.S-06 — Human-in-the-loop approval inbox
As a **Regional/District Manager**, I want a dedicated approval inbox for high-risk agent-proposed
actions, so that I can review evidence and policy basis and approve/reject/modify in one place.
- Given a pending approval, when opened, then it shows recommendation, confidence, policy basis, impacted objects, and proposed action per source-spec §13.4.
- Given the manager requests more evidence, then the agent run pauses and, if capable, gathers additional context (e.g., re-runs `get_recent_failures` with a wider window) before re-presenting.
- Given approval or rejection, then `agent.action_approved`/`agent.action_rejected` events are published and `human_approval` is recorded on the `AgentRun`.
Priority: Phase 2 | Size: L | Dependencies: EPIC-11.S-04, EPIC-11.S-05

### EPIC-11.S-07 — Full agent run audit trail
As an **Enterprise Admin**, I want every agent run to record the complete trace (agent version,
model, prompt version, retrieved sources, tools called, arguments/results hashes, decision,
confidence, policy decision, human approval, execution result, latency, cost, trace_id), so that
every autonomous action is fully explainable and auditable.
- Given a completed agent run, when inspected, then all fields of the Agent Run Record (source-spec §20.4) are populated and immutable.
- Given the AI governance NFR (source-spec §23: "every autonomous action traceable to policy and evidence"), when audited, then the trace alone is sufficient to reconstruct the decision without external context.
Priority: Phase 2 | Size: M | Dependencies: EPIC-11.S-01 through EPIC-11.S-06

### EPIC-11.S-08 — Kill switch (tenant/agent/tool-level)
As an **Enterprise Admin**, I want to instantly disable an agent, a specific tool, or all AI features
for my tenant, so that I can respond to an incident or governance concern immediately.
- Given a kill switch is activated at the tool level, when any agent attempts that tool, then it is blocked immediately without a deploy, and in-flight runs fail gracefully with a logged reason.
- Given a tenant-level AI disablement, then no agent runs are triggered for that tenant across all agents until re-enabled.
Priority: Phase 2 | Size: M | Dependencies: EPIC-11.S-02

### EPIC-11.S-09 — Agent evaluation and offline testing pipeline
As an **Enterprise Admin** (via AI Governance), I want agents evaluated on precision/recall, false
escalation rate, missed critical event rate, tool selection correctness, policy adherence,
unauthorized action block rate, human override rate, outcome improvement, latency, and cost, so that
promotion from Recommend to Execute autonomy is evidence-based.
- Given an agent version, when evaluated offline against a labeled dataset, then all metrics in source-spec §20.5 are computed and stored per version.
- Given the Release Path for AI (source-spec §24.3: OFFLINE EVALUATION→SHADOW→RECOMMENDATION→APPROVAL MODE→CONTROLLED AUTONOMY), then an agent cannot advance a stage without meeting the configured evaluation gate for the prior stage.
Priority: Phase 2 | Size: L | Dependencies: EPIC-11.S-07

### EPIC-11.S-10 — Domain-specific agents (Compliance Monitor, Resolution, Audit Intelligence, Executive Insights)
As a **Compliance Monitor** (**AI Agent**), I want to detect recurring or contextual operational
failures and hand off structured findings to the Resolution, Training, and Communication agents, so
that the multi-agent portfolio (source-spec §13.1) operates as one coherent loop rather than isolated
bots.
- Given a detected recurring failure, when the Compliance Monitor runs, then it produces a finding referencing prior `Issue`s (via `get_recent_failures`) and hands off to the Resolution Agent for a proposed corrective task.
- Given each agent in the portfolio, when invoked, then it operates strictly within its declared responsibility and default autonomy (Recommend, per source-spec §13.1) unless tenant policy has explicitly raised it.
Priority: Phase 2 | Size: XL | Dependencies: EPIC-11.S-01 through EPIC-11.S-06, EPIC-07, EPIC-08, EPIC-09, EPIC-10

### EPIC-11.S-11 — Agent memory scoping and retention policy
As an **Enterprise Admin**, I want explicit, configurable retention/scoping policy for any agent
memory, so that agents don't retain sensitive context indefinitely or across tenant/scope
boundaries.
- Given a memory retention policy (e.g., 30-day rolling context), when exceeded, then stored agent memory entries are purged automatically.
- Given cross-tenant retrieval is attempted by any agent context or memory lookup, then it is blocked and logged as a policy violation (source-spec §21.3: "No cross-tenant retrieval").
Priority: Phase 2 | Size: M | Dependencies: EPIC-11.S-01

---

## EPIC-12 — Control Tower

**Module:** `11-analytics-control-tower`
**Goal:** Deliver role-specific operational, regional, and executive dashboards that answer "what is
wrong, where, how serious, why, what's been done, what needs a human, and is it working."

### EPIC-12.S-01 — Store/Site Manager operational dashboard
As a **Store/Site Manager**, I want a real-time view of my location's exceptions, approvals, team
status, and health, so that I can run daily operations effectively.
- Given the dashboard loads, when rendered, then it reflects near-real-time state (bounded staleness) for open exceptions and pending approvals.
- Given a location health score, when computed, then it factors verified completion rate, overdue rate, and open critical issues.
Priority: MVP | Size: L | Dependencies: EPIC-03, EPIC-07

### EPIC-12.S-02 — Regional/District Manager territory view
As a **Regional/District Manager**, I want a view across all locations in my territory showing
trends, at-risk locations, and repeated failure patterns, so that I can direct attention where it
matters most.
- Given a territory of N locations, when viewed, then locations are ranked/heat-mapped by risk (e.g., open critical issues, repeat failures, overdue rate).
- Given a location is flagged "at risk," when drilled into, then the manager sees the specific contributing exceptions and trend direction.
Priority: MVP | Size: L | Dependencies: EPIC-12.S-01, EPIC-15

### EPIC-12.S-03 — Executive Control Tower
As an **Executive**, I want an enterprise-wide view answering what's wrong, where, how serious, why,
what's been done, what needs intervention, and whether it's working, so that I can govern risk at a
glance.
- Given the executive view, when loaded, then it shows enterprise verified compliance %, week-over-week trend, count of critical exceptions, locations at risk, repeated failure patterns, and unacknowledged critical issues — matching the source-spec §14.3 layout.
- Given a metric is clicked, then it drills down through region → location → specific issue without leaving the control tower context.
Priority: MVP | Size: L | Dependencies: EPIC-12.S-02, EPIC-15

### EPIC-12.S-04 — KPI computation and definitions
As an **Enterprise Admin**, I want the platform's canonical KPI set (Verified Operational Execution
Rate, On-Time Completion Rate, Repeat Failure Rate, Exception Resolution Time, Time to Intervention,
Automation Rate, Frontline Adoption, AI Override Rate) computed consistently, so that every
dashboard agrees on definitions.
- Given each KPI, when computed, then it uses the exact definition from source-spec §14.2 (e.g., Verified Operational Execution Rate = verified work completed correctly / work expected).
- Given a KPI is segmented (by location, workflow type, risk, evidence type, verification method, per the North-Star Metric definition in source-spec §26.1), then all segmentation dimensions are queryable.
Priority: MVP | Size: L | Dependencies: EPIC-15

### EPIC-12.S-05 — Exception explainability drill-down
As a **Regional/District Manager**, I want to click into any flagged exception and see why it was
flagged, its evidence, its policy basis, and current status, so that I trust and can act on the
control tower's signal.
- Given an exception, when opened, then it shows the triggering rule/condition, linked evidence, current issue lifecycle stage, and any agent recommendations tied to it.
- Given the exception was AI-flagged (e.g., Visual AI or Compliance Monitor), then model/version and confidence are shown per P8 Explainability.
Priority: MVP | Size: M | Dependencies: EPIC-12.S-01, EPIC-07

### EPIC-12.S-06 — Custom dashboard and report builder
As an **Operations Admin**, I want to build a custom dashboard/report from available metrics and
dimensions, so that I'm not limited to pre-built views.
- Given a metric+dimension picker, when configured, then a saved custom view is generated and shareable with specific roles/scopes.
- Given a saved report, when scheduled, then it can be emailed periodically to designated recipients.
Priority: Phase 2 | Size: L | Dependencies: EPIC-12.S-04, EPIC-15

### EPIC-12.S-07 — Intervention effectiveness tracking
As an **Executive**, I want to see whether a specific intervention (corrective action, training,
policy change) actually improved the metric it targeted, so that I know whether our responses work.
- Given an intervention linked to an issue/training/policy change, when tracked, then before/after metric trend is shown over a configurable window.
- Given multiple interventions target the same metric, then their relative contribution/effectiveness is comparable side by side.
Priority: Phase 2 | Size: L | Dependencies: EPIC-12.S-04, EPIC-09.S-07

### EPIC-12.S-08 — Data export and BI integration surface
As an **Enterprise Admin**, I want to export control tower data or connect it to our BI tool
(Power BI/Tableau-class), so that our existing enterprise reporting can incorporate this data.
- Given an export request, when generated, then it respects the requester's scope (no cross-scope data leakage in export).
- Given a BI connector is configured, then it reads from the analytics warehouse (not the transactional DB) per conventions.md architecture.
Priority: Phase 2 | Size: M | Dependencies: EPIC-12.S-04, EPIC-13, EPIC-15

### EPIC-12.S-09 — Mobile manager notifications for critical control-tower events
As a **Store/Site Manager**, I want to be pushed a notification when my location crosses a critical
threshold (e.g., new critical issue, health score drop), so that I don't have to keep the dashboard
open to stay aware.
- Given a threshold breach, when detected, then a push notification is sent to the location's manager(s) within the platform's notification delivery SLA.
- Given notification fatigue risk, then thresholds and frequency caps are configurable per tenant.
Priority: MVP | Size: M | Dependencies: EPIC-12.S-01, EPIC-05.S-08

---

## EPIC-13 — Integration Hub

**Module:** `12-integrations-extensibility`
**Goal:** Provide a configuration-first integration experience for standard connectors plus an API/
SDK surface for custom integrations, with full connection lifecycle, secrets, rate limits,
monitoring, retries, and audit.

### EPIC-13.S-01 — Identity connector configuration (SAML/OIDC/SCIM)
As an **Enterprise Admin**, I want to configure identity connectors from the Integration Hub UI, so
that SSO/SCIM setup doesn't require engineering support.
- Given a connector configuration form, when saved, then connection state (connected/error/pending) is displayed and testable via a "test connection" action.
- Given credentials/secrets are entered, then they are stored in central secret management (never in plaintext config) per source-spec §21.2.
Priority: MVP | Size: M | Dependencies: EPIC-02.S-01, EPIC-02.S-02, EPIC-02.S-03

### EPIC-13.S-02 — HR system connector (Workday/SuccessFactors/BambooHR-class)
As an **Enterprise Admin**, I want to sync employee/role/location data from our HR system, so that
user provisioning and org structure stay current without manual duplication.
- Given a scheduled HR sync, when run, then new hires/terminations/transfers reflect as `User`/`UserScope` changes within the configured sync interval.
- Given a sync conflict (e.g., employee already exists with different ID mapping), then it is surfaced for admin resolution rather than silently overwritten.
Priority: Phase 2 | Size: L | Dependencies: EPIC-13.S-01, EPIC-01

### EPIC-13.S-03 — ERP connector (SAP/Oracle/Dynamics-class)
As an **Enterprise Admin**, I want to connect our ERP for reference data (products, assets, cost
centers), so that form lookups and reporting dimensions match our system of record.
- Given a configured ERP connector, when reference data is pulled, then it populates the bounded reference dataset used by form lookups (EPIC-04.S-08) and is cached appropriately for offline use.
- Given ERP data changes, then updates propagate on the configured sync cadence with change tracking.
Priority: Phase 2 | Size: L | Dependencies: EPIC-13.S-01

### EPIC-13.S-04 — Data platform / warehouse connector (Snowflake/BigQuery/Databricks-class)
As an **Enterprise Admin**, I want to stream operational data to our own data platform, so that our
data team can combine it with other enterprise data.
- Given a configured warehouse connector, when data changes occur, then CDC-based delivery (per conventions.md: Debezium + event backbone) streams to the target platform within the agreed latency SLA.
- Given schema evolution occurs, then downstream consumers are notified via a versioned schema contract, not a silent breaking change.
Priority: Phase 2 | Size: L | Dependencies: EPIC-15, EPIC-13.S-01

### EPIC-13.S-05 — BI tool connector (Power BI/Tableau-class)
As an **Enterprise Admin**, I want a native or semi-native connector to our BI tool, so that
executives can view platform data inside their existing BI environment.
- Given the connector is configured, then it authenticates via a scoped service credential, never a shared admin account.
- Given row-level scope restrictions apply to the requesting principal, then the BI connector enforces the same scope filtering as the platform's own APIs.
Priority: Phase 2 | Size: M | Dependencies: EPIC-13.S-04

### EPIC-13.S-06 — Collaboration connector (Teams/Slack-class)
As a **Store/Site Manager**, I want critical alerts and approval requests delivered into our Teams/
Slack channel, so that I don't need to context-switch into the platform for urgent items.
- Given a critical issue/escalation event, when configured, then a message is posted to the mapped channel with a deep link back to the item.
- Given an approval action is available (e.g., approve/reject), when supported by the platform, then it can be actioned directly from the chat message where the integration supports interactive components.
Priority: Phase 2 | Size: M | Dependencies: EPIC-13.S-01, EPIC-07

### EPIC-13.S-07 — Operational systems connectors (POS, inventory, workforce, maintenance)
As an **Operations Admin**, I want to connect POS/inventory/workforce/maintenance systems, so that
operational triggers (e.g., low stock, understaffed shift) can generate work items automatically.
- Given a POS/inventory event feed, when an out-of-stock condition is received, then it can trigger a `WorkItem` via the event-triggered scheduling mechanism (EPIC-03.S-05).
- Given a maintenance system integration, when an asset fault is reported, then an Asset Inspection work item is generated and linked to the `Asset`.
Priority: Phase 2 | Size: XL | Dependencies: EPIC-03.S-05, EPIC-13.S-01

### EPIC-13.S-08 — Custom REST/webhook/SFTP-CSV integration
As an **Enterprise Admin**, I want to build a custom integration via REST API, outbound webhooks, or
scheduled SFTP/CSV transfer, so that systems without a pre-built connector can still integrate.
- Given a webhook subscription, when a canonical event fires, then delivery is signed, versioned, and retryable with idempotent redelivery semantics (per conventions.md and source-spec §18.1/§19.4).
- Given the Integration recovery acceptance test (source-spec §24.1), when a webhook delivery is retried after a transient failure, then it never creates duplicate downstream work due to replay.
- Given an SFTP/CSV export is scheduled, then it runs on the configured cadence and delivers to the configured destination with a delivery confirmation log.
Priority: MVP | Size: L | Dependencies: EPIC-13.S-01

### EPIC-13.S-09 — Integration connection lifecycle and secret rotation
As an **Enterprise Admin**, I want every integration to show connection state, owner, scope, and
support secret rotation without downtime, so that integrations remain secure and don't silently
break.
- Given a secret nearing expiry, when the rotation window opens, then the admin is notified and can rotate without interrupting active sync jobs (dual-secret grace period).
- Given a connector fails repeatedly, then its state changes to `ERROR` and the configured owner is alerted.
Priority: MVP | Size: M | Dependencies: EPIC-13.S-01

### EPIC-13.S-10 — Integration rate limiting and monitoring
As an **Enterprise Admin**, I want each integration to have configurable rate limits and be
monitored for failures/latency, so that a misbehaving integration cannot degrade the platform or
exceed a partner's API limits.
- Given a configured rate limit, when exceeded, then requests are queued/throttled per policy rather than dropped silently.
- Given integration failures, when they exceed a threshold, then they surface on the Observability dashboard (source-spec §22.2: "Integration failures") and can trip a circuit breaker.
Priority: MVP | Size: M | Dependencies: EPIC-13.S-09

### EPIC-13.S-11 — Developer API/SDK surface
As an **Enterprise Admin** (via their engineering team), I want a documented API/SDK for custom
enterprise integrations, so that we can build integrations the Hub doesn't natively support.
- Given the API surface, when accessed, then it follows the standard conventions (base path, OAuth2/OIDC, idempotency keys, cursor pagination, versioned errors — per conventions.md).
- Given an SDK is published, then it wraps common patterns (auth, pagination, retries) so integrators don't reimplement them.
Priority: MVP | Size: L | Dependencies: none beyond core services

---

## EPIC-14 — Security & Compliance

**Module:** `13-security-compliance`
**Goal:** Enforce tenant isolation, encryption, auditability, privacy controls, and disaster
recovery across every layer of the platform so enterprise security and compliance requirements are
met by default, not by exception.

### EPIC-14.S-01 — Tenant isolation enforcement across all layers
As an **Enterprise Admin**, I want tenant isolation proven at API authorization, application
services, database queries, object storage, caches, search, vector retrieval, event routing, worker
processing, analytics, exports, and AI context assembly, so that a tenant ID alone is never trusted
as sufficient isolation.
- Given the Tenant isolation acceptance test (source-spec §24.1), when Tenant A attempts any access path to Tenant B's records, files, search results, embeddings, analytics, or agent context, then it is denied at every layer, not just the API gateway.
- Given Postgres RLS is enabled as a defense-in-depth backstop (per conventions.md), when a query bypasses application-layer scoping due to a bug, then RLS still prevents cross-tenant rows from returning.
Priority: MVP | Size: XL | Dependencies: EPIC-01, EPIC-02

### EPIC-14.S-02 — Encryption in transit and at rest
As an **Enterprise Admin**, I want all data encrypted in transit (TLS) and at rest (KMS-managed
keys), so that we meet enterprise security baselines.
- Given any client-server or service-service call, when made, then TLS is enforced with no plaintext fallback.
- Given data at rest in Postgres, object storage, and caches, then it is encrypted using tenant-aware KMS keys with defined rotation cadence.
Priority: MVP | Size: L | Dependencies: EPIC-14.S-01

### EPIC-14.S-03 — Central secret management and key rotation
As an **Enterprise Admin**, I want all service credentials, integration secrets, and signing keys
centrally managed with rotation, so that no secret is hardcoded or long-lived beyond policy.
- Given a secret rotation schedule, when due, then rotation occurs without service downtime via dual-secret grace periods.
- Given a suspected credential compromise, then an admin can force-revoke and rotate a secret immediately.
Priority: MVP | Size: M | Dependencies: EPIC-14.S-02

### EPIC-14.S-04 — SSO/MFA enforcement
As an **Enterprise Admin**, I want to enforce SSO and MFA tenant-wide, so that authentication meets
enterprise security policy (see also EPIC-02).
- Given MFA is mandated by tenant policy, when a user without MFA logs in, then access is blocked until enrollment completes.
- Given a break-glass admin account exists for emergency access, then its use is logged with heightened audit detail and requires post-use review.
Priority: MVP | Size: M | Dependencies: EPIC-02.S-01, EPIC-02.S-09

### EPIC-14.S-05 — Tamper-evident audit logging
As an **Auditor/Inspector**, I want a tamper-evident audit trail of actor, action, object,
before/after state, and timestamp for every significant event, so that the audit log itself is
trustworthy evidence.
- Given any mutating action, when performed, then an `AuditLog{id, actor, action, object, before, after, timestamp}` record is written per the domain model.
- Given an attempt to modify or delete an `AuditLog` entry directly, then it is rejected at the data-access layer; logs are append-only.
- Given a tamper-evidence mechanism (e.g., hash chaining), when verified, then any post-hoc alteration is detectable.
Priority: MVP | Size: L | Dependencies: EPIC-14.S-01

### EPIC-14.S-06 — PII classification and AI redaction controls
As an **Enterprise Admin**, I want PII/sensitive fields classified and optionally redacted before
being sent to an AI model, so that sensitive data exposure to model providers is controlled.
- Given a field classified as PII, when included in an agent's context, then it is redacted or excluded per the tenant's configured policy before the model call.
- Given a tenant/provider/model allowlist (source-spec §21.3), when an agent attempts to use a non-allowlisted model, then the call is blocked.
Priority: Phase 2 | Size: M | Dependencies: EPIC-11.S-01, EPIC-14.S-01

### EPIC-14.S-07 — Configurable data retention and deletion
As an **Enterprise Admin**, I want configurable retention policies across data classes (submissions,
evidence, agent runs, audit logs) with a defined lifecycle
(`CREATE→ACTIVE→ARCHIVED→RETENTION HOLD (optional)→DELETE/DESTROY`), so that we meet contractual and
regulatory data handling obligations.
- Given a retention policy per data class, when the retention period elapses without a hold, then data transitions to eligible-for-deletion and is purged on the next cleanup cycle.
- Given a legal/regulatory hold is applied, then deletion is blocked regardless of age, and the hold itself is logged.
Priority: MVP | Size: L | Dependencies: EPIC-06.S-05, EPIC-14.S-05

### EPIC-14.S-08 — Vulnerability scanning and penetration testing cadence
As an **Enterprise Admin**, I want continuous dependency/vulnerability scanning and periodic
penetration testing, so that known vulnerabilities are found and fixed proactively.
- Given a new critical CVE affecting a dependency, when detected by automated scanning, then it is triaged within a defined SLA (e.g., 24–72h depending on severity).
- Given a penetration test cycle, when completed, then findings are tracked to remediation with owners and due dates, mirroring the platform's own Issue/CAPA model internally.
Priority: MVP | Size: M | Dependencies: none (ongoing program, cross-cutting)

### EPIC-14.S-09 — Rate limiting and abuse controls
As an **Enterprise Admin**, I want tenant-aware and principal-aware rate limiting on all APIs, so
that abusive or runaway clients cannot degrade service for others.
- Given a principal exceeds their configured rate limit, when a request arrives, then it receives `429` with a `Retry-After` header, not a silent drop or crash.
- Given a suspected abuse pattern (e.g., credential stuffing), when detected, then automated throttling/lockout engages per security policy.
Priority: MVP | Size: M | Dependencies: EPIC-14.S-01

### EPIC-14.S-10 — Disaster recovery drills and RPO/RTO validation
As an **Enterprise Admin**, I want defined RPO/RTO per service tier and periodic DR drills, so that
recovery objectives are proven, not assumed, before production launch.
- Given a defined RPO/RTO per tier (source-spec §23), when a DR drill is executed, then actual recovery time/point is measured against the target and gaps are remediated.
- Given a regional outage simulation, when run, then failover procedures are validated end-to-end including data-plane residency constraints (EPIC-01.S-01).
Priority: MVP | Size: L | Dependencies: EPIC-14.S-01

### EPIC-14.S-11 — Deactivated-user access enforcement
As an **Enterprise Admin**, I want a deactivated user to be immediately blocked from creating new
work while their historical actions remain attributable, so that offboarding is secure without
corrupting audit history.
- Given the User deactivation acceptance test (source-spec §24.1), when a user is deactivated, then they cannot create new work but historical actions remain attributable to their identity.
- Given a deactivated user's active sessions, when deactivation occurs, then sessions are invalidated within a bounded time (e.g., ≤5 minutes).
Priority: MVP | Size: S | Dependencies: EPIC-02.S-03, EPIC-14.S-05

---

## EPIC-15 — Analytics & ML

**Module:** `11-analytics-control-tower`
**Goal:** Build the warehouse/lakehouse-backed analytics foundation, canonical metrics, and
forecasting/anomaly-detection capability that powers the Control Tower and Agent Runtime's
statistical reasoning.

### EPIC-15.S-01 — CDC pipeline into the analytics warehouse
As an **Enterprise Admin** (via Data/Analytics), I want operational data streamed via CDC into the
lakehouse, so that analytical workloads never burden the transactional database.
- Given a change in the transactional Postgres DB, when captured by Debezium (per conventions.md), then it is delivered to the warehouse within the defined freshness SLA.
- Given a schema change in a source table, when it occurs, then the pipeline handles it via a versioned schema contract without silent data loss.
Priority: MVP | Size: XL | Dependencies: EPIC-01, EPIC-03, EPIC-07

### EPIC-15.S-02 — Canonical fact and dimension model
As an **Enterprise Admin** (via Data/Analytics), I want a canonical star/snowflake schema for work,
issues, evidence, and agent activity facts, so that every dashboard and export uses one consistent
model.
- Given the fact tables, when defined, then they support segmentation by location, workflow type, risk, evidence type, and verification method per the North-Star Metric definition (source-spec §26.1).
- Given a dimension changes (e.g., org hierarchy move, EPIC-01.S-03), then fact rows retain effective-dated attribution consistent with the transactional system's historical truth.
Priority: MVP | Size: L | Dependencies: EPIC-15.S-01

### EPIC-15.S-03 — KPI metric library and computation jobs
As an **Enterprise Admin**, I want the canonical KPI set (source-spec §14.2) computed as scheduled/
incremental jobs against the warehouse, so that Control Tower dashboards load fast and consistently.
- Given a KPI definition, when computed, then it matches source-spec §14.2 exactly and is recomputed incrementally, not full-refresh, at scale (100k locations / millions of work items/day per source-spec §23).
- Given a KPI computation job fails, then the prior value is retained with a staleness indicator rather than showing a wrong or blank number.
Priority: MVP | Size: L | Dependencies: EPIC-15.S-02, EPIC-12.S-04

### EPIC-15.S-04 — Predictive risk scoring
As a **Regional/District Manager**, I want a predictive risk score per location forecasting
likelihood of a compliance failure, so that I can intervene before an exception occurs, not just
after.
- Given historical issue/evidence/completion data, when a statistical model scores a location, then the score and its top contributing factors are surfaced (explainability per P8).
- Given the model's forecast usefulness success signal (source-spec §5), then predicted-vs-actual accuracy is tracked over time per location cohort.
Priority: Phase 2 | Size: XL | Dependencies: EPIC-15.S-02

### EPIC-15.S-05 — Anomaly detection on operational metrics
As a **Compliance Monitor** (**AI Agent**), I want statistical anomaly detection on operational
metrics (e.g., sudden spike in a specific violation type at a cluster of locations), so that
systemic issues surface before they become widespread.
- Given a metric time series, when it deviates beyond a statistically defined threshold, then an anomaly event is generated and available as agent context via `get_recent_failures`-class tools.
- Given a detected anomaly, when reviewed, then false-positive feedback can be recorded to tune future sensitivity.
Priority: Phase 2 | Size: L | Dependencies: EPIC-15.S-04, EPIC-11.S-03

### EPIC-15.S-06 — Repeat-failure and root-cause analytics
As an **Executive**, I want enterprise-wide analytics on repeat failure rate and common root causes,
so that I can direct systemic investment (equipment, training, policy) where it has the most impact.
- Given closed issues with recorded RCA method/findings (EPIC-07.S-05), when aggregated, then top root-cause categories are ranked by frequency and cost-of-failure proxy.
- Given the Repeat Failure Rate KPI, then it is computed exactly as defined (repeated exceptions / total exceptions) and segmentable by location/category.
Priority: MVP | Size: M | Dependencies: EPIC-15.S-02, EPIC-07.S-10

### EPIC-15.S-07 — Automation rate and AI governance analytics
As an **Enterprise Admin** (AI Governance), I want Automation Rate and AI Override Rate tracked
enterprise-wide, so that I can measure how much eligible work is safely automated and how much
humans still need to correct.
- Given eligible-for-automation work items, when completed, then the share completed via approved automation (agent L2+ action) versus human action is tracked as Automation Rate.
- Given agent recommendations, when a human changes or rejects them, then AI Override Rate is computed per agent and per tenant.
Priority: Phase 2 | Size: M | Dependencies: EPIC-15.S-03, EPIC-11.S-07

### EPIC-15.S-08 — IoT/asset sensor data ingestion and preventive analytics
As an **Operations Admin**, I want sensor/IoT data (e.g., refrigeration temperature feeds) ingested
alongside manual submissions, so that preventive maintenance interventions can be triggered before a
manual inspection would catch the issue.
- Given a connected IoT feed, when a sensor reading crosses a configured threshold, then it can trigger the same rule/work-item pipeline as a manually captured field value (EPIC-04.S-04, EPIC-03.S-05).
- Given historical sensor + maintenance data, when analyzed, then a preventive-intervention recommendation model surfaces assets at elevated failure risk.
Priority: Phase 2 (basic) / Phase 3 (advanced forecasting) | Size: XL | Dependencies: EPIC-15.S-01, EPIC-13.S-07

### EPIC-15.S-09 — Advanced forecasting and optimization
As an **Executive**, I want longer-horizon forecasting (e.g., projected compliance trend, staffing/
capacity-driven risk) and optimization recommendations (e.g., inspection frequency tuning), so that
resource allocation decisions are data-driven.
- Given a forecasting model, when run, then it produces a confidence interval alongside the point forecast, not a single deterministic number.
- Given an optimization recommendation (e.g., "reduce inspection frequency at low-risk locations"), when presented, then it shows the expected trade-off (cost saved vs. risk change) for human decision.
Priority: Phase 3 | Size: XL | Dependencies: EPIC-15.S-04, EPIC-15.S-08

### EPIC-15.S-10 — Analytics access control and export governance
As an **Enterprise Admin**, I want analytics queries and exports scoped exactly like transactional
data, so that a manager cannot see enterprise-wide data through a dashboard or export they couldn't
see directly.
- Given a scoped user requests a dashboard/export, when the query runs against the warehouse, then row-level filters mirror the transactional authorization model (no "analytics is public" loophole).
- Given an export action, then it is logged in `AuditLog` with the requester, scope, and row count exported.
Priority: MVP | Size: M | Dependencies: EPIC-15.S-02, EPIC-02.S-05, EPIC-14.S-05

---

## Traceability Summary

| Epic | Name | # Stories MVP | # Stories Phase 2 | # Stories Phase 3 |
|---|---|---|---|---|
| EPIC-01 | Tenant & Organization Graph | 8 | 1 | 0 |
| EPIC-02 | Identity & Authorization | 8 | 2 | 0 |
| EPIC-03 | Work Core | 10 | 2 | 0 |
| EPIC-04 | Form Platform | 10 | 1 | 0 |
| EPIC-05 | Mobile Offline | 12 | 0 | 0 |
| EPIC-06 | Evidence | 7 | 2 | 0 |
| EPIC-07 | Issues & CAPA | 9 | 2 | 0 |
| EPIC-08 | Comms & Knowledge | 4 (+1 split*) | 6 (+1 split*) | 0 |
| EPIC-09 | Learning | 0 (2 basic in MVP*) | 8 | 0 |
| EPIC-10 | Visual AI | 6 (pilot) | 2 | 1 |
| EPIC-11 | Agent Runtime | 0 | 11 | 0 |
| EPIC-12 | Control Tower | 6 | 3 | 0 |
| EPIC-13 | Integration Hub | 5 | 6 | 0 |
| EPIC-14 | Security & Compliance | 10 | 1 | 0 |
| EPIC-15 | Analytics & ML | 4 | 4 (+2 split*) | 2 (+... ) |
| **Total** | | **~99** | **~51** | **~3** |

\* Several stories carry a split priority (e.g., "Basic in MVP / Enhanced in Phase 2") per the PRD
Capability Map (source-spec §5: Communications, Knowledge, Learning, Visual AI, Agent Runtime,
IoT/Assets all show a phased MVP→Phase 2→Phase 3 progression). Where a story's header shows two
priorities, it is counted under its primary (first-delivered) phase above and re-scoped for the
later phase without a new story ID.

Total stories in this document: **~153** across 15 epics, consistent with the 6–12 stories/epic
target and full functional coverage of PRD §6–15 and Appendix E.
