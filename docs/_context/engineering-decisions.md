# Engineering Decisions Register — Cross-Cutting, Binding for All Modules

This document exists to remove ambiguity. Every module's functional-spec/HLD/LLD/architecture-diagram
must be read **together with this register**: where a module document is silent on something covered
here (auth mechanics, versioning, environments, testing gates, coding standards), this register's
answer is the answer — a module document never needs to repeat it, and a module document must never
contradict it. Where a genuine per-module research/decision item existed as an "Open Question" or
"Decision needed," it has been resolved in that module's own file directly (see each module's
"Resolved Decisions" section, which replaces the old "Open Questions/Assumptions" heading) — this
register only covers decisions that apply platform-wide.

Every decision below states the choice, the concrete parameters, and the rationale, so a coding agent
can implement it without asking a follow-up question. Where a decision differs from what a reasonable
engineer might guess, that is called out explicitly.

## 1. AuthN/AuthZ

- **Protocol**: OAuth 2.1 + OpenID Connect. Web Admin Console uses Authorization Code + PKCE. Mobile
  app uses Authorization Code + PKCE with the native app as a public client (no client secret, per
  OAuth 2.1's removal of the implicit grant and mandatory PKCE for all public clients).
- **Token sender-constraining**: Access tokens are DPoP-bound (RFC 9449) — the mobile app and web
  admin each generate a device/session-bound key pair at login; every API call includes a `DPoP` proof
  header. This is deliberately chosen over mTLS for the client leg (mTLS is used internally, see
  below) because DPoP does not require client certificate provisioning/rotation UX on consumer mobile
  hardware, per RFC 9700's guidance that public clients "MUST be sender-constrained or use refresh
  token rotation" — we do both, belt-and-suspenders, because the offline grace window (below)
  increases the value of a stolen refresh token if it were the only control.
- **Token lifetimes**: Access token TTL = 15 minutes. Refresh token TTL = 90 days, **rotating**
  (single-use; each refresh issues a new refresh token and immediately invalidates the old one; reuse
  of an already-rotated refresh token revokes the entire token family and forces re-authentication —
  standard rotation-detection per RFC 9700).
- **Offline grace window**: A device that cannot reach the Authorization Server (offline in the field)
  continues to use its last-issued access token past its 15-minute TTL **for read/render and local
  writes only** (queued, not yet synced) for up to **72 hours**, tracked by a signed, device-local
  "last successful auth" timestamp the app itself cannot forge (the JWT's own `iat`, verified against
  device-local monotonic clock, not wall-clock, to resist clock manipulation). At 72 hours offline, the
  device moves to a **read-only/no-new-fills** state until it reconnects and re-authenticates — this
  is a deliberate, tenant-non-configurable ceiling (see Security/Compliance module), because a lost or
  stolen device that stays offline indefinitely must not retain indefinite write capability.
- **Internal service-to-service (Kotlin/Spring Boot microservices)**: mTLS via a service mesh
  (Linkerd or Istio — either is acceptable, mesh choice is an infra-team call, not a spec ambiguity)
  for transport identity, plus a short-lived (5 min) internal JWT asserting the calling service's
  identity and the originating tenant/user context (so downstream RLS/OPA checks still have the real
  principal, not just "some internal service called me").
- **Web Admin session**: standard OIDC session, 12-hour absolute session TTL, sliding 30-minute idle
  timeout, step-up re-auth (password or WebAuthn) required for: raising an agent's autonomy level,
  changing an approval route, exporting PII-bearing data.
- **Password/credential policy**: Passwordless preferred (WebAuthn/passkeys) for the Web Admin
  Console; where password auth is retained for a tenant's SSO-less users, NIST 800-63B guidance
  applies (no forced periodic rotation, breached-password check against a k-anonymity API, minimum 12
  characters, no composition rules).
- **Identity Provider**: platform runs its own OIDC provider (Keycloak, self-hosted) as the default,
  with SAML/OIDC federation to a tenant's own IdP (Okta, Azure AD, Google Workspace) for Enterprise
  tier — this is already implied by Identity & Organization module's SSO requirements; this register
  just names the concrete default IdP technology so there is no ambiguity for a build team.

## 2. Mobile Device & OS Support Matrix

- **Android**: minimum **API 29 (Android 10)**, current + 5 prior major OS releases supported at any
  time (rolling window, re-evaluated at each annual planning cycle). Minimum hardware: **quad-core
  CPU, 2GB RAM** — deliberately lower than a premium field-service app's typical floor (e.g. 4GB RAM
  hex-core devices are common in developed-market field service tooling) because this platform's
  explicit differentiator is low-end-device performance for frontline/emerging-market deployments
  (source-spec §23, FR-FORM-062/NFR-FORM-04). Devices with 2–3GB RAM automatically render in the
  already-specified "Tier 2 Device" degraded mode (lazy-evaluated sections, reduced animation,
  disabled non-essential background sync) rather than being blocked from installing.
- **iOS**: minimum **iOS 16**, current + 2 prior major releases supported (Apple's own adoption curve
  makes a 5-version window unnecessary — iOS updates reach >90% of active devices within 12 months).
  Minimum hardware: iPhone 8 / A11 Bionic equivalent.
- **App freshness**: the mobile client enforces a **soft update prompt** at 60 days since a device's
  installed version was superseded, and a **hard update block** (app remains usable for already-cached
  offline work, but new template-schema fetches and submission sync are refused with a clear message)
  at 180 days, to bound how many client-side rule-evaluation-engine versions must be supported
  concurrently server-side (this module already promises identical on-device/server evaluation
  semantics, low-level-design.md §7.1 of Forms & Workflow Builder — an unbounded tail of ancient client
  versions would make that guarantee unmaintainable).
- **Web Admin Console**: last 2 major versions of Chrome, Edge, Safari, Firefox (evergreen browsers,
  auto-update assumed) — no IE/legacy support, no polyfill budget spent below this line.

## 3. API Versioning & Deprecation

- URI versioning, `/api/v1/` as already established (conventions.md). A breaking change ships as
  `/api/v2/` on the same service; both versions run concurrently.
- A deprecated version is supported for a **minimum 12 months** from the day `/vN+1` ships, advertised
  via `Deprecation` and `Sunset` HTTP response headers (RFC 8594) on every `/vN` response once its
  successor exists, plus an in-product admin banner for any tenant whose integration is still calling
  the deprecated version (detected via access logs, not guesswork).
- Event schemas (Kafka) version via a schema registry (Confluent Schema Registry, or Buf for
  Protobuf-based events if the team prefers Protobuf over the currently-specified JSON envelope —
  either is acceptable; JSON + registry-enforced schema validation is the default per conventions.md's
  existing canonical envelope). Schema evolution must be backward-compatible (new optional fields
  only) within a major event version; a breaking event change is a new `event_type` version suffix
  (e.g., `submission.created.v2`), never a silent field-meaning change on the same type.

## 4. Repository Structure

- **Polyrepo**, one repo per deployable service/app, plus one shared repo:
  - `platform-contracts` — OpenAPI 3.1 specs for every service's REST API, JSON Schema for every Kafka
    event envelope/payload, and generated client SDKs (Kotlin, TypeScript) published as versioned
    packages. Every service and the mobile/web clients depend on this repo's published artifacts,
    never on hand-copied types. This is the single source of truth that keeps 25 modules'
    documented contracts from drifting once code exists.
  - One repo per microservice (`svc-work-orchestration`, `svc-forms`, `svc-identity`, … one per module
    that owns a service per high-level-design.md's "Module Responsibilities").
  - `mobile-app` — one React Native app (not one app per module — the frontline associate uses one
    app; modules are internal feature boundaries, not separate installs).
  - `web-admin` — one web app (Next.js/React) serving the Builder, Control Tower, and all admin
    surfaces across modules.
  - `infra` — Terraform (cloud resources) + Helm charts + ArgoCD Application manifests (GitOps).
- Rationale for polyrepo over monorepo: 25+ independently-scaled Kotlin services with very different
  release cadences (a Form Service schema-compiler change vs. an IoT ingestion tuning change) benefit
  more from independent CI pipelines and independent deploy cadence than from atomic cross-service
  commits, which are rare in this system because cross-service contracts already flow through
  `platform-contracts` and versioned events, not shared code.

## 5. CI/CD

- **CI**: GitHub Actions per repo. Every PR runs: lint (ktlint+detekt for Kotlin, ESLint+Prettier for
  TS), unit tests, a contract-diff check against `platform-contracts` (fails the build on an
  undeclared breaking change), and a Testcontainers-based integration test suite (real Postgres/Kafka
  in ephemeral containers, not mocks, for anything touching persistence or eventing).
- **CD**: container images built in CI, pushed to a private registry, deployed via ArgoCD (GitOps —
  the desired state lives in the `infra` repo, ArgoCD reconciles the cluster to match). `main` branch
  auto-deploys to **dev**. Promotion to **staging** is a manual ArgoCD sync triggered by a tag.
  Promotion to **prod** requires one additional human approval and deploys via canary: 10% of traffic
  for 15 minutes, auto-promote to 100% if the service's own SLO burn-rate alerts (per its NFR table)
  stay green, auto-rollback on breach.
- **Mobile release**: Fastlane automates build/sign/upload. Internal test track auto-publishes on
  every `main` merge. Public release is a staged rollout (10% → 50% → 100%, each stage held a minimum
  24 hours, promoted manually) via Play Console/App Store Connect staged-rollout features, not a
  single atomic release to 100% of the fleet — a bad build caught at 10% is a minor incident, at 100%
  it is a field-wide outage of frontline data capture.
- **Web admin release**: standard blue/green behind the same ArgoCD/K8s pipeline as backend services.

## 6. Environments

- **dev**: shared cluster, ephemeral per-PR preview namespaces (spun up on PR open, torn down on
  merge/close) for any service whose change needs live-integration verification beyond CI's
  Testcontainers suite.
- **staging**: one persistent, prod-topology-equivalent environment per major release train.
  Populated with synthetic tenants and anonymized/synthetic data only — **never** a copy of real
  tenant data, even redacted (this is a hard rule, not a preference, given the PII/health/safety data
  categories several modules handle — Employee Engagement, EHS, HACCP).
- **prod**: multi-region, tenant-tier sharding per conventions.md. A given tenant's data plane lives
  in exactly one region (data residency), selected at tenant-provisioning time based on the tenant's
  declared jurisdiction (EU tenants -> EU region, by default, to simplify GDPR/EU AI Act residency
  questions rather than architecting cross-region data-residency exceptions).

## 7. Testing Strategy and Gates

- **Unit tests**: minimum 80% line coverage on domain/business-logic packages (rules engines, state
  machines, policy evaluation) — explicitly excluding generated code (OpenAPI clients, DB migration
  scaffolding). Coverage gate enforced in CI, not aspirational.
- **Contract tests**: every service publishes its OpenAPI spec to `platform-contracts`; every consumer
  (mobile, web, another service) runs a contract-verification test against the latest compatible
  version on every CI run (Pact-style consumer-driven contracts, or an OpenAPI-diff gate — either
  satisfies this; OpenAPI-diff is the lower-effort default given `/api/v1/` is already
  schema-documented per module).
- **Integration tests**: Testcontainers-backed, real Postgres + Kafka + Redis per test suite run,
  covering at minimum every state-machine transition and every documented error code per module's
  low-level-design.md §Error Handling table — those tables are the test plan, not just documentation.
- **Mobile E2E**: Maestro (chosen over Detox for its declarative YAML flows and built-in
  network-condition simulation, which this platform needs constantly given the offline-first
  requirement) run against a device farm (Firebase Test Lab for Android, a macOS runner + iOS
  Simulator/App Store Connect device farm for iOS) on every release-candidate build, covering: full
  offline fill → reconnect → sync flow, the multi-level approval flow, and cross-template forwarding
  (both `INLINE_CHAIN` and `SPAWN_WORK_ITEM` modes) as the three canonical "hard" flows.
- **Load testing**: k6 scripts per module, run against staging before any release that touches a
  module's hot path, targeting the exact NFR numbers already specified in that module's functional
  spec (e.g., Form Service NFR-FORM-04/05, Work Orchestration's assignment-latency NFR) — a release
  that regresses a documented NFR fails the release gate, it does not ship with a "we'll fix it later"
  note.
- **Chaos testing**: quarterly, scripted (Kafka broker kill, Postgres primary failover, Redis
  eviction storm) against staging, verifying every module's own "Failure Modes and Degradation" table
  holds under actual fault injection, not just on paper.
- **Accessibility**: Web Admin Console tested against WCAG 2.1 AA (automated via axe-core in CI, plus
  a manual screen-reader pass before each quarterly release) — Enterprise Admin/Operations Admin
  personas include users who may require assistive technology, and several tenants (public sector,
  large enterprise) will contractually require this.

## 8. Secrets, Config, and Data Protection

- **Secrets**: HashiCorp Vault (or an equivalent cloud KMS + External Secrets Operator combination —
  Vault is the default reference implementation). No secret ever committed to a repo or baked into a
  container image; short-lived dynamic database credentials issued per-service via Vault's database
  secrets engine where supported (Postgres), rotated automatically.
- **Config**: 12-factor — environment-specific config via K8s ConfigMaps (non-secret) and
  Vault-injected Secrets (secret), never hardcoded per-environment branches in application code.
- **Feature flags**: a flag service (self-hosted Unleash, or a vendor — implementation-agnostic) gates
  every autonomy-level increase (Agent Runtime), every new module's tenant-level enablement, and every
  risky rollout (e.g., a new forwarding mode) — flags are the mechanism for the staged-rollout pattern
  already specified per-module (e.g., Forms & Workflow Builder's Pilot rollout, FR-FORM-045), not a
  separate ad hoc mechanism per module.
- **Encryption**: TLS 1.3 in transit everywhere (external and internal, per the mTLS decision above);
  AES-256 at rest for all Postgres volumes and object storage (Evidence media); envelope encryption
  with a per-tenant Data Encryption Key (DEK) wrapped by a platform Key Encryption Key (KEK) in the
  cloud KMS, for `SENSITIVE`-classified `pii_class` fields (Forms module §3.0) and for EHS/HACCP
  incident records specifically — this is the concrete mechanism behind every module's "encryption at
  rest" NFR line, stated once here instead of repeated 25 times with room to drift.

## 9. Observability

- **Tracing/metrics/logs**: OpenTelemetry SDK instrumented in every service (Kotlin) and in the
  mobile/web clients (traces for critical user flows: fill → submit, approval decision, forward
  action). Traces exported to a tracing backend (Tempo/Jaeger — implementation-agnostic), metrics to
  Prometheus, scraped/federated into Grafana for dashboards built directly from each module's NFR/SLO
  tables (one Grafana dashboard per module, generated from the same numbers already in this spec).
- **Logs**: structured JSON, every log line carries `trace_id`/`tenant_id`/`request_id` for
  correlation, shipped to a central log store (Loki or ELK — implementation-agnostic) with the
  retention policy from §Security/Privacy (module 13) applied per log category.
- **Mobile crash/ANR**: Sentry (or Firebase Crashlytics) — mandatory, wired in from the first mobile
  release, because a crash on a low-end device in the field is otherwise invisible to the platform
  team entirely.
- **Alerting/on-call**: Prometheus Alertmanager -> PagerDuty (or Opsgenie), alert thresholds derived
  directly from each module's NFR/SLO table (an SLO breach files a page, not a "check the dashboard
  sometime" ticket).

## 10. Coding Standards

- **Kotlin**: official Kotlin style guide, enforced via `ktlint` (formatting) + `detekt` (static
  analysis) in CI as a hard gate, not a warning. Spring Boot 3.x, Java 21 LTS runtime.
- **TypeScript** (React Native + Next.js web admin): `strict: true` in `tsconfig.json` platform-wide,
  ESLint (`airbnb-typescript` base config + a small platform ruleset for offline-safety lint rules,
  e.g., banning a direct network call outside the designated sync layer) + Prettier, enforced in CI.
- **Commit convention**: Conventional Commits (`feat:`, `fix:`, `refactor:`, …), used to drive
  automated changelog generation per service.
- **Branching**: trunk-based development, short-lived feature branches (target <3 days to merge),
  incomplete work ships behind a feature flag rather than a long-lived branch. No permanent
  per-release branches — releases are tags off `main`.
- **Code review**: minimum 1 approving review + green CI (including the contract-diff gate) required
  to merge, enforced by branch protection, no exceptions for "small" changes to a published API
  contract (a small change to a public contract is exactly the class of change this gate exists for).

## 11. Definition of Done (per user story, all modules)

A story is not "done" until all of the following hold, not merely "code merged":
1. Code merged to `main` (behind a feature flag if the containing module/feature is not yet fully
   rolled out), passing every CI gate in §5/§7 above.
2. If the story touches a REST endpoint or event: `platform-contracts` updated and published in the
   same change set, never as a follow-up.
3. Unit + contract + relevant integration tests green; a new documented error code (per that module's
   Error Handling table) has an automated test proving it is actually returned under the stated
   condition.
4. If the story is perf-sensitive (anything on a module's NFR table): a k6 load-test result attached
   to the PR showing the target is met, not merely "should be fine."
5. If the story touches client-facing behavior: offline behavior explicitly verified (what happens
   with no connectivity, what happens on reconnect) — "didn't think about offline" is not an
   acceptable state for this platform given its core positioning.
6. If the story touches a mutating action: an audit-log entry is verifiably produced (per that
   module's event/audit requirements) — checked in the integration test, not assumed.
7. If the story touches the Web Admin Console: WCAG 2.1 AA automated check passes.
8. If the story's behavior deviates in any way from what this spec package currently documents (a
   necessary implementation-time adjustment is normal and expected): the corresponding
   functional-spec.md/low-level-design.md is updated in the same PR, so the spec and the shipped
   system never silently diverge. This is the single most important rule in this list — a build that
   quietly drifts from its own spec defeats the purpose of having written one.

## 12. AI Governance — EU AI Act Alignment (Binding, Not Tenant-Configurable Downward)

Employment/worker-management AI (task allocation, performance monitoring, promotion-relevant
evaluation) is classified **high-risk** under the EU AI Act, Annex III §4, with obligations enforceable
from August 2, 2026 (Articles 9, 12, 13, 14). Several of this platform's agents and features fall
squarely in that category for any EU tenant: Compliance Monitor and Resolution Agent (module 10) when
their output concerns a specific employee's performance/compliance pattern, Gamification's
leaderboards/scoring (module 22) if used in a performance-review context, and Employee Engagement's
`EngagementScore` (module 18) if surfaced to a manager at the individual level (it structurally is not
— see that module's N=5 aggregation threshold — which already keeps it out of this category by
design).

Rather than building a region-conditional compliance path, the platform implements these controls
**globally, as defaults**, because (a) it is the safer and simpler engineering posture for a
multi-tenant system where a tenant's jurisdiction can be ambiguous (a global enterprise with EU
employees using a non-EU-provisioned tenant), and (b) every one of these controls is good practice
regardless of jurisdiction:

- **Article 14 (human oversight)**: hard rule, added to module 10 functional-spec.md — any agent
  output that would materially affect a **named individual employee's** performance record, standing,
  or continued employment (as opposed to a location/process-level recommendation) **always** requires
  human approval before taking effect, regardless of that agent's configured autonomy level. This is
  not a tenant-configurable-downward setting; an Enterprise Admin can add more scenarios requiring
  approval, never remove this one.
- **Article 12 (logging)**: already satisfied structurally by the `AgentRun`/`AgentToolExecution`
  audit trail (module 10 §3, stage 11) — this register fixes the retention floor at **minimum 3
  years** for any audit record touching an individual-employee-affecting decision (longer than the
  general audit-log retention elsewhere in the platform), matching common labor-record retention norms
  and giving headroom under GDPR's own retention-minimization principle by keeping only the decision
  record, not the full context payload, past 1 year.
- **Article 13 (transparency)**: **new functional requirement**, added to module 10 — every tenant
  must expose an "AI in this workplace" notice, reachable from the mobile app's account/settings
  surface, listing every active agent, what it does, and a contact path to raise a concern or request
  human review of a specific agent decision. This is a hard requirement, not admin-optional, for any
  tenant with EU-based employees, and is built as a platform default rather than a per-tenant toggle
  per the rationale above.
- **Article 9 (risk management)**: **new functional requirement**, added to module 10's Agent
  Governance surface — a documented per-agent risk-assessment artifact (a structured record, not free
  text: intended purpose, affected-persona categories, known failure modes, mitigation, human-oversight
  mechanism, last-reviewed date), required before an agent may be enabled for a tenant and re-reviewed
  at minimum annually or on any autonomy-level increase.

These four bullets are the authoritative addition; module 10's functional-spec.md carries them as
numbered FR items (see that module's own file) so they are traceable and testable like every other
requirement in this package, not left as prose only in this register.
