# Module 01 — Identity & Organization

> The foundation module every other module sits on. It holds each customer company (a "tenant"), the company's org tree (regions, districts, stores), its users, and the answer to "who can see and do what, where." It is configured mainly by Enterprise and Operations Admins, and silently consumed by every login, task, form, and report in the platform.

| | |
|---|---|
| **Module code** | None assigned in the spec (identified by folder `01-identity-organization`) |
| **Owning service** | Identity Service + Organization Service + Authorization Service (three logical services, deployed as one bounded-context service initially) |
| **Epics** | EPIC-01, EPIC-02 |

## In plain terms
- Build the company structure of any depth — Enterprise → Region → District → Store, or custom node types — and move stores between regions without breaking old reports ("this audit belongs to Region West, because that's where the store was in August").
- Invite users, log them in with single sign-on (SAML/OIDC), and auto-provision/deprovision them from corporate identity systems like Okta or Azure AD (SCIM).
- Give people roles scoped to parts of the tree — e.g. "Store Manager at Site 42" or "Auditor across Region West until March 31" — with rights flowing down to child locations by default.
- Add explicit deny rules ("this role can see Region West, but not this one site") and attribute conditions ("only during business hours").
- Use the Effective Access preview to see exactly why a user does or doesn't have a permission at a location — including which scope row or deny rule caused it.
- Trust the safety net: cached permissions expire within seconds, sessions die immediately on deactivation, and a versioned authorization regression test suite guards every endpoint against privilege-escalation bugs.

## Key concepts
| Concept | What it means |
|---|---|
| Tenant | One customer company. The root isolation boundary: nothing is ever visible across tenants. |
| OrganizationNode | One entry in the org tree (a region, district, brand, or store). Effective-dated: each change creates a new version row, keyed by a stable `node_key`. |
| Role | A named bundle of permissions (`work_item:approve`, `issue:export`, …). System roles are shared; tenants can define their own. |
| UserScope | The link that says "this user holds this role at this subtree, from this date to that date" — optionally with a `DENY` effect or attribute conditions. |
| Effective Access preview | An admin tool that shows the full resolved permission set for a user at a node, with the provenance of every allow/deny. |
| SSO (SAML 2.0 / OIDC) | Corporate single sign-on: users authenticate at their company's identity provider; just-in-time user creation is configurable per tenant. |
| SCIM provisioning | Automatic user/group syncing from an identity provider; removing someone from a mapped group instantly closes their access and revokes sessions. |
| Closure table | A performance index of all ancestor/descendant pairs that lets permission checks resolve "everything under Region West" in O(1) lookups at 100k+ node scale. |

## How it connects to the rest
- Depends on no other business module — it is the platform's foundation; every other module (Work Orchestration 02, Forms 03, Mobile 04, Evidence 05, Issues/CAPA 06, and onward) resolves tenant, org scope, and permissions through it.
- Work Orchestration (02) consumes locations, timezones, and role/scope data to decide who is eligible for assigned work; it also reacts to `user.deactivated` to mass-reassign open work.
- The Mobile app (04) caches the pre-resolved effective permission set this module hands it; it never recomputes permissions on-device.
- Analytics/Control Tower (11) uses its dual org-attribution model (current structure vs. structure as of event time) for correct historical reporting, and the Agent Runtime (10) reuses its OPA-based policy evaluation pattern for agent actions.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-ORG-001–081 covering tenant lifecycle, effective-dated org hierarchy, users, roles/scopes/deny, SSO, SCIM, Effective Access, historical attribution, and the authorization regression test suite — plus UX states, edge cases, and resolved decisions. |
| [high-level-design.md](./high-level-design.md) | The three-service split (Identity / Organization / Authorization), data ownership boundaries, Redis permission-cache strategy, scaling to 100k nodes / 1M users, failure-mode table, and justified technology choices. |
| [low-level-design.md](./low-level-design.md) | Full Postgres schema (`tenants`, effective-dated `organization_nodes`, `user_scopes`, sessions, IdP configs), API shapes, the permission-check algorithm, closure-table maintenance on reorg, caching, concurrency, and background jobs. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component diagram, the cache-first permission-check sequence, the ER diagram, and the effective-dating timeline showing how a store's history stays correctly attributed across a rename and a reorg. |
