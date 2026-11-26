# Integrations & Extensibility (Integration Hub) — Functional Specification

Module: `12-integrations-extensibility`
Epic: EPIC-13 (Integration Hub)
Source: `source-spec.md` §15 (Integrations and Extensibility, all subsections), §18.1 (API
Conventions incl. webhooks), §24.1 (Integration recovery acceptance test), Appendix E EPIC-13

## 1. Purpose

This module is the platform's boundary with the rest of the enterprise's software estate. It owns
the **Integration Hub**: a configuration-first experience for connecting standard third-party
systems (identity, HR, ERP, data/BI, collaboration, operational systems) and a governed API/SDK
surface for custom integrations that partner developers and enterprise IT build themselves.

Every integration configured through this module — regardless of connector type or transfer
mechanism — carries the same governance envelope described in source-spec §15.1: **connection
state, owner, scope, secret rotation, rate limits, monitoring, retry policy, and audit logging**.
This module is the single place that envelope is implemented; individual connector adapters plug
into it rather than reinventing credential handling, retry, or audit logging per connector.

This module does **not** implement business logic for the systems it connects to. It moves data and
events in and out of the platform reliably, idempotently, and observably, and it exposes the
platform's own domain events and data to the outside world under tenant-scoped authorization.

## 2. Scope

### 2.1 In scope

- **Connector catalog & configuration**: a registry of supported connector types (HR, ERP, Data/BI,
  Collaboration, Operational systems) with a per-connector setup wizard: credential/OAuth entry,
  scope selection (which entities/fields/locations sync), field mapping, sync direction (inbound,
  outbound, bidirectional), and schedule (real-time/event-driven vs. batch).
- **Webhook subscription management**: tenant/partner-configured subscriptions to canonical
  platform events (source-spec §18.3), including event-type selection, target URL, signing-secret
  generation and rotation, retry policy configuration, delivery log, and delivery replay.
- **Inbound API/SDK surface**: service-credential-based (OAuth2 client-credentials) API access for
  custom integrations, with scoped permissions, tenant-aware rate limits, and an SDK (TypeScript/
  Java/Python) wrapping the REST surface with typed models and built-in retry/idempotency helpers.
- **Batch transfer (SFTP/CSV)**: configuration and scheduling of SFTP pickup/drop-off jobs and CSV
  batch ingestion/export, with validation, field mapping, and a job run history.
- **Connection health monitoring & alerting**: per-integration health state, error-rate/latency
  dashboards, and configurable alerting (email/Slack/Teams/PagerDuty-class) on connector degradation
  or failure.
- **Secret rotation workflow**: scheduled and on-demand rotation of connector credentials and webhook
  signing secrets, coordinated with the platform's central secret manager (see conventions.md).
  Secrets themselves are never stored in this module's database — only references (vault path/key
  id, rotation metadata).
- **Per-connector audit logging**: every configuration change, credential rotation, sync run, and
  data push/pull is recorded in `AuditLog` with before/after state, actor, and trace_id.
- **Outbound data delivery** to Data/BI/Collaboration/Operational-system targets driven by the
  platform's own domain events (via the Event Backbone) or by scheduled BI extracts.
- **Inbound data ingestion** from HR/ERP/Operational systems (roster sync, org/location reference
  data, product/asset master data) that other modules (primarily Identity & Organization, Work
  Orchestration) consume.

### 2.2 Out of scope

- **SAML/OIDC/SSO and SCIM identity provisioning** — owned by Identity & Organization (module
  `01-identity-organization`, EPIC-02). This module references the same connector-registry pattern
  and appears in the connector catalog UI for discoverability (an admin sees "Identity" alongside
  "HR"/"ERP" in one catalog), but the SSO session/SCIM user-lifecycle logic, its schema, and its
  provisioning conflict rules are owned and specified by Identity & Organization. Where an
  HR-connector and SCIM both attempt to provision the same user, the conflict-resolution policy is
  defined jointly (see §8, "SCIM vs HR-connector provisioning conflict") but implemented in
  Identity & Organization's user-lifecycle state machine; this module only supplies the HR-side sync
  events and defers to Identity's resolution.
- **Agent Runtime tool execution** (EPIC-11) — an agent's outbound notification or tool call is not
  a "webhook" in this module's sense; Agent Runtime owns its own tool-execution audit trail. This
  module's webhooks are for external systems subscribing to platform domain events.
- **Notification Hub push/email/SMS delivery** — end-user notifications are a separate concern from
  system-to-system webhook delivery, even though both share retry/dedupe patterns.
- **Analytics warehouse internals** (EPIC-15) — this module owns the BI/Data connector configuration
  and the extract/export mechanism; the warehouse schema, metric definitions, and ML pipelines are
  owned by Analytics & Control Tower.
- **Billing/commercial system integrations** — not part of the customer-facing Integration Hub;
  handled by internal commercial tooling.

## 3. Personas served

| Persona | Relationship to this module |
|---|---|
| Enterprise Admin | Primary configurator: browses the connector catalog, runs setup wizards, defines field mappings and scopes, manages webhook subscriptions, rotates secrets, reviews connection health and audit logs. |
| Operations Admin | Consumes synced data for operational configuration — e.g., an HR roster sync populates candidate `User`/`OrganizationNode` records that Operations Admin reviews and activates; consumes ERP-sourced product/asset reference data used in form lookups. |
| External system (API/webhook consumer) | A third-party system (BI tool, ERP, ticketing system, SFTP endpoint) that receives webhook deliveries or SFTP/CSV files, or calls the inbound REST API with a service credential. Not a human persona but a first-class actor with its own reliability/security contract. |
| Partner developer | Builds a custom integration using the SDK and service-credential auth; consumes API documentation, sandbox/test-mode credentials, and webhook event catalog. |
| Enterprise IT / Security reviewer | Reviews connector scopes, credential rotation history, and audit logs as part of vendor security review and periodic access recertification. |

## 4. Functional Requirements

### 4.1 Connector catalog & configuration

- **FR-INT-001**: The system shall present a connector catalog grouped by category (Identity
  [cross-reference only, links to Identity & Org module], HR, ERP, Data, BI, Collaboration,
  Operational Systems, Custom/API), each entry showing connector name, category, supported sync
  directions, and configuration status (`NOT_CONFIGURED`, `CONFIGURED`, `DEGRADED`, `DISABLED`).
- **FR-INT-002**: Selecting a connector shall launch a per-connector setup wizard with steps:
  (1) authentication (OAuth2 authorization-code flow for SaaS connectors, API key/basic auth for
  legacy systems, SFTP key-pair for batch connectors), (2) scope selection (which `OrganizationNode`
  subtree, which entity types, which fields), (3) field mapping (source field → platform field, with
  transform functions: trim, date-format, enum-remap, concatenate, constant), (4) sync direction and
  schedule (real-time/event-driven, polling interval, or batch cadence), (5) review & test
  connection, (6) activate.
- **FR-INT-003**: The wizard shall support a "test connection" step that performs a scoped read-only
  call (or a signed test file drop for SFTP) against the external system and reports success/failure
  with the raw error detail before allowing activation.
- **FR-INT-004**: Field mapping shall support required-field validation: an integration cannot be
  activated if a platform-required field has no source mapping and no default/constant value
  configured. Validation errors are surfaced inline per field (see §7, field-mapping validation
  errors state).
- **FR-INT-005**: An Enterprise Admin shall be able to assign an **owner** (a `User`) and a
  **scope** (an `OrganizationNode` subtree) to every configured `Integration`. The owner receives
  health/failure alerts by default; scope bounds which tenant data the connector may read/write.
- **FR-INT-006**: The system shall support disabling (pausing) an integration without deleting its
  configuration, and reactivating it later with the same field mappings and credentials (subject to
  credential-expiry re-validation).
- **FR-INT-007**: Deleting an integration shall be a two-step confirmation (type-to-confirm the
  connector name) and shall revoke the associated credential reference in the secret vault
  immediately; historical audit log entries and delivery logs are retained per tenant retention
  policy, not deleted with the integration.

### 4.2 HR / ERP / Data / BI / Collaboration / Operational connectors

- **FR-INT-010**: HR-class connectors (Workday/SuccessFactors/BambooHR-class) shall support inbound
  sync of worker records (name, employee id, work email, job title, department, location/cost-center
  code, employment status, hire/termination date) mapped to candidate `User` and `OrganizationNode`
  assignment records, staged for Operations Admin review before activation (see §4.7 conflict
  resolution) unless the tenant has configured auto-activation for a given field-mapping profile.
- **FR-INT-011**: ERP-class connectors (SAP/Oracle/Dynamics-class) shall support inbound sync of
  location/site master data, product/asset master data, and cost-center hierarchies, and outbound
  push of operational facts (e.g., completed maintenance work orders, verified inventory counts) to
  the ERP as configured by field mapping.
- **FR-INT-012**: Data-platform connectors (Snowflake/BigQuery/Databricks-class) shall support
  scheduled or CDC-style outbound extract of platform fact/dimension tables (work items, submissions,
  issues, evidence metadata — never raw evidence binaries by default) into a tenant-owned external
  schema/dataset, using the tenant's own connection credentials (the platform never stores a
  standing copy of the tenant's warehouse credentials beyond the vault reference).
- **FR-INT-013**: BI connectors (Power BI/Tableau-class) shall support either (a) a direct query
  connector against the Analytics module's semantic layer using a scoped read-only service account,
  or (b) a scheduled flat extract (CSV/Parquet to the tenant's storage) for BI tools that cannot
  query live. BI connector scope is capped to `OrganizationNode` subtrees the configuring admin
  themselves has read scope over.
- **FR-INT-014**: Collaboration connectors (Teams/Slack-class) shall support outbound notification
  delivery (mirroring a subset of Notification Hub channels) and, optionally, inbound slash-command/
  interactive-message actions (e.g., acknowledge an announcement, approve an escalation) that are
  translated into platform API calls under the invoking user's mapped identity.
- **FR-INT-015**: Operational-system connectors (POS, inventory, workforce management, maintenance
  systems) shall support both inbound (sales/inventory facts feeding risk scoring and form lookups)
  and outbound (work orders, maintenance tickets created from platform Issues pushed to the external
  maintenance system) directions, each independently configurable.

### 4.3 Webhook subscriptions

- **FR-INT-020**: The system shall allow an Enterprise Admin or partner developer (via API) to create
  a `WebhookSubscription` specifying: target URL (HTTPS only), one or more subscribed event types
  from the canonical event catalog (source-spec §18.3), an optional filter expression (e.g.,
  `location.region == "APAC"`), and a retry policy profile (default or custom).
- **FR-INT-021**: Every `WebhookSubscription` shall be issued a signing secret at creation. The
  system shall sign every delivery payload using HMAC-SHA256 over the raw request body and deliver
  the signature in a header (see low-level-design.md §3 for the exact header/format), so the
  consumer can verify authenticity without a shared session or additional round trip.
- **FR-INT-022**: The system shall support **secret rotation** for a webhook subscription with a
  dual-secret grace window: both the old and new secret validate incoming-signature-equivalent
  outgoing signatures for a configurable overlap period (default 24 hours) so the consumer can roll
  over without a delivery gap.
- **FR-INT-023**: The system shall maintain a **delivery log** per subscription showing, for each
  attempted delivery: event id, event type, timestamp, HTTP status returned, attempt number,
  duration, and outcome (`DELIVERED`, `FAILED`, `RETRY_SCHEDULED`, `DEAD_LETTERED`).
- **FR-INT-024**: An admin shall be able to **replay** a specific failed or dead-lettered delivery
  from the delivery log. Replay shall reuse the original event payload and event id, and the
  delivery must be constructed so that a consumer applying idempotent processing (keyed on event id)
  does not create duplicate downstream work — this directly satisfies the source-spec §24.1
  "Integration recovery" acceptance test.
- **FR-INT-025**: The system shall support pausing and resuming a webhook subscription without
  losing queued/pending deliveries (paused deliveries remain `PENDING` and resume delivery attempts
  once unpaused, respecting their original retry schedule baseline).

### 4.4 Inbound API/SDK surface

- **FR-INT-030**: The system shall issue **service credentials** (OAuth2 client-credentials grant:
  `client_id` + `client_secret`, or a signed JWT for higher-security tenants) scoped to specific API
  permission scopes (e.g., `work_items:read`, `issues:write`, `evidence:read`) and an
  `OrganizationNode` subtree, independent of any human user session.
- **FR-INT-031**: Service credentials shall be tenant-aware and principal-aware rate limited per
  source-spec §18.1, with limits configurable per tenant tier and visible to the admin as a quota
  dashboard (requests used / limit / reset window).
- **FR-INT-032**: The platform shall publish an SDK (TypeScript, Java, Python at minimum) that wraps
  the REST API with typed request/response models, automatic `Idempotency-Key` generation for unsafe
  methods, automatic retry with exponential backoff and jitter on `429`/`5xx`, and built-in webhook
  signature verification helpers.
- **FR-INT-033**: The system shall provide a sandbox/test-mode tenant or test-mode flag for service
  credentials so partner developers can exercise the API and receive synthetic webhook deliveries
  without touching production tenant data.

### 4.5 SFTP/CSV batch transfer

- **FR-INT-040**: The system shall allow configuration of a `BatchJob` describing: direction
  (import/export), transfer mechanism (SFTP pickup/drop, HTTPS presigned-URL pickup/drop, or direct
  upload through the admin UI), file format (CSV with configurable delimiter/encoding/header row),
  field mapping (reusing the same mapping engine as real-time connectors), a schedule (cron
  expression or manual trigger), and a target/source entity type.
- **FR-INT-041**: Each execution of a `BatchJob` shall produce a `BatchJobRun` record with file
  name/checksum, row counts (total/accepted/rejected), a downloadable error report for rejected
  rows (row number, field, reason), and overall run status.
- **FR-INT-042**: Batch import validation shall run in two phases: (1) structural validation (column
  count, encoding, required columns present) that fails the entire file if violated, and (2)
  row-level validation (per source-spec Form validation patterns: required, type, cross-field) that
  fails only the offending rows while accepting valid rows, unless the tenant has configured
  all-or-nothing mode for a given job.
- **FR-INT-043**: The system shall support manual re-trigger of a failed or partially-failed
  `BatchJobRun` and shall allow re-processing of only the previously rejected rows via a corrected
  re-upload keyed to the original run.

### 4.6 Connection health, monitoring, secret rotation

- **FR-INT-050**: Every `Integration` shall expose a health state derived from recent delivery/sync
  outcomes: `HEALTHY`, `DEGRADED` (elevated error rate or rate-limited), `FAILING` (consecutive
  failures beyond threshold), `CREDENTIAL_EXPIRING` (secret/token expiry within a configurable
  window), or `DISABLED`. The catalog card and integration detail page always reflect the current
  state.
- **FR-INT-051**: The system shall support configurable alert routing (email, in-app notification to
  the integration owner, Slack/Teams/PagerDuty-class webhook) triggered on health-state transitions
  into `DEGRADED`, `FAILING`, or `CREDENTIAL_EXPIRING`.
- **FR-INT-052**: Secret rotation shall be schedulable (e.g., every 90 days) or on-demand, and shall
  never leave a window where zero valid credential exists — the vault-backed rotation workflow
  creates the new secret, validates it (test connection or a synthetic signed request), and only
  then revokes the old one, except where the external system requires manual coordination (in which
  case the UI walks the admin through a "rotate, don't revoke old until confirmed" flow).
- **FR-INT-053**: A `CREDENTIAL_EXPIRING` alert shall be raised at 30/14/3 days before known
  expiry (where the external system communicates expiry, e.g., OAuth refresh token or API-key TTL)
  and shall include a direct deep link to the rotation workflow.

### 4.7 Provisioning conflicts and orphaned references

- **FR-INT-060**: When both an HR connector and Identity & Organization's SCIM provisioning are
  configured for the same tenant, the system shall apply a deterministic precedence policy,
  configurable per tenant, with a documented default: **SCIM is authoritative for identity/session
  attributes (email, active/inactive status, SSO-linked identity)**; **the HR connector is
  authoritative for organizational attributes (job title, department, cost center, manager, location
  assignment)** — the two are merged per-field, not last-write-wins wholesale. A conflicting field
  write from the non-authoritative source for that field is logged as a `PROVISIONING_CONFLICT`
  audit event and discarded, never silently overwritten in the other direction.
- **FR-INT-061**: If an inbound ERP or HR sync references an `OrganizationNode` (location/cost
  center) that does not yet exist in the platform's org graph, the record shall be staged with an
  **orphaned-reference** flag rather than rejected outright, and surfaced in a reconciliation queue
  for the Operations Admin. Once the referenced node is created (or an admin manually maps the
  external code to an existing node), the queue reprocesses staged records automatically.
- **FR-INT-062**: Orphaned-reference records older than a configurable retention window (default 30
  days) shall trigger an escalating alert to the integration owner and, if still unresolved past a
  second threshold (default 90 days), shall be moved to a `STALE_ORPHAN` state excluded from active
  reconciliation but retained for audit.

## 5. UX States

| State | Trigger | UI behavior |
|---|---|---|
| Connector not yet configured | Catalog entry never completed setup wizard | Catalog card shows "Not configured" badge with a single "Set up" CTA; no health data shown. |
| Connection degraded / rate-limited | Elevated error rate, or external system returning 429s | Persistent banner on the integration detail page and a badge on the catalog card: "Connection degraded — see details"; links to the delivery/sync log filtered to failures. |
| Webhook delivery failing, needs attention | N consecutive delivery failures for a subscription (default N=5) | Subscription row shows a red "Delivery failing" badge; owner receives an alert; delivery log highlights the failing sequence with a "Replay" and "Pause subscription" action inline. |
| Secret rotation in progress | Rotation workflow started, new secret not yet confirmed | Integration detail shows "Rotation in progress" state; the rotate/delete actions are disabled until rotation completes or is cancelled; a visible countdown/step indicator (create → validate → cutover) is shown. |
| Field-mapping validation errors | Required platform field unmapped, or a mapped transform is invalid | Wizard step 3 blocks "Next" and shows inline errors per unmapped/invalid field; a summary banner lists total error count; "Save as draft" remains available so partial configuration is not lost. |
| Batch job partially failed | Row-level validation rejected some rows | Run detail shows accepted/rejected counts with a downloadable error CSV; status badge "Completed with errors" (not a hard failure) unless all-or-nothing mode was configured. |
| Orphaned references pending | Inbound sync referenced unknown org node | Reconciliation queue badge with count; each row offers "Map to existing node" or "Create node" actions. |
| Empty state — no integrations at all | New tenant, nothing configured | Catalog view with category tiles and a "Popular connectors" section; no health widgets rendered. |
| Loading | Fetching catalog/health data | Skeleton cards per category; does not block navigation. |
| Permission denied | Non-admin attempting to view/configure | Standard scope-denied pattern (per Identity & Organization module); integration owner and Enterprise Admin always retain access regardless of general org scope, since integrations are tenant-global objects with explicit ownership. |

## 6. Field-Level Behavior Notes

- Connector credentials entered in the setup wizard are transmitted directly to the backend over
  TLS and immediately written to the secret vault; they are never persisted in this module's
  database, request logs, or client-side state beyond the submission call. The database stores only
  a vault reference (`credential_ref`).
- Webhook signing secrets are shown in full exactly once at creation/rotation ("copy now, you won't
  see this again" pattern) and thereafter shown only as a masked suffix (last 4 characters) for
  identification.
- Field mapping transform functions are validated against sample data (first N rows of a test
  extract, or a synthetic payload for event-driven connectors) before the mapping can be saved as
  active, catching type-mismatch errors early.

## 7. Validation & Edge Cases

- **Webhook consumer endpoint down for an extended period**: after the configured retry budget is
  exhausted (see NFR-INT-002), the delivery is marked `DEAD_LETTERED`, written to a
  per-subscription dead-letter queue, and an alert is sent to the subscription owner. Dead-lettered
  deliveries remain replayable indefinitely (subject to the tenant's audit retention window) via
  FR-INT-024.
- **Duplicate webhook delivery due to replay must not create duplicate downstream work**: every
  webhook payload carries the original immutable `event_id`. The platform's own delivery engine
  guarantees at-least-once delivery (never exactly-once) by design — the acceptance test in
  source-spec §24.1 is satisfied by contract: consumers are required (and the SDK enforces this by
  default) to dedupe on `event_id` before acting. The delivery log itself also refuses to double-log
  a replay as a "new" delivery attempt sequence — a replay increments the existing delivery's
  attempt history rather than creating a second `WebhookDelivery` row for the same `(subscription_id,
  event_id)` pair. See low-level-design.md §6 for the idempotency-key/dedup pseudocode.
- **ERP sync producing orphaned references when a location doesn't exist yet**: handled by
  FR-INT-061/062 (staged orphan + reconciliation queue), not by hard rejection, so that a sync run
  is never blocked entirely by one bad or premature reference.
- **SCIM vs. HR-connector user-provisioning conflict**: handled by FR-INT-060 (field-level
  authority split, logged conflicts, no silent overwrite). This module emits `PROVISIONING_CONFLICT`
  audit events; Identity & Organization's user record is the system of record for the merged result.
- **Partial batch failure**: accepted rows commit; rejected rows are reported with reasons and never
  silently dropped (FR-INT-042).
- **Credential expiry mid-sync**: a sync run that fails partway through due to credential expiry is
  marked `FAILED` with the partial-completion boundary recorded (e.g., "records 1–4,200 of 10,000
  processed before failure") so a retry can be scoped to avoid reprocessing successfully-synced
  records where the source system supports incremental cursors; where it does not, the full run is
  retried and downstream idempotency (client_uuid/dedup keys) absorbs the reprocessing safely.
- **Rate limit exceeded on inbound API**: returns `429` with `Retry-After` header and the standard
  error envelope (`code: RATE_LIMITED`); the SDK's built-in backoff handles this transparently for
  SDK consumers.
- **Webhook target URL changes ownership** (e.g., admin repoints a subscription to a new URL that
  is actually a different, unrelated system): the system requires re-verification (a challenge/
  response handshake against the new URL) before resuming delivery, to prevent accidental data
  leakage to a URL no longer controlled by the intended consumer.
- **Clock skew in HMAC signature verification**: the signed payload includes a timestamp; the
  platform's own delivery does not depend on consumer clock, but the SDK's verification helper
  documents a recommended tolerance window (default ±5 minutes) for consumers implementing their
  own verification.

## 8. Non-Functional Requirements

- **NFR-INT-001 (webhook delivery latency)**: p95 webhook delivery attempt (event published on the
  Event Backbone → first HTTP delivery attempt dispatched) under 5 seconds under normal load.
- **NFR-INT-002 (retry budget)**: default retry policy is exponential backoff with jitter across a
  maximum of 8 attempts spanning approximately 24 hours (e.g., 30s, 2m, 10m, 30m, 1h, 3h, 6h, 12h)
  before dead-lettering; tenants may configure a shorter or longer budget within platform limits (min
  3 attempts / 5 minutes, max 15 attempts / 72 hours).
- **NFR-INT-003 (batch job SLA)**: a scheduled batch job shall start within 5 minutes of its
  scheduled time and complete within a per-job configurable SLA (default: 4x the historical p95 run
  duration, minimum 30 minutes) before triggering an SLA-breach alert.
- **NFR-INT-004 (inbound API availability/latency)**: inbound API surface shares the platform-wide
  99.9% availability and p95 < 300 ms targets (source-spec §23), measured separately from
  third-party outbound call latency (which this module cannot control).
- **NFR-INT-005 (secret isolation)**: no connector credential, webhook signing secret, or service
  credential secret is ever persisted outside the central secret vault, logged in plaintext, or
  included in any exported audit record (audit records reference `credential_ref`/rotation event
  metadata only, never secret values).
- **NFR-INT-006 (per-tenant rate isolation)**: a single tenant's integration traffic (inbound API
  calls, outbound webhook fan-out, batch job volume) cannot degrade another tenant's integration
  throughput — enforced via per-tenant rate limiting and worker-pool bulkheading (see
  high-level-design.md §Scalability).

## 9. Dependencies

- **Identity & Organization (`01-identity-organization`)**: owns SSO/SCIM identity integrations
  (cross-referenced, not implemented here); supplies `OrganizationNode` scope resolution used to
  bound every connector's data access; consumes HR-connector sync output for user/org provisioning
  and resolves SCIM-vs-HR conflicts per FR-INT-060.
- **Work Orchestration, Issues/CAPA, Evidence** (`02`, `06`, `05`): primary internal data sources for
  outbound integrations — completed work items, issues, and evidence metadata are the payloads
  pushed to ERP/BI/Data/Collaboration/Operational-system connectors and delivered via webhooks to
  external subscribers, all sourced from the canonical domain events those modules publish onto
  the Event Backbone.
- **Analytics & Control Tower (`11-analytics-control-tower`)**: BI connectors query or extract from
  the Analytics semantic layer/warehouse; this module owns the connector/credential/scheduling
  mechanics, Analytics owns the underlying metrics and schema being exposed.
- **Event Backbone (Kafka, per conventions.md)**: this module is the largest single consumer of the
  canonical event catalog (source-spec §18.3) for outbound webhook fan-out, and a producer of its
  own integration lifecycle events (`integration.connected`, `integration.degraded`,
  `webhook.delivery_failed`, etc. — see high-level-design.md).
- **Central Secret Manager (per conventions.md)**: all credential and signing-secret storage,
  rotation orchestration, and access-audit trails are delegated to the platform's secret manager;
  this module never becomes a secondary secret store.

## 10. Resolved Decisions

- **SCIM-vs-HR-connector field-authority split (FR-INT-060)**: SCIM owns identity/session attributes,
  HR owns organizational attributes, as the platform default for every tenant, and this is exposed as
  a per-tenant configurable policy from GA (not deferred to a post-pilot confirmation) so a tenant
  whose HR system is also its SSO IdP's attribute source can flip specific attributes' authority
  without a platform code change. Shipping the override as a day-one config option is cheaper than
  retrofitting it, and the enterprise-integration pattern it needs to handle is common enough to
  design for up front rather than treat as a pilot surprise.
- **Outbound Data-platform extract credentials**: tenant-brought warehouse credentials (tenant's own
  Snowflake/BigQuery/Databricks compute/storage) are the default path. For tenants without their own
  warehouse infrastructure, the platform provides a **platform-hosted extract bucket** fallback
  (S3/GCS, per-tenant prefix, pull-based, tenant-scoped signed URLs) — this is a firm, built fallback,
  not a hypothetical: smaller tenants are a real segment of this platform's target market (frontline
  ops SaaS skews toward operationally-mature-but-not-data-platform-mature organizations), so shipping
  without a no-warehouse path would leave a known customer segment unable to use this module's export
  capability at all.
- **Partner/third-party connector marketplace**: out of scope for MVP and Phase 2, confirmed against
  the roadmap (source-spec §25.4 places it in the later Autonomous Ops phase). The SDK, service-
  credential model, and typed connector-config schema are deliberately marketplace-ready so this
  remains additive later; no further scoping decision is needed now.
- **Pre-built ERP/HR connectors at GA**: ship deep, fully-tested connectors for the two highest-demand
  systems per category at GA — **Workday** for HR, **SAP** (S/4HANA) for ERP — and cover the long tail
  through the generic REST/SFTP/CSV mechanism plus a documented field-mapping guide. Two per category
  (not three) keeps the GA connector-maintenance surface bounded to what the integrations team can
  realistically keep at parity with each vendor's API changes, while covering the largest share of
  the addressable enterprise segment; partner-built connectors extend this list post-GA via the SDK.
- **Webhook retry-policy granularity**: one retry policy per subscription (not per-event-type) is the
  permanent MVP-and-beyond default. Per-event-type override adds configuration-surface complexity for
  a case with no demonstrated tenant need; it is revisited only if pilot/GA tenant usage data shows
  demand, not built speculatively now.
