# Module 24 — App Marketplace & Template Catalog

> Setting up an operating program (opening/closing checklists, safety audits, food-safety logs)
> from scratch takes weeks; this module offers a catalog of pre-built "apps" — curated bundles of
> form templates, a default workflow and schedule, and optionally a connector configuration —
> that a tenant admin can browse, preview, and install in one click. It is the platform's answer
> to "go live with a food-safety program in an afternoon, not a quarter," and it also gives
> internal teams and vetted partners a governed pipeline to publish new apps. Enterprise/
> operations admins install apps; internal and partner publishers author them; marketplace
> reviewers gate what gets listed.

| | |
|---|---|
| **Module code** | MKT (`FR-MKT-###`) |
| **Owning service** | Marketplace Service |
| **Tier** | 3 |

## In plain terms
- Browse a searchable catalog of apps filtered by category (safety, quality, opening/closing,
  food safety, …), industry vertical, rating, and whether your plan tier qualifies.
- Preview exactly what an install will create in your tenant — every template, the default
  workflow ("runs daily at 8:00 AM, assigned to Store Manager role"), and any connector that
  needs credentials later — via a dry run that writes nothing.
- Install in one click: the bundle becomes ordinary tenant-owned templates and workflows (never a
  shared reference back to the marketplace master), fully editable like hand-built content.
- Stay in control of updates: new app versions show an "update available" badge, and pulling an
  update is always opt-in — tenant customizations are never silently overwritten; conflicting
  changes go through a side-by-side reconciliation view.
- Publish apps through a review pipeline: DRAFT → SUBMITTED → IN_REVIEW → APPROVED → LISTED,
  with structured rejection reasons, review SLAs, and a sandbox "preview install" test before
  anything reaches the catalog.

## Key concepts
| Concept | What it means |
|---|---|
| `AppListing` | The stable catalog "product page" identity for an app — name, publisher, category, aggregate rating — like a `Template` is to its versions. |
| `AppVersion` | One immutable, versioned snapshot of the bundle (templates + optional workflow/schedule + optional connector config) with its own publishing lifecycle. |
| Bundle item | One constituent of an `AppVersion`: a template definition, a workflow/schedule definition, or a credential-free connector config template. |
| `AppInstallation` | The per-tenant record of installing a specific `AppVersion`, with pointers to every tenant-owned object the install created. |
| Materialization | The install step that turns bundle items into real, independent, tenant-owned rows via the owning modules' own APIs — the marketplace never keeps runtime control. |
| `Publisher` | The authoring identity (internal product team or vetted partner) that owns listings and submits versions for review. |

## How it connects to the rest
- **Forms & Workflow Builder (module 03)** — an "app" is a packaged set of `TemplateVersion`s plus
  a Workflow; this module packages and installs them but never authors form fields or re-implements
  the builder. On install it calls Forms' template-creation API, and the resulting templates are
  owned by Forms from that instant (the module's central guarantee: copy-on-install, then hands off).
- **Work Orchestration (module 02)** — the default `Schedule` on an installed app becomes ordinary
  WorkItem scheduling; this module never executes or schedules work itself.
- **Integrations & Extensibility (module 12 / EPIC-13)** — a bundled connector installs as a
  tenant-owned `Integration` shell in `NOT_CONFIGURED` state; live credentials are always supplied
  by the tenant through the Integration Hub's own setup wizard, never through this module.
- **Identity & Organization (module 01)** — install/uninstall/publish permissions reuse the
  existing role/scope model, and installs always target an `OrganizationNode` scope.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-MKT-001–072 covering the app package format, publishing pipeline, browse/discovery, install (materialization), update/uninstall flows, and ratings. |
| [high-level-design.md](./high-level-design.md) | The catalog + packaging + install-orchestration boundary, component diagram, and the "copy-on-install is the central design point" multi-tenancy section. |
| [low-level-design.md](./low-level-design.md) | Schema and ER diagram, publisher/reviewer/tenant REST endpoints, install and upgrade sequence diagrams, and materialization pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Component diagram spanning publisher and tenant sides, submission-review-listing and install sequence diagrams, and the publishing pipeline state machine. |
