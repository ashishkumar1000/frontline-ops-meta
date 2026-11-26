# External Integrations

**Analysis Date:** 2026-11-27

> **How to read this document:** This repo is **specification-only** — there is no running
> application, so there are **no live platform integrations** (no API keys, no databases, no
> webhooks firing). Everything in the "Specified platform integrations" blocks below is
> *documentation of a future system*, written at build-ready depth in `docs/`. The only integrations
> that actually exist today are the **AI-engineering tooling** wired into the repo (BMAD + GSD +
> ZCode skills), listed first and marked **LIVE**. Future GSD phases implementing code from these
> specs must treat the spec blocks as requirements to build against, not as existing connections.

## APIs & External Services

### LIVE (repo tooling, exists today)

**GSD update check (only outbound network touchpoint in the repo):**
- `.claude/hooks/gsd-check-update.js` (SessionStart hook, wired in `.claude/settings.json`)
  spawns `.claude/hooks/gsd-check-update-worker.js`, which shells out to `npm view` for the
  `get-shit-done-cc` package (logic in `.claude/get-shit-done/bin/check-latest-version.cjs`) and
  caches the result in `~/.cache/gsd/gsd-update-check.json`.
  - Client: npm CLI (no SDK, no stored credentials — uses whatever npm auth the machine already has).
- All other GSD hooks (`.claude/hooks/*.js`, `*.sh`) and BMAD scripts (`_bmad/scripts/*.py`) are
  local-file operations only.

**BMAD / GSD / ZCode skill integrations (file-based, no network):**
- BMAD personas + workflows: `_bmad/config.toml`, `_bmad/bmm/`, `_bmad/core/`, output dir
  `_bmad-output/` (currently empty); skills exposed via `.claude/skills/` and `.zcode/skills/`
  (29 `bmad-*` skills each).
- GSD subagents/commands: `.claude/agents/` (33 `gsd-*.md` agents), `.claude/commands/gsd/`
  (slash commands), `.claude/get-shit-done/workflows/` (playbooks).
- Routing contract between the two systems is defined in the root `CLAUDE.md` / `AGENTS.md`
  (BMAD = planning/discovery only, outputs to `docs/`; GSD = all code execution, reads
  `.planning/codebase/`).

### Specified platform integrations (documentation only — no code exists)

**ERP / HR connectors (Integration Hub, module 12):**
- **Workday** — deep, fully-tested HR connector at GA (inbound worker-record sync → `User` /
  `OrganizationNode` staging). Requirement FR-INT-010 in
  `docs/modules/12-integrations-extensibility/functional-spec.md`; GA vendor decision in §10
  "Resolved Decisions" of the same file.
  - Auth: per-connector credentials stored as vault references only (`credential_ref`, never raw
    secrets — `docs/modules/12-integrations-extensibility/low-level-design.md`, `connector_credentials`
    table with `vault_ref`, `rotation_state`, dual-secret grace window).
- **SAP S/4HANA** — deep ERP connector at GA (inbound site/product/asset/cost-center master data;
  outbound work orders and verified counts). FR-INT-011, same files.
- Long-tail HR/ERP (SuccessFactors/BambooHR/Oracle/Dynamics-class) — covered by the generic
  REST/SFTP/CSV mapping engine, not dedicated connectors (module 12 §10).
- Field-mapping wizard with mandatory "test connection" step: FR-INT-003 (`functional-spec.md` §4.1).

**Data-platform / BI / Collaboration / Operational connectors:**
- Snowflake / BigQuery / Databricks-class — scheduled or CDC-style outbound extract to tenant-owned
  warehouses using tenant-brought credentials; platform-hosted extract bucket (S3/GCS, signed URLs)
  as the no-warehouse fallback. FR-INT-012 + §10 Resolved Decisions,
  `docs/modules/12-integrations-extensibility/functional-spec.md`.
- Power BI / Tableau-class — scoped read-only semantic-layer query or scheduled CSV/Parquet extract.
  FR-INT-013, same file.
- Teams / Slack-class — outbound notification mirroring + optional inbound slash-command actions
  executed under the invoking user's mapped identity. FR-INT-014.
- POS / inventory / workforce-management / maintenance systems — bidirectional operational
  connectors. FR-INT-015.
- Payroll export — outbound connector consuming `timesheet.approved`/`timesheet.exported` events
  (module 15 owns timesheet data, never payroll computation): `docs/01-FSD.md` §12 table.

**Inbound partner API / SDK surface:**
- OAuth2 client-credentials service credentials (`client_id` + secret, or signed JWT for
  high-security tenants), scoped to API permissions + `OrganizationNode` subtree, sandbox/test
  mode. FR-INT-030/031/033, `docs/modules/12-integrations-extensibility/functional-spec.md` §4.4.
- Published SDKs (TypeScript, Java, Python minimum) with typed models, auto `Idempotency-Key`,
  retry with backoff+jitter, webhook signature-verification helpers. FR-INT-032.

**AI model providers (Agent Runtime, module 10):**
- OpenAI / Anthropic / Azure OpenAI / AWS Bedrock — accessed exclusively through a
  **model-provider-agnostic gateway** (LiteLLM-style abstraction) with typed tool-calling, request
  queuing/cost control, circuit breaker and secondary-provider failover.
  Sources: `docs/_context/conventions.md` (§ Agent Runtime / AI),
  `docs/modules/10-agentic-operations/high-level-design.md` (§ Model Gateway, failure-modes table),
  ADR-007-adjacent retrieval design in `docs/modules/09-visual-ai/high-level-design.md`
  (pluggable vision-model backends behind the same gateway).
  - Auth: provider keys held by the gateway, never embedded in prompts or services; tenant data
    isolation rules for outbound model calls in `docs/01-FSD.md` §9.2 (AI Data Controls).
- No AI vendor keys exist in this repo — the gateway is a design, not a deployment.

**Identity providers (module 01 + auth register):**
- **Keycloak (self-hosted)** — the platform's default OIDC provider, named concretely in
  `docs/_context/engineering-decisions.md` §1.
- Federated enterprise IdPs — Okta, Azure AD / Entra ID, OneLogin, Google Workspace via
  SAML 2.0 / OIDC; user provisioning via **SCIM 2.0** push
  (`docs/modules/01-identity-organization/high-level-design.md` § technology choices and §5.3;
  endpoints `POST /api/v1/scim/v2/Users`, `/Groups`).

**IoT devices / gateways (module 17):**
- **MQTT broker** ingestion — per-tenant topic namespace, mTLS transport, broker-level ACLs so
  tenant A cannot publish into tenant B's topic tree
  (`docs/modules/17-iot-sensor-platform/high-level-design.md` § multi-tenancy, § technology choices).
- **HTTP webhook ingestion** — signed webhook path for retrofit sensors / building-management
  exports / LoRaWAN-class network servers that cannot run MQTT. Both paths normalize into one
  internal ingestion contract before evaluation.
- Target ingest volume: 50,000 readings/second platform-wide (NFR-IOT-001; ADR-009 in `docs/01-FSD.md`).

**Notifications (Notification Hub, FSD §3 + module 07):**
- **FCM** (Android push) and **APNs** (iOS push) behind a unified hub with channel fallback chains
  (`docs/modules/07-communications-knowledge/high-level-design.md` failure-modes table).
- Email / SMS providers — vendor-agnostic placeholders (no vendor named in the spec); fallback
  channel after push per FR-COMMS-011.

## Data Storage

All entries below are **specification only** — no database is provisioned or connected in this repo.

**Databases:**
- **PostgreSQL** (Aurora/Cloud SQL-class, managed) — authoritative operational store; row-level
  multi-tenancy via `tenant_id` on every table + Postgres RLS as defense-in-depth
  (ADR-002, `docs/01-FSD.md`). Connection identity is enforced via the
  `app.tenant_id` transaction-local setting (`SET LOCAL` per transaction) — this is the closest
  thing to a "connection env var" the spec defines; no literal env-var names exist yet.
- **TimescaleDB** — Postgres extension on the same fleet for `sensor_readings` hypertables
  (ADR-009, `docs/01-FSD.md`; module 17).
- **pgvector** — vector store inside the tenant-scoped Postgres tier for RAG embeddings, with a
  `VectorStore` port for managed-vector-DB escape hatch (>5M vectors/tenant threshold) (ADR-007).
- **Redis** — disposable cache/session/rate-limit/idempotency state, never a system of record
  (`docs/_context/conventions.md`; `docs/01-FSD.md` §1.1 data-plane notes).
- **OpenSearch** — operational/document search; never used for authorization decisions.
- **Lakehouse** (Snowflake/BigQuery/Databricks-class) — analytics marts fed by CDC, read by the
  Control Tower API, never by OLTP paths (`docs/_context/conventions.md` § Analytics).

**File Storage:**
- **S3-compatible object storage** (AWS S3 / GCS) — evidence media only; per-tenant prefix,
  KMS-encrypted, signed-URL access, resumable multipart upload protocol specified in
  `docs/01-FSD.md` §7.3 sequence diagram and the Evidence Service block in §3.
- Local filesystem: not used by the platform spec; this repo itself is plain files only.

**Caching:**
- Redis (see above) — permissions cache, session cache, rate-limit counters, idempotency keys.
- Nothing cached in this repo (static docs).

## Authentication & Identity

All **specification only** (no auth code, no provider accounts in this repo).

**Auth Provider:**
- Keycloak self-hosted (default) + tenant IdP federation (Okta / Entra ID / Google Workspace /
  OneLogin) — see APIs section above.
- Implementation approach (binding): OAuth 2.1 + OIDC; Authorization Code + PKCE for both web and
  mobile public clients; **DPoP sender-constrained access tokens** (RFC 9449); 15-minute access
  TTL; 90-day single-use rotating refresh tokens with family revocation on reuse; 72-hour offline
  grace ceiling for field devices; mTLS via service mesh (Linkerd or Istio) + 5-minute internal
  JWTs for service-to-service; WebAuthn/passkeys preferred for web admin with step-up re-auth for
  sensitive actions. Full register: `docs/_context/engineering-decisions.md` §1.
- Agent authorization: every agent tool call passes an **OPA/Rego** policy evaluation (ADR-006,
  `docs/01-FSD.md`; autonomy levels L0–L4 in `docs/_context/conventions.md` § State machines).

## Monitoring & Observability

All **specification only**.

**Error Tracking:**
- Sentry (or Firebase Crashlytics) for mobile crash/ANR — mandatory from first mobile release
  (`docs/_context/engineering-decisions.md` §9). None installed here.

**Logs:**
- Spec: structured JSON logs with `trace_id`/`tenant_id`/`request_id`, shipped to Loki/ELK-class
  central store. Repo reality: no logs — the only observability artifacts are GSD's own hook
  caches (`~/.cache/gsd/`) and `.claude/gsd-migration-journal/`.

**Metrics/Tracing (spec):** OpenTelemetry SDK in every service → Tempo/Jaeger-class tracing,
Prometheus metrics, Grafana dashboards generated from each module's NFR/SLO tables
(`docs/01-FSD.md` §10, engineering-decisions §9). Alerting: Prometheus Alertmanager → PagerDuty /
Opsgenie.

## CI/CD & Deployment

All **specification only** — this repo has no pipelines.

**Hosting:**
- Spec target: Kubernetes (EKS/GKE-class), multi-region data planes with per-tenant residency
  pinning, Terraform + Helm + ArgoCD Application manifests in a planned `infra` repo
  (`docs/01-FSD.md` §1.2; `docs/_context/engineering-decisions.md` §4–6). This repo deploys
  nothing and hosts only documents.

**CI Pipeline:**
- None live. Spec: GitHub Actions per repo in the future polyrepo layout, with ktlint+detekt
  (Kotlin), ESLint+Prettier (TS), contract-diff gates against `platform-contracts`,
  Testcontainers integration suites; CD via ArgoCD GitOps with canary + auto-rollback; mobile via
  Fastlane with staged store rollouts (`docs/_context/engineering-decisions.md` §5, §7).

## Environment Configuration

**Required env vars:**
- **None.** No `.env` files exist (verified), and no application reads environment variables in
  this repo. The AI tooling needs no secrets to run.
- Spec-level config policy (for future implementation repos): 12-factor — non-secret config in
  Kubernetes ConfigMaps, secrets injected from HashiCorp Vault (default reference implementation)
  or cloud KMS + External Secrets Operator (`docs/_context/engineering-decisions.md` §8).

**Secrets location:**
- Repo: none — verified no `.env*`, `*secret*`, `*credential*`, or key files exist; `.gitignore`
  covers only `.DS_Store`/zip artifacts. Keep it that way: connector and webhook secrets are
  specified as **vault references only** (`credential_ref` columns, never inline secrets —
  `docs/modules/12-integrations-extensibility/low-level-design.md`).
- Spec: Vault database secrets engine issues short-lived per-service Postgres credentials; AES-256
  at rest with per-tenant DEK wrapped by platform KEK in cloud KMS for SENSITIVE fields
  (engineering-decisions §8).

## Webhooks & Callbacks

**Incoming (repo today):** None — nothing listens. (`_bmad-output/` and `.planning/` are consumed
by local CLI tooling, not HTTP.)

**Incoming (specified):**
- IoT HTTP webhook ingestion with payload signatures (module 17,
  `docs/modules/17-iot-sensor-platform/high-level-design.md`).
- SCIM 2.0 provisioning push from tenant IdPs (module 01).
- SAML/OIDC federation callbacks (module 01; engineering-decisions §1).
- Teams/Slack interactive-message actions translated into platform API calls under the invoking
  user's identity (module 12, FR-INT-014).

**Outgoing (repo today):** Only the GSD npm update check described above — not a webhook.

**Outgoing (specified) — Integration Hub webhook relay:**
- At-least-once delivery of canonical domain events to tenant-subscribed HTTPS URLs; retry
  schedule immediate → 30 s → 2 m → 10 m → 1 h → 6 h → daily up to 7 days → dead-letter visible in
  `GET /webhooks/{id}/deliveries` (`docs/01-FSD.md` §6.6).
- Every delivery signed: `X-Signature: hmac-sha256=...` over the raw body with a rotatable
  per-subscription secret (24-hour dual-secret grace window, FR-INT-021/022 in
  `docs/modules/12-integrations-extensibility/functional-spec.md`).
- `X-Event-Id` header for consumer-side dedup; auto-pause of subscriptions exceeding 50 % failure
  over a rolling hour (FSD §6.6).
- Delivery-log contract and manual replay of failed deliveries (FR-INT-023/024).
- Webhook design explicitly follows Stripe/GitHub/Slack signing conventions
  (`docs/modules/12-integrations-extensibility/high-level-design.md`).

---

*Integration audit: 2026-11-27*
