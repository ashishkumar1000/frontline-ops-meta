# Technology Stack

**Analysis Date:** 2026-11-27

> **Repository nature (read this first):** This is a **specification-only repository**. It contains
> no application source code, no dependency manifests, no build system, and no tests. The
> deliverable is a build-ready markdown documentation package (135 `.md` files under `docs/`)
> describing a future platform, plus newly installed AI-engineering tooling (BMAD + GSD). The
> sections below record what actually exists today; the final section records the *target*
> application stack as fixed by the binding spec documents — it is specification content, not
> implemented code. Future GSD phases must treat "nothing exists yet" as the starting baseline.

## Languages

**Primary:**
- **Markdown** — the entire repo deliverable: 135 `.md` files under `docs/` (master PRD
  `docs/00-PRD.md`, platform FSD `docs/01-FSD.md`, backlog `docs/02-Epics-and-Stories.md`,
  shared registers in `docs/_context/`, and 25 module folders under `docs/modules/` each holding
  `README.md`, `functional-spec.md`, `high-level-design.md`, `low-level-design.md`,
  `architecture-diagram.md`). Root-level legacy drafts also exist: `functional-spec_1.md`,
  `high-level-design.md`, `low-level-design_1.md`, `architecture-diagram_1.md` (Module 03 history;
  canonical versions live in `docs/modules/03-forms-workflow-builder/`).
- **Mermaid** (diagrams-as-code, embedded in Markdown fences) — all architecture, sequence, ER and
  state diagrams across `docs/`, including all 25 `docs/modules/*/architecture-diagram.md` files.
  Renders natively on GitHub/GitLab/VS Code per `docs/_context/conventions.md` (§ Diagram
  conventions). New docs must use Mermaid, not images.

**Secondary (tooling scripts only — no application code):**
- **JavaScript (CommonJS)** — GSD hooks and helpers: `.claude/hooks/gsd-check-update.js`,
  `gsd-context-monitor.js`, `gsd-prompt-guard.js`, `gsd-read-guard.js`, `gsd-read-injection-scanner.js`,
  `gsd-statusline.js`, `gsd-update-banner.js`, `gsd-check-update-worker.js`, and
  `.claude/get-shit-done/bin/*.cjs` (`gsd-tools.cjs`, `check-latest-version.cjs`,
  `verify-reapply-patches.cjs`). `.claude/package.json` contains only `{"type":"commonjs"}` to set
  module scope for these scripts.
- **Bash** — GSD shell hooks: `.claude/hooks/gsd-session-state.sh`, `gsd-phase-boundary.sh`,
  `gsd-validate-commit.sh`, `gsd-workflow-guard.sh`-family wired in `.claude/settings.json`.
- **Python** — BMAD installer/config scripts: `_bmad/scripts/render_skill.py`,
  `resolve_customization.py`, `resolve_config.py`, `memlog.py`, `config_utils.py`.
- **TOML / YAML / JSON (config, not code)** — `_bmad/config.toml`, `_bmad/config.user.toml`,
  `_bmad/custom/config.toml`, `_bmad/core/config.yaml`, `_bmad/bmm/config.yaml`,
  `_bmad/_config/manifest.yaml`, `_bmad/_config/*.csv`, `.claude/settings.json`,
  `.claude/gsd-install-state.json`, `.claude/gsd-file-manifest.json`.
- **TypeScript / JSON (spec content only)** — API request/response examples, JSON Schema-style
  event envelopes, and OpenAPI conventions are *described* inside the spec docs (e.g.
  `docs/01-FSD.md` §6.2/§6.4); no `.ts` files exist anywhere in the repo.

## Runtime

**Environment:**
- **No application runtime.** Nothing in this repo compiles, builds, serves, or executes as a product.
- Tooling runtimes present on the machine:
  - **Node.js v25.9.0** — invoked by absolute path in `.claude/settings.json` hook commands
    (`/Users/admin/.nvm/versions/node/v25.9.0/bin/node`) for all `.js` GSD hooks.
  - **Python 3** — for the `_bmad/scripts/*.py` helpers (no pinned version file; no `requirements.txt`).
  - **Bash** — for `.sh` GSD hooks.

**Package Manager:**
- **None.** No root `package.json`, no lockfile, no `requirements.txt` / `pyproject.toml` / `go.mod`
  anywhere in the repo (verified by filesystem search). The only manifest is
  `.claude/package.json`, which is a one-line `{"type":"commonjs"}` marker, not a dependency list.
- Lockfile: **missing / not applicable**.

## Frameworks

**Core (application):**
- **None.** No framework code exists. The *target* framework set is specified (see final section)
  but has zero implementation.

**Testing:**
- **None installed.** The testing strategy is fully specified in
  `docs/_context/engineering-decisions.md` §7 (80% unit-coverage gate, Pact/OpenAPI-diff contract
  tests, Testcontainers integration suites, Maestro mobile E2E, k6 load tests, quarterly chaos
  drills, axe-core accessibility) — all of it is future work, nothing runnable today.

**Build/Dev (AI-engineering tooling, installed and live):**
- **GSD (get-shit-done) v1.42.3** — version recorded in `.claude/get-shit-done/VERSION`.
  Installed at `.claude/` with 33 subagent definitions in `.claude/agents/`, slash commands in
  `.claude/commands/gsd/` (e.g. `add-tests.md`, `audit-fix.md`, `code-review.md`,
  `complete-milestone.md`), workflow playbooks in `.claude/get-shit-done/workflows/`, hook scripts
  in `.claude/hooks/`, and install state in `.claude/gsd-install-state.json` (baseline scan
  migration applied 2026-11-26; profile `.claude/.gsd-profile` = `full`).
- **BMAD Method (BMM module, v6 layout)** — installed at `_bmad/` with module config
  (`_bmad/bmm/config.yaml`, `_bmad/core/config.yaml`), agent personas defined in
  `_bmad/config.toml` (Analyst "Mary", PM "John", UX "Sally", Architect "Winston", Dev "Amelia"),
  team overrides in `_bmad/custom/config.toml`, and output routed to the empty `_bmad-output/`
  directory. Skill mirror exposed in `.claude/skills/` (29 `bmad-*` skills) and `.zcode/skills/`.
- **ZCode skills** — `.zcode/skills/` mirrors the 29 `bmad-*` skills for the ZCode CLI runtime.
- **Mermaid rendering** — native in GitHub; no local build step required.
- **Git** — repository metadata present at `.git/` (the repo is version-controlled; git operations
  in GSD flows are handled by the orchestrator, not mappers).

## Key Dependencies

**Critical:**
- **None.** There is no dependency manifest and no vendored library. Do not add runtime
  dependencies to this repo — it produces documents, not binaries.

**Infrastructure (tooling state files, informational):**
- `.claude/gsd-file-manifest.json` — inventory of every file GSD installed (44 KB).
- `.claude/gsd-install-state.json` — schemaVersion 1; tracks applied GSD migrations (first-time
  baseline scan, 2026-11-26) with checksums; journal at `.claude/gsd-migration-journal/`.
- `_bmad/_config/manifest.yaml`, `files-manifest.csv`, `skill-manifest.csv`, `bmad-help.csv` —
  BMAD installer manifests (treat as generated; regenerate via the BMAD installer, not by hand).
- `_bmad/config.toml` header explicitly says it is installer-managed and regenerated on every
  install — durable overrides go in `_bmad/custom/config.toml` (committed) or
  `_bmad/custom/config.user.toml` (gitignored by `_bmad/custom/.gitignore`).

## Configuration

**Environment:**
- **No `.env` files exist** (verified: no `.env*` anywhere). No environment variables are required
  to read or edit this repo.
- Spec-level environment/config policy is *documented*, not implemented: 12-factor config via
  Kubernetes ConfigMaps (non-secret) + Vault-injected secrets, in
  `docs/_context/engineering-decisions.md` §8.
- Project-level instruction files that govern AI agents in this repo: `CLAUDE.md` and `AGENTS.md`
  at the repo root (32 lines each; they define the mandatory BMAD-planning → GSD-execution routing
  and treat `docs/` + `.planning/codebase/` as the source of truth).

**Build:**
- No build config files (no `tsconfig.json`, no bundler config, no Makefile, no docker-compose).
- The only behavioral config is AI-tooling config: `.claude/settings.json` (SessionStart,
  PreToolUse, PostToolUse hook wiring with 5–10 s timeouts), `_bmad/config.toml` +
  `_bmad/custom/config.toml`, and `.gitignore` (ignores `.DS_Store`, `*.zip`, `*.zip.tmp`;
  `_bmad/render/.gitignore` and `_bmad/custom/.gitignore` ignore BMAD render/user-override output).

## Platform Requirements

**Development:**
- Any machine with: a Markdown editor, a Mermaid-capable previewer (GitHub renders all diagrams
  natively), `git`, and a Claude Code / ZCode CLI installation to run the installed GSD hooks and
  BMAD skills (hooks need Node.js v25.x on PATH at the path hard-coded in `.claude/settings.json`).
- No database, no container runtime, no cloud account required for this repo today. (CI-relevant
  test tooling — Testcontainers, k6, Maestro — is specified for the future implementation repos,
  not this one.)

**Production:**
- **N/A — nothing deploys from this repo.** The production deployment target is *specified* for
  the future platform: Kubernetes (EKS/GKE-class) with per-service Helm charts, Terraform IaC in a
  planned `infra` repo, ArgoCD GitOps with Argo Rollouts canary (10 %/15 min auto-promote),
  multi-region tenant-residency data planes — see `docs/01-FSD.md` §1.2 and
  `docs/_context/engineering-decisions.md` §4–§6. The spec also fixes a **polyrepo** strategy
  (`platform-contracts`, one repo per service, `mobile-app`, `web-admin`, `infra`), meaning code
  written from these specs will NOT live in this repository.

## Target Application Stack (Specification — Not Yet Implemented)

This is the binding target stack every module doc assumes and no future phase may reinvent
(source: `docs/_context/conventions.md` § Target Tech Stack; deepened by ADRs in `docs/01-FSD.md`
§2 and `docs/_context/engineering-decisions.md`):

| Layer | Specified technology | Where specified |
|---|---|---|
| Mobile app | React Native + TypeScript, native Kotlin/Swift modules, WatermelonDB local store, WorkManager/BGTaskScheduler, FCM + APNs | `docs/_context/conventions.md`; ADR-005 in `docs/01-FSD.md` |
| Web admin | React + TypeScript (Next.js-class), BFF-driven GraphQL/REST | `docs/_context/conventions.md`; `docs/_context/engineering-decisions.md` §4 |
| Backend services | Kotlin + Spring Boot 3.x on Java 21 LTS; modular monolith at bounded-context boundaries (13 core services), gRPC internal, REST external | ADR-001 `docs/01-FSD.md`; `docs/_context/engineering-decisions.md` §10 |
| API Gateway / BFF | Node.js (TypeScript) or Spring Cloud Gateway | ADR-008 `docs/01-FSD.md` |
| Primary datastore | PostgreSQL (Aurora/Cloud SQL-class) with `tenant_id` + Row-Level Security on every table | ADR-002 `docs/01-FSD.md`; `docs/_context/conventions.md` |
| Time-series | TimescaleDB hypertable on the same Postgres fleet (IoT module) | ADR-009 `docs/01-FSD.md` |
| Vectors / RAG | pgvector in the tenant-scoped Postgres tier, `VectorStore` port escape hatch | ADR-007 `docs/01-FSD.md` |
| Object storage | S3-compatible (S3/GCS), per-tenant prefix, KMS-encrypted, signed URLs | `docs/_context/conventions.md` |
| Event backbone | Apache Kafka (MSK/Confluent-class), partition key `tenant_id:aggregate_id`, schema registry | ADR-003 `docs/01-FSD.md` |
| Workflow engine | Temporal.io for all durable/timer orchestration | ADR-004 `docs/01-FSD.md` |
| Search | OpenSearch | `docs/_context/conventions.md` |
| Cache | Redis (permissions, sessions, rate limits, idempotency) | `docs/_context/conventions.md` |
| Policy engine | OPA / Rego (agent authorization + ABAC) | ADR-006 `docs/01-FSD.md` |
| AI runtime | LiteLLM-style model gateway over OpenAI/Anthropic/Azure/Bedrock | `docs/_context/conventions.md`; `docs/modules/10-agentic-operations/high-level-design.md` |
| Analytics | Debezium CDC → lakehouse (Snowflake/BigQuery/Databricks-class) | `docs/_context/conventions.md` |
| AuthN/AuthZ | OAuth 2.1 + OIDC, DPoP (RFC 9449), Keycloak default IdP, mTLS + internal JWTs | `docs/_context/engineering-decisions.md` §1 |
| Deploy | Kubernetes, Terraform, Helm, ArgoCD + Argo Rollouts | `docs/_context/conventions.md`; §5–6 `docs/_context/engineering-decisions.md` |
| Observability | OpenTelemetry → Grafana/Datadog-class, Prometheus, structured JSON logs, Sentry/Crashlytics mobile | `docs/_context/engineering-decisions.md` §9 |

**Conflict note for future phases:** `docs/_context/conventions.md` says "Kotlin + Spring Boot
microservices"; ADR-001 in `docs/01-FSD.md` (the platform-wide FSD, dated as the deepened
baseline) narrows this to a **modular monolith at bounded-context boundaries** (13 services, one
deployable each). Where the two differ, the FSD ADR is the later, more specific decision — follow
ADR-001 and do not re-open the question.

---

*Stack analysis: 2026-11-27*
