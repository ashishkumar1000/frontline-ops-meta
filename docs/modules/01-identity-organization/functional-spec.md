# Identity & Organization — Functional Specification

Module: `01-identity-organization`
Epics: EPIC-01 (Tenant & Organization Graph), EPIC-02 (Identity & Authorization)
Source: `source-spec.md` §6, §17.1, §21.1, Appendix E

## 1. Purpose

This module is the foundation every other module in the platform depends on. It owns:

- The **Tenant** as the root isolation boundary.
- The **Organization Graph** (`OrganizationNode`): an arbitrary-depth, effective-dated tree
  representing corporate, franchise, brand, business-unit, department and location structures.
- **User** identity, lifecycle, and SSO/SCIM provisioning.
- **Authorization**: RBAC + scope-based + attribute-based (ABAC) access control, role inheritance,
  explicit deny, and the "Effective Access" preview tool.
- Authorization regression testing as a standing engineering practice, not a one-time audit.

No work item, form, evidence record, issue, knowledge document, agent action, or analytics fact can
be created, read, or reported on without resolving through this module's tenant, organization-scope,
and authorization primitives. **Every other module in this platform is a downstream consumer of
Identity & Organization.**

## 2. Scope

### 2.1 In scope

- Tenant lifecycle: creation, activation, suspension, plan/tier assignment, data residency
  assignment, tenant-level policy configuration (password/session policy, SSO enforcement, IP
  allowlists at the tenant level).
- Organization hierarchy CRUD: arbitrary-depth node types (Enterprise, Country, Region, District,
  Brand, Business Unit, Department, Store/Site, and tenant-defined custom node types).
- Effective-dated hierarchy changes (node creation, move/reparent, rename, deactivation) so that
  historical reports remain attributable to the org structure in force at the time of the event.
- User identity lifecycle: invite, activate, deactivate, reactivate, delete (soft), merge duplicate
  identities from provisioning sources.
- SSO integration: SAML 2.0 (SP-initiated and IdP-initiated) and OIDC (authorization code + PKCE).
- SCIM 2.0 provisioning: user and group (role/scope) push provisioning and deprovisioning from an
  external IdP (Okta, Azure AD/Entra ID, OneLogin, Google Workspace class).
- RBAC: system-defined and tenant-custom roles, each a named bundle of permissions.
- Scope-based access: a `UserScope` binds a `User` + `Role` to an `OrganizationNode` subtree, with
  its own `valid_from`/`valid_to`.
- Role inheritance: a scope granted at a parent node applies to all descendant nodes by default,
  with an explicit deny/exception mechanism at any descendant node.
- Attribute-based policy overlays (e.g., "only if `work_item.location.brand == user.brand`",
  "only during business hours", "only for `location.type == 'Store'`") evaluated in combination
  with RBAC/scope.
- "Effective Access" preview: pick a user + a node (+ optional as-of date) and see the exact
  resolved permission set.
- Authorization regression test suite definition (horizontal/vertical privilege escalation, IDOR,
  stale permission, cross-tenant access) as a first-class, versioned test asset owned by this
  module's CI.
- Session management: session issuance, refresh, revocation (including cascading revocation on
  deactivation/SCIM deprovisioning).

### 2.2 Out of scope (owned by other modules, consumed here only by reference)

- Work item, form, evidence, issue business logic (Work/Form/Evidence/Issue services) — this module
  only supplies the scope/role context they authorize against.
- Notification delivery mechanics (Notification Hub) — this module only emits events and exposes
  user contact preferences via the `User` record's linkage to `NotificationPreference` (owned
  elsewhere).
- Agent Runtime policy engine for *agent tool execution* policy (EPIC-11) — this module supplies the
  underlying RBAC/ABAC primitives and the OPA policy evaluation pattern that the Agent Runtime
  reuses, but agent-specific autonomy levels and tool allowlists are owned by EPIC-11.
- Billing/plan metering — Tenant carries `plan` as a reference field; billing logic lives in a
  separate commercial systems integration, not in this module.

## 3. Personas served

| Persona | Relationship to this module |
|---|---|
| Enterprise Admin | Primary configurator: tenant policy, SSO/SCIM setup, top-of-tree hierarchy, system role definitions, Effective Access preview, authorization audit review. |
| Operations Admin | Configures mid-tree hierarchy (regions/districts/brands), assigns roles/scopes for their delegated subtree, uses Effective Access preview before rolling out standards. |
| Regional/District Manager | Consumer: scope-limited to their subtree; may be delegated user/role administration for their own subtree only (delegated admin). |
| Store/Site Manager | Consumer: scope-limited to one or a few leaf nodes; sees only their assigned team and location context. |
| Frontline Associate | Consumer: authenticates (often via SSO/shared-device login), scope resolves to a single location. |
| Auditor/Inspector | Consumer: scope may span multiple locations/regions for audit purposes, typically time-boxed via effective-dated `UserScope`. |
| Executive | Consumer: read-scoped across the full tenant tree (or a business-unit subtree) for reporting; enforced via a reporting-only role that inherits read but not write/administer. |
| AI Agent | Consumer: every `AgentRun` context assembly step resolves the acting principal's (service identity's or delegated user's) effective scope through this module before retrieval/tool execution; never bypasses it. |

## 4. Functional Requirements

### 4.1 Tenant lifecycle

- **FR-ORG-001**: The system shall support creation of a `Tenant` with a unique `slug`, `name`,
  `status` (`PROVISIONING`, `ACTIVE`, `SUSPENDED`, `OFFBOARDING`, `DELETED`), `plan`, `data_residency`
  region, and a default `policies` bundle (password policy, session timeout, SSO-required flag,
  SCIM-enabled flag).
- **FR-ORG-002**: Tenant creation shall provision a root `OrganizationNode` (type `ENTERPRISE`)
  automatically; a tenant cannot exist without exactly one root node active at any point in time.
- **FR-ORG-003**: Suspending a tenant (`ACTIVE → SUSPENDED`) shall immediately invalidate all cached
  permissions and active sessions for that tenant within the permission-cache TTL bound (§ NFR-ORG-002)
  and block all new API traffic except read-only admin/export endpoints needed for offboarding.
  Suspension is reversible; `DELETED` is not (soft-delete with a retention hold, per source-spec
  §21.4 data lifecycle).
- **FR-ORG-004**: Tenant-level policy changes (e.g., toggling `sso_required`) shall be effective-dated
  (`effective_at` timestamp) and audited via `AuditLog`, and shall never be applied retroactively to
  already-issued sessions without an explicit re-authentication requirement.

### 4.2 Organization hierarchy CRUD & effective dating

- **FR-ORG-010**: The system shall support arbitrary-depth `OrganizationNode` trees. Node `type` is a
  tenant-configurable enum (seeded defaults: `ENTERPRISE`, `COUNTRY`, `REGION`, `DISTRICT`, `BRAND`,
  `BUSINESS_UNIT`, `DEPARTMENT`, `SITE`); the platform shall not hard-code a fixed depth or a fixed
  Enterprise→Country→Region→Store shape.
- **FR-ORG-011**: Every `OrganizationNode` shall carry `valid_from` and `valid_to` (nullable = open-
  ended). Creating a node, renaming it, changing its `type`, or moving it (reparenting) shall never
  mutate history in place; each such change closes the current row's `valid_to` and inserts a new
  row with the same logical node identity (`node_key`, stable across versions) and a new
  `valid_from`.
- **FR-ORG-012**: A node "move" (reorg) operation shall accept an `effective_at` timestamp that may
  be in the future (scheduled reorg) or immediate. The system shall recompute the materialized path
  for the moved node and all descendants as of `effective_at`, without altering the path recorded
  against any historical fact (work items, submissions, issues) created before `effective_at`.
- **FR-ORG-013**: The system shall prevent circular hierarchies: a move operation that would make a
  node its own ancestor (directly or transitively) shall be rejected with error code
  `ORG_CYCLE_DETECTED` before any write is committed.
- **FR-ORG-014**: The system shall prevent orphaned nodes: a node cannot be created or moved without
  a valid, active (or future-effective) parent, except the single root node per tenant. Deleting
  (deactivating) a node with active children shall be rejected (`ORG_NODE_HAS_ACTIVE_CHILDREN`)
  unless the caller explicitly cascades the deactivation with `effective_at` applied to all
  descendants atomically.
- **FR-ORG-015**: Bulk hierarchy import/edit (CSV/API batch) shall validate the entire proposed
  change set (cycle check, orphan check, duplicate `node_key` check, type-transition legality) before
  committing any row, and shall return a per-row validation report on failure.
- **FR-ORG-016**: Historical reports and analytics facts shall resolve organizational attribution by
  `(node_key, as_of_timestamp)`, not by current parent pointer, so that a location moved from Region
  A to Region B after an audit was submitted still reports that audit under Region A for periods
  before the move's `effective_at` (source-spec §24.1 "Reorganization" acceptance test).

### 4.3 User lifecycle

- **FR-ORG-020**: The system shall support user invite (email or SSO-JIT), activation, deactivation,
  reactivation, and soft-delete. A `User.status` transitions
  `INVITED → ACTIVE → DEACTIVATED [→ ACTIVE]`, and `ACTIVE|DEACTIVATED → DELETED` (soft, retained for
  attribution).
- **FR-ORG-021**: Deactivating a user shall: (a) revoke all active sessions immediately, (b) prevent
  the user from creating new work items, submissions, approvals, or agent-triggering actions, (c)
  preserve every historical `AuditLog`, `Submission`, `WorkItem` and `Issue` attribution to that
  user's identity unchanged (source-spec §24.1 "User deactivation" acceptance test), and (d) retain
  the user's `UserScope` rows with `valid_to` set to the deactivation timestamp rather than deleting
  them, so that "who had access to what, when" remains answerable.
- **FR-ORG-022**: A deleted (soft) user's `identity_ref` (SSO subject / SCIM external ID) shall be
  released only after the tenant's configured retention period elapses, and re-provisioning of the
  same external identity before that period creates a new logical user unless an explicit "restore"
  action is taken by an admin.
- **FR-ORG-023**: The system shall detect and surface (not silently merge) duplicate identities
  arising from SCIM re-provisioning with a changed external ID but matching email/employee ID, via a
  reconciliation queue reviewed by an Enterprise Admin.

### 4.4 Role, scope assignment & inheritance

- **FR-ORG-030**: A `Role` is a named, tenant-scoped (or system-provided, non-editable) bundle of
  permissions. Permissions are expressed as `resource:action` pairs (e.g., `work_item:approve`,
  `org_node:administer`, `issue:export`) with optional attribute constraints.
- **FR-ORG-031**: A `UserScope` binds `(user_id, role_id, node_id)` with `valid_from`/`valid_to`. A
  user may hold multiple concurrent `UserScope` rows across different nodes and roles (e.g., "Store
  Manager at Site 42" + "Auditor at Region West" from 2026-01-01 to 2026-03-31).
- **FR-ORG-032**: By default, a `UserScope` granted at node N grants the bound role's permissions at
  N and at every descendant of N as of evaluation time (role inheritance). This is the default and
  requires no additional configuration.
- **FR-ORG-033**: The system shall support an explicit **deny** exception: an admin may attach a deny
  rule for a specific `(role_permission, node_subtree)` combination that overrides inherited grants
  for that subtree only, without revoking the parent-level grant elsewhere. Deny always wins over
  allow when both apply to the same resolved node (deny-overrides-allow conflict resolution).
- **FR-ORG-034**: Assigning conflicting concurrent scopes (e.g., two different roles at the same node
  with overlapping validity that both grant and deny the same permission through different paths)
  shall not be blocked at assignment time (business reality requires overlapping grants); the
  authorization engine resolves conflicts at evaluation time using deny-overrides-allow, and the
  Effective Access preview shall show which specific `UserScope` row is the source of any denial so
  admins can debug the conflict.
- **FR-ORG-035**: Role definition changes (adding/removing a permission from a `Role`) shall
  invalidate the permission cache for every user holding a `UserScope` against that role within the
  cache TTL bound and shall be audited.
- **FR-ORG-036**: The system shall support attribute-based conditions on a role-permission binding
  (e.g., restrict `evidence:export` to `location.type == 'Site'` and `time.hour BETWEEN 6 AND 22`,
  tenant-configurable via policy, evaluated per §5 of the LLD).

### 4.5 SSO (SAML2/OIDC)

- **FR-ORG-040**: The system shall support SAML 2.0 SP-initiated and IdP-initiated SSO per tenant,
  with tenant-specific IdP metadata (entity ID, SSO URL, signing certificate), attribute mapping
  configuration (NameID/email/given name/family name/group claims → `User` fields and optional
  group-to-role mapping), and clock-skew-tolerant assertion validation.
- **FR-ORG-041**: The system shall support OIDC (authorization code + PKCE) per tenant, with
  configurable issuer, client credentials, and claim mapping equivalent to FR-ORG-040.
- **FR-ORG-042**: Just-in-time (JIT) provisioning shall be configurable per tenant: on first
  successful SSO login for an unrecognized `identity_ref`, the system may auto-create a `User` in
  `ACTIVE` status with a default role/scope (configurable), or reject the login
  (`SSO_JIT_DISABLED_UNKNOWN_USER`) if JIT is disabled and the user is not pre-provisioned via SCIM.
- **FR-ORG-043**: SSO failures shall be classified and surfaced distinctly: IdP unreachable,
  signature/certificate invalid, assertion expired/replayed, unmapped required attribute missing,
  user deactivated, tenant SSO-required policy violated (user attempted password login when SSO is
  mandatory). Each maps to a distinct error code and a tenant-brandable failure page (see § 5.5).
- **FR-ORG-044**: The system shall support IdP-side metadata rotation (certificate rollover) with a
  configurable overlap window (old + new certificate both accepted) to avoid a hard cutover outage.

### 4.6 SCIM provisioning

- **FR-ORG-050**: The system shall implement SCIM 2.0 `/Users` and `/Groups` endpoints (create, read,
  update, patch, deactivate/delete) compliant enough for Okta, Azure AD/Entra ID, and OneLogin
  connectors to provision and deprovision users and group memberships without custom mapping code.
- **FR-ORG-051**: A SCIM group shall map to a tenant-configured `(Role, OrganizationNode)` pair (a
  "provisioning binding"); adding a user to a mapped SCIM group creates/extends the corresponding
  `UserScope`, and removing them closes its `valid_to` at the sync timestamp.
- **FR-ORG-052**: SCIM deprovisioning (`active: false` PATCH or DELETE) shall deactivate the `User`
  per FR-ORG-021 and cascade an immediate session revocation (see LLD sequence diagram (c)).
- **FR-ORG-053**: SCIM sync failures (malformed payload, unmapped group, duplicate external ID,
  rate-limit from IdP-initiated bulk push) shall be captured in a per-tenant SCIM error log visible
  to the Enterprise Admin, with retry semantics for transient failures and a dead-letter state for
  permanent ones requiring manual reconciliation.
- **FR-ORG-054**: A background reconciliation job shall periodically (tenant-configurable interval,
  default 24h) diff the last-known SCIM state against current `User`/`UserScope` state and flag
  drift (e.g., a user deactivated in the IdP but not yet reflected here due to a missed webhook) as
  a corrective task for admin review, not a silent auto-fix.

### 4.7 Effective Access preview

- **FR-ORG-060**: The Admin Console shall provide an "Effective Access" tool: given a `User` and an
  `OrganizationNode` (and an optional `as_of` date, default now), it shall display the complete
  resolved permission set (read/create/approve/reassign/export/administer, per resource type) that
  the user holds at that node, including which node in the tree contributed each permission via
  inheritance, and which (if any) deny rule suppressed a permission.
- **FR-ORG-061**: The preview shall support comparing two `as_of` dates (e.g., before/after a
  scheduled reorg or a role change) to show a diff of what access will change and when.
- **FR-ORG-062**: The preview computation shall use the same authorization evaluation code path used
  by the live Authorization Service (no parallel/divergent implementation), to guarantee the preview
  is truthful.
- **FR-ORG-063**: The preview shall be usable by a delegated admin only for users/nodes within their
  own granted scope; an Enterprise Admin can preview any user/node in the tenant.

### 4.8 Historical attribution after reorganization

- **FR-ORG-070**: When an `OrganizationNode` is moved, renamed, or has its `type` changed, all
  historical facts referencing that node by `node_key` shall continue to resolve their org-path
  label using the node version whose `[valid_from, valid_to)` window contains the fact's
  `occurred_at` timestamp, not the current version.
- **FR-ORG-071**: Reports and exports that aggregate "by current org structure" vs. "by org structure
  as of event time" shall be explicitly distinguishable options in the reporting layer's query
  contract (this module exposes both resolution modes via its API; the Analytics module chooses
  which to use per report).

### 4.9 Authorization regression testing

- **FR-ORG-080**: The module shall maintain a versioned, CI-executed authorization regression test
  suite covering: horizontal privilege escalation (user A accessing user B's same-level resource),
  vertical privilege escalation (lower role attempting higher-role action), IDOR (direct object
  reference bypassing scope filter), stale permission (role/scope revoked but cached permission still
  granting access past TTL), and cross-tenant access (tenant A principal attempting any read/write
  against tenant B data) per source-spec §21.1 and §24.1.
- **FR-ORG-081**: Every new permission-bearing endpoint added anywhere in the platform shall be
  required (via a CI gate owned by this module) to have at least one authorization regression test
  registered against it before merge.

## 5. UX States

| State | Behavior |
|---|---|
| Empty org tree (new tenant) | Show only the auto-created root node with a guided "Add your first region/site" call to action; block navigation into node-scoped admin screens until at least one child node exists. |
| Large org tree (10k–100k+ nodes) | Tree UI is virtualized (windowed rendering, lazy-loaded children per expand, server-side search-to-node with path breadcrumb jump); never renders the full tree client-side; default view loads only the current tenant's top 2 levels expanded. |
| Effective-dated change preview | Before committing a reorg/role change, show a diff preview (nodes affected count, users affected count, active work items whose location will change org-path, scheduled effective date) — modeled on the Impact Preview pattern used by the Form Platform module for consistency. |
| SCIM sync error | Persistent admin-visible banner + itemized error log entry per failed record; does not block unrelated successful records in the same batch (partial success). |
| SSO failure | Distinct, branded error screens per FR-ORG-043 classification; never a generic "login failed"; includes a support/trace_id reference and, where applicable, a fallback path (break-glass local admin login) if tenant policy allows it. |
| Effective Access preview — no access | Explicitly renders "No access at this node" rather than an empty table, and states why (no `UserScope` covers this subtree, or a deny rule suppresses all permissions). |
| Reorg in flight, active work at moved node | Warn admin that N `WorkItem`s are `IN_PROGRESS`/`AVAILABLE` at the affected node(s); the move proceeds (attribution rules protect history) but the admin is shown the count and a link to review before confirming. |
| User deactivation with open approvals/ownership | Warn admin of open `Issue`/`WorkItem` ownership or pending `Approval` steps assigned to the user; require explicit reassignment or acknowledgment before completing deactivation. |

## 6. Validation & Edge Cases

- Circular hierarchy prevention: validated server-side on every create/move (FR-ORG-013); client UI
  also disables invalid drop targets in the tree editor as a UX aid, not as the sole control.
- Orphaned nodes: a node's parent must exist and be valid as of the child's `valid_from` (FR-ORG-014).
- Duplicate `node_key` within a tenant across time-versions is expected (same logical node, multiple
  effective-dated rows); duplicate `node_key` with overlapping `[valid_from, valid_to)` windows is a
  data integrity violation, enforced via an exclusion constraint (LLD §2).
- Role assignment conflicts: overlapping allow/deny at the same node resolved by deny-overrides-allow
  (FR-ORG-034); overlapping allow grants from different roles are additive (union of permissions).
- Mid-flight reorg during active work: a reorg does not retroactively change the org attribution of
  work already `SUBMITTED`/`VERIFIED`; work still `IN_PROGRESS` at the moment `effective_at` passes
  is attributed to the *new* node going forward but its history keeps prior-node attribution for
  events before the cutover (FR-ORG-070).
- Self-scope escalation: a user cannot grant themselves a role/scope exceeding their own effective
  permissions; the assignment API validates the caller's own resolved permission covers
  `org_node:administer` at the target node and that the role being granted is not outside the
  caller's own held permission set for delegated admins (Enterprise Admins are exempt from the
  ceiling check).
- SCIM race: near-simultaneous SCIM deprovision and an in-flight interactive session; deprovision
  wins — session revocation is enqueued synchronously as part of the deprovision transaction (LLD
  sequence diagram (c)), not eventually-consistent only.
- Tenant suspension mid-session: in-flight API calls complete only if already past the authorization
  check at the moment of suspension; new calls are rejected once the suspension flag is visible to
  the Authorization Service's cache (bounded by cache TTL, FR-ORG-003).

## 7. Non-Functional Requirements (module-specific)

| Requirement | Target | Notes |
|---|---|---|
| Permission check latency | p95 < 15 ms, p99 < 40 ms (cache hit); p95 < 80 ms (cache miss, full evaluation) | Sits inside the platform's overall API p95 < 300 ms budget (source-spec §23); every other module's request path includes at least one permission check. |
| Effective Access preview response | p95 < 500 ms for a node with up to 100k descendant nodes | Uses precomputed closure/materialized path, not live tree walk. |
| Org hierarchy scale | 100,000+ active nodes per large tenant, 1,000,000+ total users across tenants | Materialized path + closure table strategy (HLD §4). |
| SSO assertion validation | p95 < 200 ms including IdP round trip where applicable | Excludes IdP-side latency, which is out of this module's control. |
| SCIM throughput | Sustain 50 requests/sec sustained, burst 200/sec per tenant | Matches large-IdP bulk sync bursts (e.g., initial provisioning of 50k users). |
| Session revocation propagation | < 5 seconds from deactivation/SCIM event to all edge enforcement points | Enforced via cache invalidation event, not TTL expiry alone. |
| Authorization cache staleness ceiling | 5 seconds default TTL, tenant-configurable down to 1 second for high-security tenants | Trade-off documented in HLD §7. |
| Availability | 99.95% (higher than the platform's general 99.9% floor) | Every module's availability is bounded by this module's; treated as a Tier-0 dependency. |

## 8. Dependencies

This module has no upstream dependency on any other business module. Every other module depends on
it:

- **Work Orchestration, Form Platform, Evidence, Issues/CAPA, Communications/Knowledge, Learning,
  Visual AI, Control Tower, Integration Hub** — all resolve `tenant_id` + `OrganizationNode` scope +
  `UserScope`/`Role` permissions for every read/write.
- **Agent Runtime (EPIC-11)** — reuses this module's OPA-based policy evaluation pattern and RBAC/ABAC
  primitives as the substrate for agent action authorization; agent-specific autonomy levels are
  layered on top, not a replacement.
- **Analytics/Control Tower** — consumes the dual-resolution org attribution model (current-structure
  vs. as-of-event-time) for correct historical reporting.
- **Notification Hub** — consumes `User` contact/locale/timezone fields and scope to determine
  eligible recipients for role-triggered/location-triggered notifications.

## 9. Resolved Decisions

- **Custom node type attribute schemas (FR-ORG-010)**: Custom node types carry custom attribute
  schemas. Each tenant-defined node `type` may register a JSON-Schema document that validates the
  `attributes` jsonb column on `organization_nodes` for nodes of that type; the API rejects a
  create/update whose `attributes` fail the registered schema. This is decided as firm behavior
  (not optional) because an unvalidated free-form `attributes` bag would make Effective Access and
  reporting integrations that key off specific attributes unreliable across tenants.
- **Physical site-to-node mapping (FR-ORG-010, FR-ORG-014)**: A single physical Site/Store maps to
  exactly one leaf `OrganizationNode`. A multi-brand co-located site is modeled as multiple sibling
  `SITE` nodes sharing one `physical_location_id` value, never as one node carrying multiple brands.
  This keeps authorization scope strictly 1:1 with a node, which is required for the permission-cache
  key shape (`perm:{tenant_id}:{user_id}:{node_id}`, low-level-design.md §5) to remain unambiguous.
- **Permission cache TTL (FR-ORG-003, FR-ORG-035, NFR-ORG "Authorization cache staleness ceiling")**:
  Default TTL is **5 seconds**, tenant-configurable down to **1 second** for regulated tenants that
  require a tighter staleness bound; this is the platform default already fixed by
  `engineering-decisions.md` §8 (encryption/config) working in concert with this module's own NFR
  table (§7) — no further tenant configuration path is added beyond that 1–5 second range, since
  going below 1 second materially increases Redis load for negligible security benefit given the
  event-driven invalidation triggers already in place (low-level-design.md §5).
- **Break-glass local-admin login**: Offered, but only as a narrowly-scoped exception path. Exactly
  one break-glass account per tenant is permitted, it is disabled by default, an Enterprise Admin
  must explicitly opt in per tenant, mandatory MFA (WebAuthn/TOTP) is enforced on it with no
  exception, every use is logged to `AuditLog` with a dedicated `BREAK_GLASS_LOGIN` event type and
  triggers an immediate notification to all Enterprise Admins of the tenant, and the account cannot
  hold `org_node:administer` or `role:administer` permissions beyond what is needed to restore normal
  SSO access. This is decided as the firm design because it eliminates the risk of break-glass
  becoming a silent, unmonitored bypass of tenant-mandated SSO.
- **SCIM reconciliation job interval (FR-ORG-054)**: Default interval is **24 hours**, tenant-
  configurable down to **1 hour** for regulated tenants requiring faster drift detection (the
  configurable floor of 1 hour, not lower, keeps reconciliation-job load bounded across a large
  tenant population while still catching a missed-webhook drift well within a single business day
  for the tenants that need it).
