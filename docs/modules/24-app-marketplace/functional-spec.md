# Functional Specification — App Marketplace & Template Catalog

Module: `24-app-marketplace` (shortcode `MKT`)
Tier: 3 (ecosystem/packaging layer)
Source: `new-modules-addendum.md` Module Map row 24; source-spec.md §25.4 (Autonomous Ops roadmap
— "Template/workflow marketplace"), §8 (No-Code Forms and Workflow Builder), §15 (Integrations and
Extensibility), §6 (Organization, Identity and Access). Owning service: **Marketplace Service**
(see high-level-design.md). Competitive reference: Wooqer's "500+ ready apps" positioning.

## 1. Purpose and Scope

### 1.1 Purpose

The App Marketplace & Template Catalog lets a tenant discover, preview, and one-click install
pre-built **apps** — curated bundles of one or more `TemplateVersion`s, an optional default
`Workflow`/`Schedule`, and an optional `Integration` connector configuration — instead of building
an operating program from scratch in the Forms & Workflow Builder. It also gives internal product
teams and approved partners a governed path to author, submit, and publish new apps into the
catalog. This module shortens time-to-value for new tenants and new use cases ("go live with a
food-safety program in an afternoon, not a quarter") by packaging proven configurations of
primitives that already exist elsewhere in the platform.

The module's role in the core product loop is narrow and specific: it accelerates the **PLAN**
step by letting an Operations Admin adopt a vetted starting point, which then behaves exactly like
any hand-built template/workflow for every subsequent step (EXECUTE, CAPTURE EVIDENCE, VERIFY,
etc.) — installed content is ordinary tenant-owned content from that point forward.

### 1.2 In Scope

- The **app package** definition: bundled `TemplateVersion`(s), an optional default `Workflow` and
  `Schedule` configuration, an optional reference to an `Integration` connector config template,
  and package metadata (category, industry vertical, description, screenshots, changelog, minimum
  plan tier, required feature flags) (§4.1).
- The **publishing pipeline**: `DRAFT -> SUBMITTED -> IN_REVIEW -> APPROVED -> LISTED ->
  DEPRECATED`, with a `REJECTED` branch, including review criteria, reviewer tooling, and SLA
  tracking (§4.2).
- The **tenant browse/discovery experience**: catalog search, filter by category/industry/rating,
  app detail page, install-preview ("what will be created in your tenant"), and install
  confirmation (§4.3).
- The **install (materialization) flow**: one-click install that creates tenant-scoped, independent
  copies of the bundled `TemplateVersion`(s), `Workflow`, `Schedule`, and (optionally) provisions an
  `Integration` shell — never a shared mutable reference back to the marketplace master (§4.4).
- The **update/upgrade flow**: publisher releases a new `AppVersion`; installed tenants see an
  update-available signal and may opt in to pull the update as a net-new materialization, never a
  silent overwrite of the tenant's post-install customizations (§4.5).
- The **uninstall flow**: removing an `AppInstallation` record and the marketplace-managed linkage,
  while historical `WorkItem`s/`Submission`s created from the app's materialized templates remain
  intact per platform immutability principles (§4.6).
- **Ratings and reviews**: optional tenant-admin ratings/reviews of installed apps, aggregated per
  `AppListing` (§4.7).
- Publisher (partner/internal) onboarding, app authoring workspace, and submission tooling (§4.8).

### 1.3 Out of Scope

- **Authoring form fields, logic, or workflow steps.** This module does not re-implement the Forms
  & Workflow Builder. An app's bundled content is authored in the Forms & Workflow Builder (or
  imported from an existing tenant's templates by an internal publisher with export rights) and
  then *packaged* by this module; the packaging step is a wrapping/versioning/metadata operation,
  not a content-authoring surface. Editing a template's fields happens in the Builder, whether
  that's pre-publication authoring by a Publisher or post-install customization by an installing
  tenant.
- **Authoring or testing connector configuration.** This module does not implement OAuth flows,
  field mapping UIs, or connection health monitoring. It stores a *connector config template*
  (a redacted, credential-free shape of an `Integration` configuration) and, at install time, hands
  the tenant off to the Integration Hub's existing setup wizard pre-populated with that template —
  it never provisions live credentials on the tenant's behalf.
- **Billing, revenue share, or paid-app monetization mechanics.** Flagged as an open business-model
  question in §10; this spec assumes all cataloged apps are free to install for a tenant whose plan
  tier and feature flags qualify.
- **The generic `Template`/`TemplateVersion` state machine, schema validation, or submission
  rendering** (owned by Forms & Workflow Builder module `03`) — this module only calls that
  module's template-creation API and stores a reference to the resulting `template_version_id`.
- **The generic `WorkItem` scheduling/assignment/SLA machinery** that drives execution of an
  installed app's workflow post-install (owned by Work Orchestration module `02`).
- **Integration credential storage, secret rotation, or connector runtime execution** (owned by
  Integrations & Extensibility module `12`).
- **Per-tenant install permission/role definitions themselves** (owned by Identity & Organization
  module `01`) — this module only *enforces* the existing scope/role model when deciding who may
  browse, install, or uninstall.

## 2. Personas

| Persona | Job in this module | Primary surface |
|---|---|---|
| Enterprise/Operations Admin | Browses the catalog, previews install impact, installs/uninstalls apps for their tenant, opts into updates, rates installed apps. | Web Admin Console — Marketplace |
| Internal Publisher | An Anthropic-platform (product) team member who authors a reference app (e.g., "QSR Opening/Closing Pack") from proven customer configurations and submits it for review. | Web Admin Console — Publisher Workspace |
| Partner Publisher | A vetted external partner (systems integrator, ISV, industry consultant) with a Publisher account scoped to their own listings only, who authors and submits vertical-specific apps (e.g., "Hospital Environmental Rounds Pack") for review. | Publisher Portal (same workspace, partner-scoped) |
| Marketplace Reviewer (platform-operator-equivalent role) | Reviews submitted `AppVersion`s against content, security, and quality gates before they may reach `LISTED`; can approve, reject with reasons, or request changes. | Web Admin Console — Review Queue |
| Regional/District Manager | Consumer of the operational outcomes of an installed app (read-only relative to this module) — sees installed-app-sourced `WorkItem`s in their normal Control Tower views with no special handling. | Manager Control Tower |
| AI Agent — Workflow Builder Agent (cross-reference) | Not a marketplace actor itself, but an installed app's default `Workflow` may later be edited by the same agent-assisted authoring surface as any hand-built workflow, post-install. | Builder (out of this module's scope) |

## 3. Key Concepts and Terminology

| Term | Definition |
|---|---|
| `AppListing` | The tenant-agnostic, marketplace-owned "product page" identity for an app — stable across versions (name, publisher, category, aggregate rating). Analogous to `Template`'s role relative to `TemplateVersion`. |
| `AppVersion` | An immutable, versioned snapshot of an app's bundle contents and metadata, carrying its own lifecycle state. Analogous to `TemplateVersion`. |
| Bundle item | One constituent of an `AppVersion`'s package: a template definition, a workflow/schedule definition, or a connector config template. Stored in a marketplace-owned staging format (§ low-level-design.md), not directly in tenant Forms/Integration tables. |
| `AppInstallation` | The record of a specific tenant having installed a specific `AppVersion`, plus pointers to the tenant-owned copies (`TemplateVersion` ids, `Workflow`/`Schedule` ids, `Integration` id) that materialization produced. |
| Materialization | The process of converting an `AppVersion`'s bundle items into real, tenant-owned, independently mutable rows in the Forms & Workflow Builder's and (optionally) Integration Hub's own tables, scoped to the installing tenant. |
| Publisher | The authoring identity (internal product team or partner organization) that owns one or more `AppListing`s and submits `AppVersion`s for review. |

## 4. Functional Requirements

### 4.1 App package definition

- **FR-MKT-001**: An `AppListing` shall have: `name`, `publisher_id`, `category` (enum:
  `SAFETY`, `QUALITY`, `OPENING_CLOSING`, `FOOD_SAFETY`, `MAINTENANCE`, `HR_ONBOARDING`,
  `CUSTOMER_EXPERIENCE`, `SUSTAINABILITY`, `OTHER`), one or more `industry_vertical` tags (enum:
  `RETAIL`, `QSR_RESTAURANT`, `HOSPITALITY`, `FACILITIES`, `LOGISTICS`, `MANUFACTURING`,
  `HEALTHCARE_ADJACENT`, `GENERAL`), a short description, a long description (rich text), an icon
  image, and 1–8 screenshot images.
- **FR-MKT-002**: An `AppVersion` shall bundle: (a) one or more template bundle items, each a
  self-contained template definition (schema, fields, logic — the same shape the Forms &
  Workflow Builder's export/import contract produces, see dependency in §8) with a designated
  "primary" template; (b) zero or one default workflow bundle item (a `Workflow` definition
  referencing the bundled template(s) by their in-bundle logical id, plus an optional default
  `Schedule`); (c) zero or one connector config bundle item (a credential-free `Integration`
  configuration template — connector type, field mapping skeleton, suggested scope — with no live
  secrets); (d) a semantic `version` string, a `changelog` (required for every version after the
  first), a `minimum_plan_tier`, and a list of `required_feature_flags`.
- **FR-MKT-003**: A template bundle item shall declare which of its fields/sections are
  **recommended-immutable** (a soft flag surfaced in the install preview and in the Builder after
  install, e.g., "this field drives the app's built-in risk scoring — edit with caution") versus
  freely editable. This is advisory metadata only; per §1.3 the platform never blocks a tenant from
  editing their own materialized copy.
- **FR-MKT-004**: An `AppVersion` shall be immutable once it reaches `SUBMITTED` (bundle contents
  and metadata are frozen; a publisher wishing to change content must create a new draft version).
  This mirrors the `TemplateVersion` immutability principle (source-spec §8.3) at the app level.
- **FR-MKT-005**: A `AppListing` may have at most one `AppVersion` in each of `DRAFT`, `SUBMITTED`,
  and `IN_REVIEW` at a time (a publisher cannot submit two competing drafts of the same listing
  concurrently); it may have any number of historical `LISTED`/`DEPRECATED`/`REJECTED` versions.

### 4.2 Publishing pipeline

- **FR-MKT-010**: `AppVersion.status` shall follow: `DRAFT -> SUBMITTED -> IN_REVIEW -> APPROVED ->
  LISTED -> DEPRECATED`, with `SUBMITTED -> REJECTED` and `IN_REVIEW -> REJECTED` branches, and
  `REJECTED -> DRAFT` (the publisher may revise and resubmit as a new draft; a rejected version
  itself is terminal and retained for audit). See architecture-diagram.md for the full
  `stateDiagram-v2`.
- **FR-MKT-011**: Moving `DRAFT -> SUBMITTED` requires: all bundle items pass structural validation
  (schema well-formed, referenced in-bundle template ids resolve, no bundled field references an
  external tenant-specific id), at least one screenshot, a non-empty changelog (if not the first
  version), and a publisher attestation checkbox ("this app does not contain customer data,
  malicious logic, or content violating the marketplace content policy").
- **FR-MKT-012**: `SUBMITTED -> IN_REVIEW` is an automatic transition once a Marketplace Reviewer
  claims the item from the review queue; the queue is FIFO by submission time with an SLA target
  (§9 NFRs) and a per-item SLA-breach reminder background job (see low-level-design.md).
  Review criteria include: content quality (spelling, clarity, logical field configuration),
  security (no bundled connector config requests overly broad scope; no field configured to exfiltrate
  data to an unexpected external lookup endpoint), policy compliance (no prohibited industry content,
  no impersonation of another publisher/brand), and functional correctness (the reviewer runs an
  internal "preview install" into a sandbox tenant, using the same materialization code path as a
  real tenant install, to confirm the bundle installs cleanly end-to-end).
- **FR-MKT-013**: A Marketplace Reviewer may `APPROVE` (advances to `APPROVED`, then an automatic or
  manually-triggered `LIST` action moves it to `LISTED` and updates the `AppListing`'s
  "current version"), `REJECT` (requires a structured reason code plus free-text comments returned
  to the publisher), or `REQUEST_CHANGES` (returns the version to `DRAFT` with reviewer comments
  attached, distinct from a hard rejection — does not count against a publisher's rejection-rate
  metric).
- **FR-MKT-014**: `LISTED -> DEPRECATED` is publisher- or platform-operator-initiated. A deprecated
  version remains installable by tenants who already installed it (for reference/reinstall) but is
  removed from catalog search/browse results and from the "install" call-to-action; new installs of
  a deprecated `AppListing` are blocked unless a newer `LISTED` version of the same listing exists,
  in which case the catalog redirects to that version.
- **FR-MKT-015**: Every publishing-pipeline transition emits a canonical event
  (`app.submitted`, `app.review_started`, `app.approved`, `app.rejected`, `app.listed`,
  `app.deprecated`) in the standard envelope (conventions.md), consumed by publisher notifications
  and by the review-queue SLA job.

### 4.3 Tenant browse/discovery

- **FR-MKT-020**: The catalog shall support full-text search (name, description, publisher) and
  filtering by `category`, `industry_vertical`, `minimum_plan_tier` (hide apps the tenant's plan
  cannot install), average rating, and "already installed" state.
- **FR-MKT-021**: Each `AppListing` detail page shall show: description, screenshots, publisher
  identity and (for partner publishers) a verification badge, current `AppVersion` changelog,
  bundle contents summary (e.g., "3 form templates, 1 workflow, 1 Slack connector"), aggregate
  rating and review count, and required plan tier/feature flags with a clear indicator if the
  viewing tenant does not qualify.
- **FR-MKT-022**: Only `LISTED` app versions (the current `LISTED` version per listing) are visible
  in tenant-facing browse/search; `DRAFT`/`SUBMITTED`/`IN_REVIEW`/`REJECTED` versions are visible
  only to the owning Publisher and Marketplace Reviewers.

### 4.4 Install (materialization) flow

- **FR-MKT-030**: Before confirming install, the tenant shall see an **install-preview** listing
  exactly what will be created in their tenant: each template (name, field/section count, and
  whether an existing tenant template has a colliding name — see §6 edge cases), the default
  workflow and schedule (in human-readable form: "runs daily at 8:00 AM, assigned to Store Manager
  role"), and the connector config template if bundled (connector type and required scopes, with a
  note that credentials are configured after install). This preview performs a **dry-run** of the
  materialization logic (low-level-design.md §7) without writing any tenant data.
- **FR-MKT-031**: Confirming install shall trigger materialization: for each template bundle item,
  call the Forms & Workflow Builder's template-creation API to create a new tenant-scoped `Template`
  + initial `TemplateVersion` in that tenant's own tables, with content copied from the bundle
  (never a foreign-key reference into the marketplace's staging tables). For the workflow bundle
  item, create a tenant-scoped `Workflow`/`Schedule` referencing the newly created tenant
  `TemplateVersion` id(s). For the connector bundle item, create a tenant-scoped `Integration`
  record in `NOT_CONFIGURED` state pre-populated with the connector type and suggested field
  mapping skeleton, and deep-link the admin into the Integration Hub's setup wizard to supply live
  credentials — this module never holds or transmits live credentials.
- **FR-MKT-032**: Install shall require the installing user to hold an admin role with both
  `marketplace:install` permission and the underlying `template:create` / `workflow:create` /
  `integration:create` permissions at the target `OrganizationNode` scope (install always targets
  a scope, defaulting to tenant-root; an admin scoped to a subtree may install an app scoped to
  their own subtree only, per source-spec §6 scope rules).
- **FR-MKT-033**: A successful install creates one `AppInstallation` row per tenant per
  `AppListing` (a tenant may reinstall/upgrade in place, tracked as version history on the same
  `AppInstallation`, not a new row — see §4.5) recording the installed `app_version_id`, the
  installing user, the target `OrganizationNode` scope, install timestamp, and the full set of
  materialized object ids (`template_version_ids[]`, `workflow_id`, `schedule_id`,
  `integration_id`) for traceability and for driving the uninstall-impact view (§4.6).
- **FR-MKT-034**: Materialized objects (templates, workflows, integrations) are, from the instant
  of creation, **ordinary tenant-owned objects** subject to that tenant's own edit, versioning, and
  deletion rules in their owning modules. This module retains only the `AppInstallation` linkage
  record for provenance/update-notification purposes — it never re-asserts control over a
  materialized object after creation (FR-MKT-034 is the module's central architectural guarantee;
  see high-level-design.md §6 for the full rationale).
- **FR-MKT-035**: Install is transactional at the `AppInstallation` level: if any bundle item fails
  to materialize (e.g., Forms Service rejects a malformed template), the orchestrator rolls back
  already-created objects from that install attempt (deletes the partially created `Template`(s)/
  `Workflow` via the owning module's own delete API, since they are real tenant objects by that
  point) and returns a single failure to the tenant with a retriable error — see §6 edge cases.

### 4.5 Update / upgrade flow

- **FR-MKT-040**: When a Publisher lists a new `AppVersion` for an `AppListing` that one or more
  tenants have installed, each such tenant's `AppInstallation` is flagged `update_available = true`
  and the installing admin's Marketplace UI shows an "Update available" badge on that app, plus a
  changelog diff between the installed version and the new version.
- **FR-MKT-041**: An update is **opt-in**, never automatic. Pulling an update re-runs the
  install-preview (§4.4) against the new `AppVersion` and, on confirmation, performs a **new
  materialization** of the updated bundle's templates/workflow as additional tenant-owned objects
  (new `TemplateVersion`s under the same tenant `Template`, following that module's normal
  publish-a-new-version flow, or new `Template`s entirely if the bundle's shape changed) — it never
  overwrites the fields the tenant customized on the previously installed version, and never
  silently republishes over a tenant's in-flight edits. The `AppInstallation` record is updated to
  point at the new `app_version_id` and the new materialized object ids, retaining history of the
  prior materialization for audit.
- **FR-MKT-042**: If the tenant has structurally diverged from the originally installed template
  (added/removed fields, changed logic) such that an automatic merge of the new version's changes is
  not well-defined, the update flow presents a **manual reconciliation view**: the new version's
  content side-by-side with the tenant's current customized content, and the admin chooses per
  section whether to adopt the new content, keep their customization, or install the update as a
  parallel new template (leaving the customized one untouched and active). No automatic silent
  merge is performed when divergence is detected.
- **FR-MKT-043**: A tenant may dismiss an update-available notification without upgrading; the
  `AppInstallation` remains on its currently installed version indefinitely (there is no forced
  upgrade — an installed app's underlying `TemplateVersion` is subject to the same lifecycle rules
  as any other, including the option to never publish a newer version).

### 4.6 Uninstall

- **FR-MKT-050**: Uninstalling an `AppInstallation` shows an **uninstall-impact preview**: the
  templates/workflow/integration that will be affected, the count of `WorkItem`s currently
  `AVAILABLE`/`IN_PROGRESS`/`SCHEDULED` against the installed template(s) (which will be orphaned
  from future scheduling but not deleted), and an explicit statement that historical `Submission`s
  and `WorkItem`s already created remain fully intact and queryable.
- **FR-MKT-051**: Uninstall options, chosen explicitly by the admin: (a) **Unlink only** (default) —
  delete the `AppInstallation` marketplace-linkage record only; the materialized `Template`/
  `Workflow`/`Integration` objects remain in the tenant exactly as-is, now indistinguishable from a
  hand-built template, simply no longer tracked for update notifications. (b) **Unlink and retire
  content** — additionally invokes the owning module's own retirement path (`TemplateVersion ->
  RETIRED`, `Workflow` deactivated, future `Schedule` occurrences cancelled) but does not delete
  historical `Submission`/`WorkItem` rows, consistent with source-spec P7 (historical truth) and the
  platform-wide immutability principle. **Hard delete of historical records is never offered by this
  module.**
- **FR-MKT-052**: Uninstalling does not affect other tenants who installed the same `AppListing` —
  `AppInstallation` and its materialized objects are wholly tenant-scoped (§ high-level-design.md
  §6).

### 4.7 Ratings and reviews

- **FR-MKT-060**: A tenant admin whose organization has an active or previously active
  `AppInstallation` for a given `AppListing` may submit one rating (1–5 stars) and an optional
  written review, editable by that same admin, one per tenant per listing (aggregated at the
  organization level, not per-individual-admin, to prevent rating inflation by multiple admins of
  the same tenant).
- **FR-MKT-061**: `AppListing.average_rating` and `rating_count` are recomputed asynchronously on
  review create/update/delete (background job, low-level-design.md §9) and displayed with the
  underlying count so a low-sample-size rating is visibly distinguishable (e.g., "4.8 (3 reviews)").
- **FR-MKT-062**: A Marketplace Reviewer may moderate (hide) a review that violates content policy
  (abuse, PII, competitor disparagement); moderation actions are audit-logged and the original text
  is retained (not hard-deleted) for dispute resolution.

### 4.8 Publisher onboarding and authoring

- **FR-MKT-070**: An internal Publisher account is provisioned by platform operations; a Partner
  Publisher account requires an onboarding/vetting workflow (business verification, agreement
  acceptance) tracked as `Publisher.verification_status` (`PENDING`, `VERIFIED`, `SUSPENDED`).
  Only `VERIFIED` publishers may move an `AppVersion` past `DRAFT`.
- **FR-MKT-071**: The Publisher Workspace lets a publisher compose a bundle by importing template
  definitions (via the Forms & Workflow Builder's export contract — an Operations Admin at the
  publisher's own reference tenant, or an internal product team's staging tenant, exports a
  `TemplateVersion` into the bundle-item staging format), attach a workflow/schedule definition, and
  attach a connector config template, then edit metadata and submit.
- **FR-MKT-072**: A `Publisher` may only manage `AppListing`s they own; cross-publisher visibility
  is limited to the public catalog view (published listings only), never draft/submitted content.

## 5. UX States

| State | Description |
|---|---|
| Catalog browse — empty (no results) | Search/filter combination matches nothing; show "no apps match" with a suggestion to clear filters, never a blank screen. |
| Catalog browse — loading | Skeleton cards while search/filter query executes (§9 latency target applies). |
| Catalog browse — plan-ineligible app | App card shown greyed with a "Requires <plan tier>" badge and a "Contact your account team" affordance instead of an Install button, rather than hiding the app outright (visibility drives upsell). |
| App detail — install preview | Dry-run diff view (§4.4) with per-item collision/warning badges before the final confirm action. |
| App detail — update available | Badge + changelog diff entry point (§4.5). |
| Install — in progress | Non-dismissible progress indicator naming the current materialization step (creating template 1 of 3, etc.) since install can take several seconds (§9 NFR). |
| Install — partial failure | Explicit rollback confirmation state: "Install could not complete; no objects were created" (per FR-MKT-035, never a half-installed state left visible to the user). |
| Uninstall — confirmation | Impact view (§4.6) with the explicit two-option choice (unlink only vs. unlink and retire), defaulting to the non-destructive option. |
| Offline (mobile) | The Marketplace surface is a Web Admin Console-only surface (browsing/installing apps is an administrative action); the mobile app has no marketplace UI — installed content simply appears to frontline users as ordinary assigned work, with no marketplace-specific offline state to design for. |
| Permission-denied | A user without `marketplace:install` sees the catalog in read-only browse mode (if they have `marketplace:view`) with Install buttons disabled and a tooltip naming the required permission/role. |

## 6. Validation and Edge Cases

- **Bundled connector unavailable/unconfigured for tenant's plan tier**: if a connector type
  referenced by the bundle is not enabled for the tenant's plan (Integration Hub connector catalog
  entry itself gated by plan), the install preview shows the connector step as
  "Not available on your plan — the rest of this app can still be installed without it" and the
  tenant may proceed with a partial install that skips only the connector bundle item (this is the
  one case where a "partial" install is an intentional, disclosed choice rather than a failure —
  distinct from FR-MKT-035's failure rollback).
- **Name collision** between an installed app's bundled template name and a tenant's existing
  template name: materialization never fails on a name collision. The install-preview flags it, and
  materialization auto-suffixes the new template's display name (e.g., "Opening Checklist (App:
  QSR Essentials)") while leaving the tenant's pre-existing template completely untouched; the
  underlying `Template.id`/`template_key` is always newly generated and never reuses or aliases an
  existing tenant `Template.id`.
- **Publisher submits a malicious or broken app**: the mandatory `SUBMITTED -> IN_REVIEW ->
  APPROVED` human review gate (§4.2) is the platform's primary control; automated structural/security
  linting (§4.2 FR-MKT-011/012) is a pre-filter, not a substitute for human review. No `AppVersion`
  reaches `LISTED` (tenant-installable) without passing both. A version already `LISTED` that is
  later found to be harmful can be force-`DEPRECATED` by a platform operator outside the normal
  publisher-initiated flow, with existing installations flagged for admin review (a "this app was
  removed from the marketplace for policy reasons — review your installation" notification), but
  existing tenant installations and their historical data are never remotely modified or deleted by
  this action, per the immutability principle.
- **Partial install failure**: see FR-MKT-035 — rollback is all-or-nothing per install attempt,
  using the owning module's own delete APIs on whatever was created before the failure, and the
  operation is safe to retry (idempotent via an `Idempotency-Key` scoped to the install request, so a
  retried install after a transient failure does not create duplicate objects if the first attempt
  actually completed asynchronously after the client gave up).
- **Concurrent installs of the same app by many tenants**: each tenant's materialization calls are
  independent, tenant-scoped Forms/Integration API calls; there is no shared mutable state between
  concurrent installs beyond the read-only `AppVersion` bundle they all read from (see
  low-level-design.md §8 for concurrency handling).
- **Tenant uninstalls then reinstalls**: treated as a fresh install (a new set of materialized
  objects is created; the platform does not attempt to "resurrect" or relink the previously
  materialized, now-orphaned objects from the first install, since those may have been edited or
  retired independently in the interim).
- **Publisher deletes their account / is suspended**: `AppListing`s owned by a suspended publisher
  are automatically transitioned toward `DEPRECATED` for new installs (existing installations
  unaffected) rather than deleted, preserving the audit trail and any tenant's ability to view
  "what app/publisher this content originally came from."

## 7. Non-Functional Requirements

| Requirement | Target |
|---|---|
| Catalog search/browse latency | p95 < 400 ms for a filtered/paginated catalog query (cached — see low-level-design.md §6) |
| Install-preview (dry-run) latency | p95 < 1.5 s for a bundle of up to 10 templates |
| Install (materialization) latency | p95 < 8 s end-to-end for a bundle of up to 10 templates + 1 workflow + 1 connector shell (bounded by synchronous calls to Forms Service/Integration Hub; UI shows progress per §5) |
| Review-queue SLA | 95% of submissions reach a review decision (`APPROVED`/`REJECTED`/`REQUEST_CHANGES`) within 5 business days; SLA-breach reminder job (low-level-design.md §9) fires at 80% of SLA elapsed |
| Availability | Standard platform target (99.9%+ core API, per source-spec §23); a Marketplace Service outage must degrade gracefully — already-installed apps continue to function normally since materialized objects live entirely in Forms/Integration/Work Orchestration tables, independent of Marketplace Service uptime |
| Tenant isolation | Zero cross-tenant visibility of another tenant's `AppInstallation` customizations, ratings-review authorship, or install history — see high-level-design.md §6 |

## 8. Dependencies

| Module | Nature of dependency |
|---|---|
| Forms & Workflow Builder (`03`) | Provides the `Template`/`TemplateVersion` creation API this module calls during materialization; provides the template export contract publishers use to compose a bundle item; owns all post-install editing of materialized templates. |
| Integrations & Extensibility (`12`) | Provides the `Integration` creation/setup-wizard entry point this module deep-links into for a bundled connector; owns connector catalog plan-tier gating this module reads to determine availability (§6). |
| Identity & Organization (`01`) | Supplies `OrganizationNode`/`UserScope`/role model this module enforces for install/uninstall/publish permissions; supplies the tenant/plan-tier context used for eligibility checks. |
| Work Orchestration (`02`) | Consumer, not a dependency in the authoring direction — `WorkItem`s scheduled from an installed app's materialized `Workflow` behave identically to hand-built ones; this module has no direct write path into Work Orchestration tables. |
| Analytics & Control Tower (`11`) | Consumes marketplace adoption events (`app.installed`, `app.uninstalled`, `app.rated`) for catalog-health and adoption reporting; out of scope for this module to build dashboards itself. |

## 9. Resolved Decisions

- **Monetization model: every cataloged app is free to install in this baseline, gated only by plan
  tier/feature flags, never by a purchase.** No `AppListing`/`AppVersion` in this spec carries a price
  or revenue-share term, and no purchase/entitlement/billing step exists in the install flow. This is
  a firm v1 scope boundary, not a placeholder: introducing paid or revenue-share apps is a business-
  model decision with billing, tax, and partner-payout implications well beyond this module's
  technical surface, and is deliberately deferred rather than half-built speculatively. When/if the
  business adopts paid apps, this module's install flow gains a pre-install entitlement check
  analogous to the existing plan-tier check in §6 (`FR-MKT-032`'s authorization gate is the natural
  extension point) — that is a scoped, additive change, not a redesign.
- **Cross-tenant template sharing without marketplace packaging (e.g., a franchise brand pushing a
  template only to its own franchisee tenants) is not built in this baseline.** It is a related but
  distinct capability from the public marketplace (targeted distribution vs. open catalog discovery).
  The default path if/when it is needed is a new "private/organization-scoped listing" visibility
  tier on `AppListing` (reusing this module's existing publish/version/install machinery with a
  narrower audience predicate), not a new module — because the packaging, versioning, and
  materialization mechanics are identical; only the audience-visibility rule differs.
- **Partner publishers operate under a standard commercial agreement (content ownership, liability,
  support commitment) executed outside this system.** This module tracks only `Publisher.
  verification_status` (`PENDING`/`VERIFIED`/`SUSPENDED`) as the system-of-record signal gating
  publishing rights (FR-MKT-070); contract terms and their negotiation are a business/legal process
  this module does not model or store.
- **The catalog and search index are designed for thousands of listings across many publishers from
  day one**, even though the initial internal catalog is small. Retrofitting search-index/caching
  architecture after organic growth is materially more disruptive than building for the
  Wooqer-comparable "500+ apps" scale target up front; §7 NFRs and the low-level-design.md OpenSearch/
  Redis approach are sized to this target from the first release, not added later.
