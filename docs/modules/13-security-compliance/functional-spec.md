# Security, Privacy & Compliance — Functional Specification

Module: `13-security-compliance`
Epic: EPIC-14 (Security & Compliance)
Source: `source-spec.md` §21 (all subsections), §24.1 ("Tenant isolation" acceptance test), §24.2
(Definition of Done security items), Appendix E EPIC-14

## 1. Purpose

This module defines and enforces the security, privacy, tenant-isolation and AI-data-governance
contract that **every other module in the platform must implement**. It is unlike the other 14
modules in one respect: it is not a single bounded context whose UI a persona opens in isolation.
It is:

1. A **cross-cutting enforcement contract** — a set of mandatory patterns (RLS, scoped queries,
   signed URLs, OPA policy checks, audit emission) embedded inside the API Gateway, every domain
   service's data layer, the Agent Runtime's context builder, object storage access, search/vector
   retrieval, and the event backbone.
2. A **dedicated Security/Compliance control-plane service** that Enterprise Admins and Security/
   Compliance officers use directly: security policy configuration, audit log aggregation/search/
   export, key-rotation orchestration, device/session management, data-subject-request (DSR)
   workflow, incident tracking, and vulnerability/pentest program tracking.

Every requirement in source-spec §21 ("Security, Privacy and Tenant Isolation") is owned here.
The critical acceptance test this module exists to satisfy (§24.1): *"Tenant A cannot retrieve
Tenant B records, files, search results, embeddings, analytics, or agent context."*

## 2. Scope

### 2.1 In scope

- Tenant isolation contract definition and the enforcement points every module must implement
  (API authorization, application service layer, database queries via RLS, object storage,
  caches, search, vector retrieval, event routing, worker processing, analytics/warehouse,
  exports, AI context assembly).
- Security control configuration and enforcement: SSO/MFA policy, encryption (at rest/in transit)
  requirements and key management, central secret management contract, key rotation orchestration,
  least-privilege service identity contract, signed mobile build verification, device/session
  controls (session timeout, device binding, remote wipe/deprovision), audit logging (central
  aggregation, immutability, retention, query/export), security event monitoring, dependency/
  vulnerability scanning cadence, penetration testing cadence, rate limiting/abuse control policy.
- AI data control configuration and enforcement: PII/sensitive-field classification registry,
  redaction rule configuration, tenant/provider/model allowlists, AI data retention configuration,
  cross-tenant retrieval prevention contract (consumed by Agent Runtime and Knowledge Hub), source
  citation requirement, model-version traceability requirement, AI feature disablement by tenant/
  data class.
- Data lifecycle state machine (CREATE → ACTIVE → ARCHIVED → RETENTION_HOLD (optional) →
  DELETE/DESTROY) as a shared contract that Evidence, Submission, AuditLog, KnowledgeDocument, and
  AgentRun records all implement.
- Data subject rights (DSR) intake, tracking, cross-service orchestration, and audit trail
  (access, export, deletion/erasure, rectification requests) in a multi-tenant, effective-dated
  system.
- Data residency configuration per tenant (region/data-plane pinning) at provisioning time.
- Vulnerability management process definition (scanning cadence, pentest cadence, patch SLA by
  severity) and its tracking surface.
- Incident response process outline and the security incident tracking object.
- Security settings admin console, audit log search/export UI, data-residency selection at tenant
  provisioning, AI-feature-disabled-for-data-class banner (UX ownership).

### 2.2 Out of scope (owned by other modules, this module defines the contract they implement)

- Actual RBAC/ABAC scope resolution logic and the `UserScope` model — owned by Identity & Org
  (module 01); this module consumes it as the authorization layer it builds tenant isolation on
  top of, and defines the *additional* tenant-boundary checks (RLS, storage, cache, search) that
  sit alongside scope resolution.
- Evidence media hashing, storage tiering, and retention *execution* — owned by Evidence
  (module 05); this module defines the encryption-at-rest and retention-policy contract Evidence
  implements and supplies the KMS key hierarchy it uses.
- Agent Runtime's context builder, tool allowlists, and autonomy levels — owned by Agentic
  Operations (module 10); this module defines the AI data controls (redaction, model allowlist,
  no-cross-tenant-retrieval, citation, traceability, kill switch) that the Agent Runtime must call
  into and enforce.
- Integration credential storage mechanics and connector-specific OAuth flows — owned by
  Integrations & Extensibility (module 12); this module defines the secret-management and
  rotation contract Integrations implements.
- Per-module audit event *content* (what fields go in a given domain event) — each module decides
  what it logs; this module owns the central aggregation, immutability guarantee, retention, and
  query/export surface all of those events land in.
- Business continuity / disaster recovery runbooks for infrastructure failure (owned by
  Observability & Reliability, module covering §22) — this module owns the *data protection* and
  *breach* side of incident response, not general service-outage DR.

## 3. Personas served

| Persona | Relationship to this module |
|---|---|
| Enterprise Admin | Configures tenant security policy (MFA enforcement, session timeout, device binding, data residency, AI data controls, model allowlists), reviews audit logs, manages DSRs. |
| Security/Compliance Officer | Primary owner of the control plane: audit log search/export for compliance evidence, vulnerability/pentest tracking, incident response coordination, key-rotation oversight, DSR fulfillment sign-off, legal-hold management. |
| All users (as subjects of audit logging) | Every authenticated action a user takes is recorded in the audit trail; users may also be the *subject* of a data subject access/export/deletion request. |
| Auditor (external/internal compliance reviewer) | Read-only, time-boxed access to audit logs and compliance evidence exports for SOC 2 / ISO 27001 / customer security-review purposes; never has write access to production data. |
| AI Agent | Subject to AI data controls at every context-assembly and tool-execution step: redaction, allowlist, retention, and kill-switch checks apply to every `AgentRun`. |

## 4. Functional Requirements

### 4.1 Tenant isolation contract

- **FR-SEC-001**: The system MUST enforce tenant boundary checks at every access layer named in
  source-spec §21.1: API authorization, application service, database query (RLS), object storage,
  cache, search index, vector retrieval, event routing, worker processing, analytics/warehouse,
  exports, and AI context assembly. A `tenant_id` present in a request or token is necessary but
  never sufficient by itself — each layer must independently prove object-level scope.
- **FR-SEC-002**: Every Postgres table holding tenant data MUST carry a non-nullable `tenant_id`
  column and MUST have an RLS policy active in production (`FORCE ROW LEVEL SECURITY`) that
  restricts all `SELECT`/`INSERT`/`UPDATE`/`DELETE` to `tenant_id = current_setting('app.tenant_id')`.
  RLS is defense-in-depth behind application-layer scoping per convention — a bug in service-layer
  filtering must not be sufficient to leak cross-tenant rows.
- **FR-SEC-003**: Every object storage key MUST be prefixed `tenant/{tenant_id}/...`. Signed URL
  issuance MUST verify the requesting principal's tenant and scope against the object's tenant
  prefix before minting a URL; a URL request for an object outside the caller's tenant returns
  `403 SCOPE_DENIED` and is audit-logged as a potential cross-tenant access attempt.
- **FR-SEC-004**: Every search index document and vector embedding MUST carry `tenant_id` as a
  filterable field, and every query against OpenSearch or the vector store MUST inject a
  server-side `tenant_id` filter that the caller cannot override, independent of any user-supplied
  query text or metadata filter.
- **FR-SEC-005**: Every Kafka event envelope (§18.4) MUST carry `tenant_id`; every consumer
  (workflow workers, Agent Runtime, analytics pipeline, notification hub) MUST validate the event's
  `tenant_id` against its own processing context before acting and MUST reject/dead-letter events
  that fail tenant-context validation rather than silently processing them.
- **FR-SEC-006**: Redis cache keys for permission cache, session cache, rate-limit counters, and
  idempotency keys MUST be namespaced by `tenant_id` (e.g. `perm:{tenant_id}:{user_id}`); a cache
  key collision across tenants is treated as a Sev-1 security defect.
- **FR-SEC-007**: The Agent Runtime's context builder MUST resolve retrieval, tool arguments, and
  knowledge search exclusively within the acting principal's tenant and effective scope; an agent
  MUST NOT be able to retrieve, summarize, or reference another tenant's data even when a prompt
  or tool argument attempts to reference a cross-tenant identifier. A cross-tenant reference
  attempt is blocked, logged to `AuditLog`, and surfaced to the tenant's Security/Compliance
  officer as a security event.
- **FR-SEC-008**: Support-tooling access that requires cross-tenant visibility (e.g. platform
  operations debugging) MUST use an explicit, time-boxed, break-glass credential distinct from
  normal service identities; every break-glass session is fully audit-logged (who, why, which
  tenants, what was read/written) and requires post-hoc review.

### 4.2 MFA / SSO / session / device controls

- **FR-SEC-010**: Enterprise Admins can configure a tenant-level authentication policy: require
  SSO (SAML 2.0/OIDC per module 01), require MFA for all users or for privileged roles only,
  allowed MFA factors (TOTP authenticator app, WebAuthn/passkey, push, SMS as a discouraged
  fallback), and MFA re-challenge triggers (new device, sensitive action e.g. role change or DSR
  fulfillment, IP/geo anomaly).
- **FR-SEC-011**: Session policy is configurable per tenant: idle timeout (default 30 minutes web,
  15 minutes for privileged admin sessions), absolute session lifetime (default 12 hours web, 30
  days mobile with biometric re-auth), and concurrent session limits per user (default
  unrestricted for frontline associates, configurable cap for admin roles).
- **FR-SEC-012**: Mobile devices MUST register (`DeviceRegistration`) at first login: device
  identifier, platform, OS version, app version, attestation status (Play Integrity / DeviceCheck),
  and jailbreak/root detection result. A tenant policy MAY require device binding (work is only
  permitted from registered devices) and MAY reject rooted/jailbroken devices.
- **FR-SEC-013**: An Enterprise Admin or Security/Compliance officer can trigger a **remote wipe**
  for a lost/stolen/offboarded device: this revokes the device's refresh tokens and sync
  credentials immediately, and — where the mobile app supports managed-wipe (MDM-class control or
  in-app secure-enclave clear) — issues a wipe command for locally cached tenant data (forms,
  submissions queued for sync, cached knowledge documents, cached credentials). A wipe that
  targets a device with un-synced offline submissions surfaces a warning listing at-risk local
  data before confirmation.
- **FR-SEC-014**: Signed mobile builds are mandatory: the app store / enterprise distribution
  pipeline MUST verify code-signing certificates before install, and the backend MUST reject API
  calls from an app build whose attestation/signature cannot be verified once attestation
  enforcement is enabled for a tenant (see §11 Resolved Decisions for the enforcement rollout
  default).

### 4.3 Encryption, key management, secrets

- **FR-SEC-020**: All data in transit MUST use **TLS 1.3** between every client, service, and
  datastore hop, including internal gRPC/mTLS calls between services — no plaintext internal
  traffic, even inside a private network. (This module's text previously read "TLS 1.2+, TLS 1.3
  preferred"; corrected here to match engineering-decisions.md §8's platform-wide binding minimum
  of TLS 1.3 everywhere, so there is one stated version, not a range.)
- **FR-SEC-021**: All data at rest MUST be encrypted using envelope encryption via a managed KMS:
  **AES-256** Postgres storage-level encryption plus column-level encryption for defined sensitive
  fields (see AI PII classification, §4.4), AES-256 object storage server-side encryption with
  per-tenant data keys, and encrypted backups/snapshots — the concrete cipher/mechanism specified
  once platform-wide in engineering-decisions.md §8 and applied here.
- **FR-SEC-022**: Key hierarchy: one KMS-managed master key (customer-managed key, "CMK", offered
  as an enterprise-tier add-on — see §11 Resolved Decisions) per region/data-plane; a
  per-tenant data encryption key (DEK) wrapped by the regional master key; DEKs are rotated on a
  scheduled cadence (default 90 days) and on-demand (suspected compromise, tenant offboarding).
  `EncryptionKeyMetadata` in this module's control plane stores key *references and rotation
  state only* — never raw key material, which lives exclusively in the KMS.
- **FR-SEC-023**: Key rotation MUST NOT cause service downtime or data unavailability: rotation
  wraps a new DEK version, re-encrypts data lazily on next write plus a background re-encryption
  job for cold data, and the system retains the ability to decrypt with prior key versions until
  all data under that version has been migrated (see LLD §8 key-rotation orchestrator).
- **FR-SEC-024**: All application/service credentials, third-party API keys, and integration
  secrets (module 12) are stored in a central secret manager (e.g. AWS Secrets Manager/HashiCorp
  Vault-class); no secret is ever stored in application config, source control, or a database
  column in plaintext. Secret rotation is scheduled per secret type (default 90 days for
  service-to-service credentials, per-provider requirement for third-party integration secrets)
  and rotation failures raise a security alert, not a silent skip.
- **FR-SEC-025**: Every service identity (service account / workload identity) is provisioned with
  least-privilege IAM scoped to only the resources that service needs; a quarterly access review
  of service identity permissions is a standing compliance task tracked in this module's control
  plane.

### 4.4 AI data controls

- **FR-SEC-030**: The system maintains a **PII/sensitive-field classification registry**: a
  tenant-overridable mapping of field paths (form field types, submission fields, knowledge
  document sections) to sensitivity classes (`PUBLIC`, `INTERNAL`, `PII`, `SENSITIVE_PII` — e.g.
  government ID, health data, biometric). Every field entering Agent Runtime context or a
  knowledge embedding pipeline is checked against this registry.
- **FR-SEC-031**: Tenants can configure **redaction rules** per data class: `NONE` (no redaction),
  `MASK` (replace value with a typed placeholder, e.g. `[REDACTED:PII_NAME]`), or `BLOCK`
  (exclude the field/record entirely from model input). Redaction is applied by the Agent
  Runtime's context builder before any prompt is sent to a model provider, and before any document
  is embedded into the vector store.
- **FR-SEC-032**: Tenants configure a **model/provider allowlist** per data class: which model
  providers (and specific model versions) are permitted to process which sensitivity classes of
  data. A tenant may, for example, allow `SENSITIVE_PII`-tagged data only on a specific
  private/dedicated model deployment, or block it from all third-party model providers entirely
  (forcing `BLOCK` redaction for that class).
- **FR-SEC-033**: AI processing retention is configurable independent of the underlying record's
  retention: a tenant may require that prompt/response logs sent to a model provider be retained
  for 0 days (no provider-side retention, where the provider contract supports it) even though the
  underlying `Submission` retains per §4.6. Retention overrides are recorded per tenant and audited.
- **FR-SEC-034**: No cross-tenant retrieval: this module's contract (enforced structurally by
  FR-SEC-004/007) guarantees an Agent Runtime retrieval call or Knowledge Hub search cannot return
  results outside the acting principal's tenant, regardless of embedding similarity score.
- **FR-SEC-035**: Every AI-generated answer sourced from Knowledge Hub content MUST include source
  citations (document id, version, effective date) sufficient for the user to open and verify the
  underlying document; an answer with no qualifying source is labeled as a general-knowledge
  response, not a cited answer, and is flagged distinctly in the UI.
- **FR-SEC-036**: Every `AgentRun` record MUST carry model-version traceability: provider, model
  identifier, model version/snapshot, prompt template version — sufficient to reproduce or audit
  the exact configuration that produced a given recommendation or action months later.
- **FR-SEC-037**: AI feature disablement: an Enterprise Admin can disable AI features
  tenant-wide, per-agent, or per-data-class (e.g. "disable Visual AI on any image tagged
  `contains_faces`"). A disabled combination causes the Agent Runtime's policy check to reject the
  invocation before context assembly (fail-closed) and the UI surfaces an
  "AI unavailable for this data" banner rather than silently omitting the feature.
- **FR-SEC-038**: This module's PII classification (FR-SEC-030), redaction (FR-SEC-031), and model
  allowlist (FR-SEC-032) controls are the generic enforcement mechanism that module 10's EU AI Act
  high-risk-agent obligations (engineering-decisions.md §12 — human oversight, logging, transparency,
  risk management for employment/worker-management AI) build on for any agent whose output concerns
  a named individual employee. This module owns the mechanism; module 10 owns which agents/outputs
  are classified high-risk and the four Article 9/12/13/14 functional requirements themselves, so
  the classification logic is not duplicated here.

### 4.5 Audit logging

- **FR-SEC-040**: The following categories MUST be audit-logged for every tenant, at minimum:
  authentication events (login, logout, MFA challenge, failed login, session revocation), all
  create/update/delete on `User`, `UserScope`, `Role`, `SecurityPolicy`, `Tenant` configuration;
  all record access/export that includes PII/sensitive data; all `AgentRun`/`AgentToolExecution`
  records with policy decisions; all DSR lifecycle transitions; all `DeviceRegistration` and remote
  wipe actions; all key-rotation events; all admin-console configuration changes.
- **FR-SEC-041**: Audit log entries are **append-only** and immutable once written: no API or
  database role available to application code may `UPDATE` or `DELETE` an existing `audit_logs`
  row. Entries are hash-chained (see LLD §7) so that any tampering — including by a privileged
  database user — is detectable by a periodic chain-verification job.
- **FR-SEC-042**: Default audit log retention is 7 years (configurable up to regulatory maximum
  per tenant compliance profile), stored in cost-tiered storage (hot for the trailing 90 days,
  cold/archival thereafter) without loss of query or export capability.
- **FR-SEC-043**: Security/Compliance officers and Auditors can query the audit log by tenant
  scope, organization node, actor, action type, resource type/id, and date range, and export
  results in a compliance-evidence format (CSV and a hash-chain-verifiable JSON bundle) for
  external auditor review. Export itself is audit-logged.
- **FR-SEC-044**: Audit log write path failures MUST NOT silently drop events: if the central
  audit aggregation pipeline is unavailable, emitting services buffer locally and retry, and
  sustained backpressure escalates to a Sev-2 alert; a security-critical action (e.g. role change,
  DSR fulfillment, key rotation) that cannot confirm its audit write is durably queued MUST block
  and fail the originating action rather than proceed unaudited (fail-closed for security-critical
  paths; see NFR-SEC-002).

### 4.6 Data residency and data lifecycle

- **FR-SEC-050**: At tenant provisioning, an Enterprise Admin (or the platform onboarding team)
  selects a data residency region/data-plane from the tenant's contracted set (e.g. US, EU, APAC).
  Residency selection determines primary database region, object storage region, and — where
  contractually required — restricts model provider selection to providers with matching regional
  processing guarantees.
- **FR-SEC-051**: Every tenant-owned record with a lifecycle implements the shared **Data
  Lifecycle** state machine: `CREATE → ACTIVE → ARCHIVED → RETENTION_HOLD (optional) →
  DELETE/DESTROY (where policy permits)`. `RETENTION_HOLD` is an optional detour entered when a
  legal hold, active litigation, or open regulatory inquiry requires preservation beyond normal
  retention; a record under `RETENTION_HOLD` cannot transition to `DELETE/DESTROY` regardless of
  a DSR deletion request or expired retention timer until the hold is explicitly released by a
  Security/Compliance officer.
- **FR-SEC-052**: Retention policy (time-in-`ACTIVE`, time-in-`ARCHIVED` before eligible for
  delete) is configurable per tenant and per record type, defaulting to the platform baseline
  (e.g. `Submission`/`Evidence` default 7 years, `AuditLog` per FR-SEC-042, `AgentRun` default 3
  years). A background retention-sweep job (owned per-service, orchestrated centrally) transitions
  eligible records and raises `DELETE/DESTROY` only after confirming no active `RETENTION_HOLD`.
  **EU AI Act reconciliation**: for the specific subset of `AgentRun`/`AgentToolExecution` records
  that concern a named individual employee's performance/compliance pattern (module 10's high-risk
  agents, per engineering-decisions.md §12), the 3-year `AgentRun` default above **is** the Article
  12 floor the register sets — it is not a separate, shorter number to reconcile. Within that
  3-year window, the full context payload (prompt/tool-input contents) is retained for 1 year only
  and then trimmed to the durable decision record (decision, policy version, approver, timestamp,
  model-version traceability per FR-SEC-036) for the remaining 2 years, per the register's GDPR
  data-minimization rationale — this module's retention-sweep job (not module 10) executes that
  payload-trim step, since it already owns the general retention-sweep mechanism.

### 4.7 Data subject rights (DSR)

- **FR-SEC-060**: Any user (or an Enterprise Admin acting on a data subject's behalf, e.g. an
  ex-employee's GDPR/CCPA request routed through the tenant) can submit a `DataSubjectRequest` of
  type `ACCESS` (export all personal data held), `DELETION` (erase personal data), or
  `RECTIFICATION` (correct inaccurate personal data), scoped to a specific tenant.
- **FR-SEC-061**: A submitted DSR fans out to every owning service (Identity/Org, Work, Forms,
  Evidence, Issues/CAPA, Communications, Learning, Agent Runtime logs) via the DSR fulfillment
  orchestrator (LLD §8); each service reports back what personal data it holds for the subject and
  either exports it (`ACCESS`) or executes deletion/anonymization (`DELETION`) subject to its own
  retention and legal-hold constraints.
- **FR-SEC-062**: A `DELETION` request that conflicts with an active `RETENTION_HOLD` or a
  regulatory retention floor (e.g. audit logs, financial/compliance records that must be retained
  regardless of subject request) is **not silently honored**: the affected records are excluded
  from deletion, the exclusion and its legal basis are recorded on the `DataSubjectRequest`, and
  the requester/admin is shown exactly which categories were retained and why (see §7 edge cases).
- **FR-SEC-063**: DSR fulfillment has a target SLA (default 30 days for `ACCESS`/`RECTIFICATION`,
  30 days for `DELETION` unless a legal hold applies, consistent with GDPR Article 12 timing) and
  the DSR record tracks status (`SUBMITTED → IN_PROGRESS → PARTIALLY_FULFILLED/FULFILLED →
  CLOSED`), with reminders to the Security/Compliance officer as the SLA approaches.
- **FR-SEC-064**: Every DSR action is itself audit-logged, and the `DataSubjectRequest` record and
  its fulfillment trail are retained even after the underlying personal data is deleted (proof of
  compliance), stripped of the deleted content itself.

### 4.8 Vulnerability management and incident response

- **FR-SEC-070**: Dependency and vulnerability scanning runs continuously in CI (every build) and
  on a scheduled cadence against running infrastructure (default: daily for container/dependency
  CVE scans, weekly for infrastructure configuration scans).
- **FR-SEC-071**: External penetration testing is performed at least annually and after any major
  architecture change (e.g. new Agent Runtime autonomy level, new data residency region); results
  and remediation status are tracked in `security_incidents`/a linked findings table and are
  available to enterprise customers under NDA as compliance evidence.
- **FR-SEC-072**: Patch SLA by severity (from scan or pentest finding): Critical — 7 days,
  High — 30 days, Medium — 90 days, Low — next scheduled release. SLA breaches escalate to
  engineering leadership and are visible on the Security/Compliance officer's console.
- **FR-SEC-073**: A **security incident** (suspected or confirmed breach, cross-tenant data
  exposure, credential compromise, DDoS, etc.) is tracked as a `SecurityIncident` record with
  severity, affected tenant(s)/scope, detection time, containment actions, root cause, and
  customer notification status/timeline, following the incident response process (functional
  outline in §8).
- **FR-SEC-074**: Rate limiting and abuse controls are tenant-aware and principal-aware (per
  source-spec §18.1): default per-tenant API rate limit plus per-principal (user/service
  identity/agent) sub-limits, with configurable tenant-level overrides for legitimate high-volume
  integrations, and automated temporary throttling/blocking on abuse-pattern detection (credential
  stuffing, scraping, anomalous agent tool-call volume).

## 5. UX states

### 5.1 Security settings admin console (Enterprise Admin, Security/Compliance officer)

- **Empty/first-time**: tenant provisioned with platform defaults (MFA optional, 30-min session
  timeout, no device binding, standard retention); console shows a "Recommended hardening" panel
  suggesting SSO+MFA enforcement.
- **Loaded**: tabs for Authentication (SSO/MFA), Sessions & Devices, Encryption & Keys (rotation
  status, last rotation date — read-only status, no raw key display), Audit Log, AI Data Controls,
  Data Residency (read-only after provisioning, see §7), Vulnerability & Compliance, Incidents.
- **Saving/validation error**: a policy change that would lock out the current admin (e.g.
  enforcing a device-binding policy with zero registered devices) is blocked client- and
  server-side with an explicit warning before commit.
- **Permission-denied**: a non-Security-scoped admin sees the console in read-only summary form
  (their own MFA/session status only), not the tenant-wide configuration surface.

### 5.2 Audit log search/export UI

- **Empty**: "No events match this filter" with a suggestion to widen the date range or scope.
- **Loaded**: paginated result table (actor, action, resource, timestamp, outcome) with a
  detail-drawer per entry showing before/after diff where applicable and the hash-chain segment id.
- **Export in progress**: async job with progress indicator for large date-range exports; result
  delivered as a downloadable signed-URL bundle with an expiry, and the export event itself
  appears in the audit log within seconds.
- **Offline/degraded**: if the audit aggregation store is in a degraded read state, the console
  shows a banner "Audit search temporarily degraded — showing cached results as of {time}" rather
  than presenting incomplete results as complete.

### 5.3 Data residency selection at tenant provisioning

- Presented once, at tenant creation, as a required field with the contracted region list; shown
  as read-only afterward with a "Contact support to discuss migration" affordance (residency
  migration is an operational project, not a self-service toggle) — see §11 Resolved Decisions.

### 5.4 AI-feature-disabled-for-data-class banner

- Wherever an AI feature (Copilot answer, Visual AI result, Agent recommendation) would normally
  render but is blocked by an AI data control, the UI shows an inline banner: "AI assistance is
  disabled for this data category by your organization's policy" with a link for admins to the AI
  Data Controls tab, rather than hiding the feature silently or returning a generic error.

## 6. Field-level behavior — key control objects

| Object | Key fields | Behavior |
|---|---|---|
| `SecurityPolicy` | `mfa_required`, `sso_required`, `session_idle_timeout_min`, `session_absolute_ttl_hours`, `device_binding_required`, `allowed_mfa_factors[]` | Tenant-scoped singleton (one active version at a time), versioned on change, changes take effect for new sessions immediately and existing sessions at next re-auth/idle-check. |
| `DataSubjectRequest` | `subject_ref`, `request_type`, `status`, `submitted_at`, `sla_due_at`, `excluded_categories[]` | `excluded_categories` populated only when a legal hold or regulatory floor blocks full fulfillment; never silently empty when exclusions exist. |
| `EncryptionKeyMetadata` | `key_ref` (KMS ARN/id, never raw key), `rotation_state`, `last_rotated_at`, `next_rotation_due_at` | Reference-only; any attempt to store raw key material in this table is a build-time lint failure. |
| `DeviceRegistration` | `device_id`, `status` (`ACTIVE`/`WIPED`/`DEREGISTERED`), `attestation_result`, `last_seen_at` | `WIPED` is terminal for that device instance; a re-enrolled physical device gets a new `device_id`. |

## 7. Validation and edge cases

- **Data subject deletion request conflicting with legal-hold evidence**: the DSR orchestrator
  MUST check `RETENTION_HOLD` status on every candidate record before deleting; conflicting
  records are excluded, listed with reason (`LEGAL_HOLD`, `REGULATORY_RETENTION`) on the DSR
  record, and the requester is notified which categories could not be deleted and why, with an
  estimated hold-release re-evaluation date where known.
- **Cross-tenant data leak incident response runbook trigger**: any FR-SEC-001–008 boundary
  violation detected (RLS bypass, mis-scoped signed URL, cross-tenant search hit, agent
  cross-tenant retrieval) automatically creates a `SecurityIncident` at severity `CRITICAL`,
  pages the on-call Security/Compliance function, and freezes the offending code path via feature
  flag/kill switch pending investigation — this is a fully automated trigger, not one requiring a
  human to first notice the leak.
- **Audit log tampering detection**: the periodic chain-verification job (LLD §8) recomputing
  hash links detects any row mutation, deletion, or reordering; a verification failure raises a
  `SecurityIncident` at severity `CRITICAL` and freezes further writes to the affected partition
  pending forensic review (fail-closed — see NFR-SEC-002).
- **Encryption key rotation without downtime**: rotation must never require taking a service
  offline or blocking writes; see FR-SEC-023 and LLD §8 for the dual-key-version read path during
  migration.
- **AI redaction false negative** (PII leaks into model context despite a `MASK`/`BLOCK` rule):
  treated as a security incident, not merely a data-quality bug — the specific `AgentRun` is
  flagged, the model provider is asked (where contractually possible) to purge the affected
  prompt/response, the classification registry is patched, and affected tenants are notified per
  the incident response process if the leaked data was sensitive and left the tenant boundary
  (e.g. sent to a third-party model provider).
- **MFA enforcement change mid-session**: enforcing MFA tenant-wide does not immediately terminate
  active sessions that predate the policy; it requires MFA at next login/re-auth, with an optional
  "force re-auth within N days" grace-period setting.
- **Device binding enabled with zero registered devices**: blocked at save time (see §5.1) to
  prevent enterprise-wide lockout.
- **DSR submitted for a user who is also an active employee with ongoing legitimate work
  records**: `DELETION` requests distinguish personal/identity data (eligible for erasure/
  anonymization) from operational record content the user authored in an official capacity (e.g.
  a completed audit submission), which is anonymized (actor reference replaced with a tombstone
  identity) rather than deleted outright, preserving historical/operational integrity per
  Principle P7 while still satisfying the erasure request for personal data.

## 8. Incident response process outline

1. **Detect** — automated (SecurityIncident auto-creation per §7) or human-reported.
2. **Triage** — Security/Compliance officer assigns severity (`CRITICAL`/`HIGH`/`MEDIUM`/`LOW`)
   and scope (affected tenant(s), data categories).
3. **Contain** — kill switch / feature flag / credential revocation / network isolation as
   applicable; contained state and timestamp recorded on `SecurityIncident`.
4. **Investigate** — root cause analysis using audit log query surface (§4.5) and system logs;
   findings recorded.
5. **Notify** — affected tenant(s) notified per contractual/regulatory SLA (e.g. 72-hour breach
   notification under GDPR where applicable); notification timeline tracked on the incident.
6. **Remediate** — fix deployed, verified against the specific failure mode; linked to the patch
   SLA (§4.8) if it originated from a known vulnerability class.
7. **Close & review** — post-incident review recorded; process/control gaps feed back into
   `SecurityPolicy` defaults or the vulnerability scanning ruleset.

## 9. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-SEC-001 | Audit log write latency | p95 < 200 ms from action to durable queue acknowledgement; p99 < 1 s |
| NFR-SEC-002 | Audit durability for security-critical actions | Fail-closed: the originating action blocks/fails if the audit write cannot be durably queued (no "audit best-effort" mode for role/policy/key/DSR actions) |
| NFR-SEC-003 | Audit log durability (storage) | 11-nines object-storage-class durability for archived partitions; no data loss on single-AZ failure for hot partitions |
| NFR-SEC-004 | Key rotation frequency | DEKs: 90 days scheduled, on-demand on suspected compromise; secrets: 90 days default |
| NFR-SEC-005 | Vulnerability scanning cadence | CI: every build; infra dependency scan: daily; infra config scan: weekly |
| NFR-SEC-006 | Penetration test cadence | Annual minimum, plus after major architecture change |
| NFR-SEC-007 | Chain-verification job cadence | Every partition verified at least every 24 hours; sampled continuous verification for the active (hot) partition |
| NFR-SEC-008 | Security policy propagation | A `SecurityPolicy` change is enforced for new requests within 60 seconds platform-wide (cache TTL, see LLD §7) |
| NFR-SEC-009 | DSR fulfillment SLA | 30 days default for ACCESS/DELETION absent legal hold |
| NFR-SEC-010 | Tenant isolation test coverage | Automated cross-tenant-access regression suite (shared with module 01) runs on every deploy to every enforcement layer in FR-SEC-001 |

## 10. Dependencies

- **Identity & Organization (module 01)**: supplies the RBAC/ABAC/scope authorization layer this
  module's tenant-isolation contract builds on; `UserScope` effective-access resolution is a
  prerequisite input to every FR-SEC-001-family check. This module in turn supplies module 01's
  authorization regression-test *pattern* extended to every other module's data layer.
- **Evidence (module 05)**: implements the encryption-at-rest, KMS key hierarchy, per-tenant
  object storage prefixing, and signed-URL scope check this module defines (FR-SEC-003, §4.3);
  also implements the `ACTIVE → ARCHIVED → RETENTION_HOLD → DELETE` lifecycle for media objects.
- **Agentic Operations (module 10)**: the Agent Runtime's context builder is the primary
  enforcement point for AI data controls (§4.4) — PII classification/redaction, model allowlist,
  no-cross-tenant-retrieval, citation, model traceability, and the kill switch are all invoked
  from there; this module owns policy configuration and audit, module 10 owns invocation.
- **Integrations & Extensibility (module 12)**: implements the central secret-management and
  rotation contract (FR-SEC-024) for every third-party connector credential; also implements
  webhook signing keys under the same key-management contract.
- **Every other module (02–09, 11)**: MUST implement RLS on every owned table, tenant-scoped
  cache keys, tenant-filtered search/analytics queries, and MUST emit audit events for every
  state-changing action into the central audit pipeline (§4.5) — this is a platform-wide
  Definition-of-Done gate (source-spec §24.2 "Security review completed" / "Audit logs present").

## 11. Resolved Decisions

- **Compliance certifications targeted**: the platform designs and operates to **SOC 2 Type II**
  from GA — audit-log immutability, access reviews, incident-response process, and vendor
  management as specified in this module are SOC 2 control-aligned by construction, so this is the
  natural first formal certification. **GDPR**-aligned DSR handling (§4.7) and residency (§4.6)
  mechanics are built from day one regardless of formal certification, given the multi-region
  enterprise customer base. **ISO 27001** and **HIPAA-adjacent** controls are deferred to Phase 2,
  triggered by confirmed customer/vertical demand rather than built speculatively now — HIPAA in
  particular requires a formal BAA program and additional PHI-specific controls beyond the generic
  `SENSITIVE_PII` class defined here, which is a material scope addition best justified by an actual
  healthcare-adjacent enterprise deal.
- **Customer-managed encryption keys (CMK/BYOK)**: offered as an **enterprise-tier add-on**, not a
  default for all tenants — the tenant supplies/owns the regional master key in their own KMS
  account, and platform DEKs are wrapped by it. This is firm, not merely recommended: CMK's
  operational complexity (a tenant revoking their CMK must degrade the tenant to a
  read-only/support-intervention state gracefully, never silently fail — see LLD §9 failure modes)
  is worth taking on only for the enterprise segment that specifically demands key sovereignty.
- **Data residency migration**: moving an existing tenant between regions post-provisioning is an
  assisted operational project executed via a documented, tooling-assisted runbook — never a
  self-service console action. Coordinated migration across Postgres, object storage, search
  indices, and the vector store with zero data loss and a defined maintenance window is high-risk
  enough that removing the human-run-runbook step is not justified until sustained customer demand
  makes the investment in a self-service flow worthwhile.
- **Mobile build attestation enforcement (FR-SEC-014)**: enabled as attestation *verification and
  logging* from GA, but not as a hard block on unattested devices for the first release cycle in
  production. The hard block turns on automatically at the start of the second release cycle after
  GA (a fixed timeline, not an open-ended "once measured" condition) unless the measured
  false-positive rate (legitimate devices failing attestation, e.g. certain enterprise MDM
  configurations) exceeds 1% of active devices, in which case Security/Compliance extends
  verify-only mode by one additional release cycle while the false-positive cause is fixed.
- **SMS as an MFA factor**: supported only as a fallback factor, never selectable as a tenant's
  sole enforced factor, given known SIM-swap risk. The admin console flags SMS as "discouraged"
  whenever an admin selects it as an available factor, and a tenant cannot configure SMS as the
  *only* enabled factor for a role (at least one non-SMS factor must remain enabled) — this is an
  enforced constraint, not just an advisory label.
