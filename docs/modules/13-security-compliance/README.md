# Module 13 — Security, Privacy & Compliance

> This module defines and enforces the security rules every other module must follow: keeping each customer's (tenant's) data fully separated, encrypting everything, controlling sessions and devices, and recording an untamperable audit trail. It exists to pass one critical test — Tenant A can never retrieve Tenant B's records, files, search results, embeddings, analytics, or AI context — and to give Security/Compliance officers one console for policy, audit logs, encryption keys, privacy requests, and incidents. Its users are Enterprise Admins, Security/Compliance officers, and external auditors; every user and AI agent in the platform is governed by it.

| | |
|---|---|
| **Module code** | `13-security-compliance` |
| **Owning service** | Security & Compliance Control-Plane Service (`security-compliance-service`) — plus a distributed enforcement contract (RLS, scoped queries, signed URLs) implemented inside every other service |
| **Epics** | EPIC-14 (Security & Compliance) |

## In plain terms
- It guarantees tenant isolation at every layer: database rows (RLS — row-level security, a database feature that filters rows by tenant), storage paths, search and AI-vector queries, caches, event streams, and AI context assembly — a tenant ID in a request is never trusted on its own.
- Admins set authentication and device policy: require SSO/MFA (multi-factor authentication), session timeouts, mobile device registration and binding, and remote wipe for lost or stolen devices.
- Everything is encrypted: TLS 1.3 in transit (even between internal services), AES-256 at rest via a managed KMS (key management service), with per-tenant keys rotated on a 90-day cadence and no downtime.
- AI is governed: fields are classified (PUBLIC/INTERNAL/PII/SENSITIVE_PII), redacted before reaching any AI model, restricted to tenant-approved model providers, and every AI answer must cite its sources with model-version traceability.
- Compliance officers search and export a tamper-evident, hash-chained audit log (7-year retention), handle data subject requests (a person's right to access, correct, or delete their personal data under GDPR/CCPA) with a 30-day SLA, and manage legal holds that block deletion.
- It tracks the vulnerability and incident program: continuous CVE scanning, annual penetration tests, patch SLAs by severity, and a documented incident-response process targeting SOC 2 Type II from day one.

## Key concepts
| Concept | What it means |
|---|---|
| **Tenant isolation contract** | The mandatory checks every module must implement at every access layer (API, database, storage, cache, search, vectors, events, AI context) so no cross-tenant data access is possible — enforced structurally, not by convention. |
| **RLS (row-level security)** | The Postgres mechanism that restricts every query to the current tenant's rows, acting as a safety net behind application-level scoping. |
| **SecurityPolicy** | The tenant-scoped configuration object (MFA, sessions, AI data controls, residency) distributed to all services through a short-TTL cache; a stale cache fails closed. |
| **AuditLog (hash-chained)** | The append-only, tamper-evident record of every security-relevant action; entries are chained by hashes so any edit — even by a privileged database user — is detected. |
| **AI data controls** | The PII classification registry plus redaction rules (NONE/MASK/BLOCK), per-data-class model/provider allowlists, mandatory source citations, and model-version traceability on every AI run. |
| **Data Lifecycle state machine** | The shared record journey — CREATE → ACTIVE → ARCHIVED → RETENTION_HOLD (optional legal hold) → DELETE/DESTROY — that Evidence, Submissions, AuditLogs, KnowledgeDocuments, and AgentRuns all implement. |
| **DSR (Data Subject Request)** | The workflow for a person's ACCESS / DELETION / RECTIFICATION request, fanned out to every owning service, with legal holds respected and a 30-day SLA. |
| **Break-glass access** | The explicit, time-boxed, fully-audited credential for support tooling that genuinely needs cross-tenant visibility — the only sanctioned exception, reviewed after every use. |

## How it connects to the rest
- It builds on Identity & Organization (01), which supplies the user/role/scope model; this module adds the tenant-boundary checks (RLS, storage, cache, search) that sit alongside scope resolution.
- It is the contract-enforcer for Agentic Operations (10): the Agent Runtime's context builder invokes this module's PII classification, redaction, model allowlist, no-cross-tenant-retrieval, citation, and kill-switch checks before anything reaches a model provider.
- It defines the encryption, key-hierarchy, and secret-rotation rules that Evidence (05) and Integrations & Extensibility (12) implement; connector credentials live only in the central secret manager this module specifies.
- Every other module (02–09, 11) must implement RLS, tenant-scoped caches, tenant-filtered search/analytics, and emit audit events into the central pipeline — this is a platform-wide Definition-of-Done gate, not a per-module option.
- It is a producer for the wider platform too: `security.policy_updated`, `dsr.fulfilled`, `key.rotated`, and `device.wiped` events flow to the Notification Hub (alerts) and Analytics (11) (compliance dashboards).

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | FR-SEC-0xx requirements in eight families: tenant isolation (001–008), MFA/SSO/session/device (010–014), encryption and keys (020–025), AI data controls (030–038), audit logging (040–044), residency and lifecycle (050–052), DSR (060–064), and vulnerability/incident management (070–074). |
| [high-level-design.md](./high-level-design.md) | The module's dual nature (control-plane service plus distributed enforcement contract), the six enforcement points, hash-chaining at scale, OPA/Rego bundle distribution, and fail-closed policy caching. |
| [low-level-design.md](./low-level-design.md) | Database schema (audit_logs, security_policies, data_subject_requests, etc.), admin REST endpoints, end-to-end isolation and cross-service DSR sequences, and hash-chain verification plus RLS policy pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component/container view with all enforcement points, tenant-isolation and DSR-fulfillment sequences, ER diagram of owned entities, and the Data Lifecycle state diagram. |
