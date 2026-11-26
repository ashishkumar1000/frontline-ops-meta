# Module 12 — Integrations & Extensibility (Integration Hub)

> This module is the platform's front door and back door to the rest of a company's software: it connects HR systems, ERPs, data warehouses, BI tools, chat platforms, and operational systems (POS, maintenance, inventory) to the frontline-ops platform. It solves the "how does this fit our existing IT estate" problem with a configuration-first connector catalog, signed and retryable webhooks, a documented API/SDK for custom builds, and SFTP/CSV batch jobs. Its main users are Enterprise Admins (who configure and monitor connections) and partner developers (who build custom integrations).

| | |
|---|---|
| **Module code** | `12-integrations-extensibility` |
| **Owning service** | Integration Hub Service (connector registry, webhook delivery engine, credential-vault broker, batch job orchestrator, inbound API/SDK surface) |
| **Epics** | EPIC-13 (Integration Hub) |

## In plain terms
- An admin picks a connector from a catalog (HR, ERP, Data/BI, Collaboration, Operational systems), walks a setup wizard — credentials, scope, field mapping, schedule, test connection — and activates it.
- HR connectors (e.g. Workday-class) sync worker records into the platform as staged candidates for review; ERP connectors (e.g. SAP-class) bring in location/product master data and push completed work orders back out.
- External systems can subscribe to platform events via **webhooks** (HTTPS callbacks with signed payloads so the receiver can verify they really came from the platform), with delivery logs, automatic retries, and one-click replay of failed deliveries.
- Developers integrate through scoped service credentials (OAuth2 client-credentials) and SDKs in TypeScript, Java, and Python that handle retries, idempotency keys, and signature checks for them.
- Every connection shows a live health state (HEALTHY, DEGRADED, FAILING, CREDENTIAL_EXPIRING) and alerts its owner; secrets are rotated on a schedule without ever leaving a window where no valid credential exists.
- Large file exchanges (roster imports, exports) run as scheduled SFTP/CSV batch jobs with per-row error reports and re-processing of only the rejected rows.

## Key concepts
| Concept | What it means |
|---|---|
| **Integration** | One configured connection to an external system: its credentials reference, data scope, owner, field mappings, sync direction, schedule, and health state. |
| **Connector catalog** | The registry of supported connector types (Identity cross-reference, HR, ERP, Data, BI, Collaboration, Operational Systems, Custom/API), each with a guided setup wizard. |
| **Field mapping** | The rule set translating an external system's fields into platform fields (with transforms like date-format or enum remap); required fields must be mapped before activation. |
| **WebhookSubscription / WebhookDelivery** | A subscription to platform events pushed to an external URL, and the per-attempt delivery record (status, retries, dead-letter) with replay support. |
| **HMAC-SHA256 signing** | The method used to sign every webhook payload with a shared secret so the receiver can prove the delivery is authentic; secret rotation uses a dual-secret overlap window so consumers never break. |
| **Service credentials** | Machine-to-machine API access (not tied to a human user), scoped to specific permissions and an organization subtree, with tenant-aware rate limits. |
| **BatchJob / BatchJobRun** | A scheduled SFTP/CSV import/export configuration and its per-run history (row counts, error report, re-trigger of rejected rows). |
| **Secret rotation (vault-backed)** | Credentials live only in the central secret manager — this module stores references — and rotation creates, validates, then revokes, never leaving zero valid credentials. |

## How it connects to the rest
- Identity & Organization (01) owns SSO/SCIM (the standard protocol for automated user provisioning); this module only links to it in the catalog UI, and the two modules merge HR-sync and SCIM user data per-field (SCIM wins on identity attributes, HR wins on job/org attributes) when both are active.
- It consumes domain events from Work Orchestration (02), Issues/CAPA (06), Evidence (05) and others to fan out webhooks and push data to ERP/BI/collaboration targets — it is the platform's broadest single event consumer.
- BI and Data-platform connectors query or extract from Analytics & Control Tower's (11) semantic layer and warehouse; this module owns the connection mechanics, Analytics owns the data and metric definitions.
- It implements the secret-management and rotation contract defined by Security, Privacy & Compliance (13); actual secret storage is always delegated to the platform's central secret manager.
- It is deliberately not the Notification Hub: end-user push/email/SMS notifications are a separate concern from system-to-system webhook delivery, and agent tool calls (module 10) keep their own audit trail rather than counting as webhooks.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | FR-INT-0xx requirements: connector catalog and setup wizard (001–007), per-category connector behavior (010–015), webhooks (020–025), API/SDK (030–033), batch transfer (040–043), health/secret rotation (050–053), and provisioning-conflict handling (060–062). |
| [high-level-design.md](./high-level-design.md) | The Integration Hub Service's four responsibilities, component diagram with per-connector adapter workers, data ownership, per-tenant rate limiting, and multi-tenancy. |
| [low-level-design.md](./low-level-design.md) | Database schema (integrations, webhook subscriptions/deliveries, field mappings, batch jobs), REST endpoints, retry/dead-letter and replay-safe delivery sequences, and idempotency pseudocode. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view, webhook delivery with retries, SFTP/CSV batch ingestion, ER diagram of owned entities, and the WebhookDelivery lifecycle state machine. |
