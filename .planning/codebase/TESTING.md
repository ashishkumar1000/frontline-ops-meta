# Testing Patterns

**Analysis Date:** 2026-11-27

> **Honest baseline:** this repository contains **no application source code, no tests, no test
> framework, and no CI configuration**. A repo-wide search found zero `*.test.*` / `*.spec.ts` /
> `conftest.py` / `*_test.go` files and zero runner configs (no `jest.config.*`, `vitest.config.*`,
> `pytest.ini`, etc.). The only package manifest is `.claude/package.json`
> (`{"type":"commonjs"}` — GSD tooling, not an app). There is no `.github/workflows/` directory.
>
> **However**, the testing standard for the future implementation is already **bindingly defined**
> in `docs/_context/engineering-decisions.md` (§5 CI/CD, §7 Testing Strategy and Gates, §11
> Definition of Done). This file records (a) what exists today — nothing — and (b) the spec-defined
> gates every future implementation must satisfy. Treat §7 of that register as the canonical test
> plan; this file summarizes it with pointers.

## Test Framework

**Runner (current):**
- None. No test runner, no test files, no CI in this repo (`README.md` states plainly: "This repo
  contains specifications, not source code").

**Planned/binding (from `docs/_context/engineering-decisions.md` §5 and §7):**
- CI: **GitHub Actions per repo** (polyrepo layout, §4). Every PR runs: lint (ktlint+detekt for
  Kotlin, ESLint+Prettier for TS), unit tests, a **contract-diff check against `platform-contracts`**
  (fails the build on an undeclared breaking change), and a Testcontainers integration suite.
- Run commands (once repos exist, per §5): `CI = lint + unit + contract-diff + Testcontainers
  integration` on every PR; `main` auto-deploys to dev; canary promotion gated on SLO burn-rate
  alerts per each module's NFR table.
- No test framework version is pinned in the specs by name for unit tests; the framework choice is
  per-language standard (Kotlin/TypeScript ecosystems), but the *gates* below are non-negotiable.

**Assertion Library:**
- Not yet chosen; spec requires machine-checkable gates (coverage %, contract diffs, NFR numbers)
  rather than a specific assertion library.

**Run Commands:**
```bash
# None exist today. Planned per engineering-decisions.md §5 (GitHub Actions, per repo):
#   PR gate:      lint + unit tests + contract-diff vs platform-contracts + Testcontainers suite
#   Release gate: k6 load test against staging targeting the module's NFR numbers (§7)
#   Quarterly:    scripted chaos run against staging (§7)
#   Mobile RC:    Maestro E2E on device farm per release-candidate build (§7)
```

## Test File Organization

**Location:**
- Not applicable yet. Planned repos that will carry tests (`engineering-decisions.md` §4):
  one repo per service (`svc-*`), `mobile-app`, `web-admin`, `platform-contracts` (contract
  verification tests live with every consumer), `infra`.

**Naming:**
- Not pinned by the specs. Use each language's ecosystem default when repos are created; the
  spec-level requirement is that **every documented error code and every state-machine transition
  has a named automated test** (§7, §11), not a specific filename pattern.

**Structure:**
```
Planned (per engineering-decisions.md §4):
  platform-contracts/   # OpenAPI 3.1 + event JSON Schema + generated SDKs (contract-test source of truth)
  svc-<module>/         # unit + Testcontainers integration + k6 scripts per module
  mobile-app/           # Maestro YAML E2E flows (offline fill→sync, multi-level approval, forwarding)
  web-admin/            # axe-core WCAG 2.1 AA checks in CI
  infra/                # Terraform/Helm/ArgoCD (no unit tests; validated in ephemeral dev namespaces)
```

## Test Structure

**Suite Organization:**
No suites exist. The spec defines what suites must cover, per layer (§7):
- Unit suites: domain/business-logic packages only — "rules engines, state machines, policy
  evaluation" — with generated code (OpenAPI clients, DB migration scaffolding) excluded from the gate.
- Integration suites: real Postgres + Kafka + Redis via Testcontainers, covering **every
  state-machine transition** (WorkItem, Issue, TemplateVersion machines fixed in
  `docs/_context/conventions.md`) and **every documented error code** in each module's
  low-level-design.md Error Handling table (e.g.
  `docs/modules/03-forms-workflow-builder/low-level-design.md` §11 — "those tables are the test
  plan, not just documentation").

**Patterns (spec-mandated, from `docs/_context/engineering-decisions.md` §11 Definition of Done):**
- Setup: Testcontainers spins ephemeral real Postgres/Kafka/Redis per suite run — **not mocks** for
  anything touching persistence or eventing.
- Teardown: ephemeral containers/PR namespaces (dev env per §6: per-PR preview namespaces spun up on
  PR open, torn down on merge/close).
- Assertion: a new documented error code must have an automated test proving it is returned under
  the stated condition; audit-log production for mutating actions is **checked in the integration
  test, not assumed** (DoD items 3 and 6).

## Mocking

**Framework:** none today; planned stance is **minimal mocking**.

**Patterns:**
```text
engineering-decisions.md §7 (binding): "Testcontainers-backed, real Postgres + Kafka + Redis per
test suite run — not mocks, for anything touching persistence or eventing."
```

**What to Mock:**
- External third-party model providers behind the LiteLLM-style gateway (Agent Runtime, module 10)
  and external SaaS APIs (Integration Hub, module 12) — vendor systems outside platform control.
- Push transport (FCM/APNs) at the edges of the Notification Hub.

**What NOT to Mock:**
- Postgres, Kafka, Redis — real containers required (§7).
- The offline sync path end-to-end — must be exercised by device-level E2E with real network
  conditioning (Maestro's built-in network simulation was selected for exactly this, §7).
- Anything with a documented NFR — proven with k6 numbers against staging, not asserted in a unit
  test ("a release that regresses a documented NFR fails the release gate", §7).

## Fixtures and Factories

**Test Data:**
- Staging is populated with **synthetic tenants and anonymized/synthetic data only** — "never a copy
  of real tenant data, even redacted" (§6; hard rule due to PII/health/safety categories in modules
  18, 20, 21). Build synthetic-tenant fixtures accordingly.
- PII handling precedent for fixture design: for `pii_class=SENSITIVE` fields, exclude rather than
  redact (resolved decision in `docs/modules/03-forms-workflow-builder/functional-spec.md` §11).

**Location:**
- Not defined yet; per-repo once implementation starts (spec leaves fixture location to each repo).

## Coverage

**Requirements:** **minimum 80% line coverage on domain/business-logic packages** (rules engines,
state machines, policy evaluation), excluding generated code (OpenAPI clients, DB migration
scaffolding). "Coverage gate enforced in CI, not aspirational" (`engineering-decisions.md` §7).
Nothing enforces it today because no code or CI exists.

**View Coverage:**
```bash
# Not available — no test suite exists in this repo yet.
# Planned: coverage report produced by the per-repo CI unit-test job; gate enforced there (§7).
```

## Test Types

**Unit Tests:**
- ≥80% line coverage on domain-logic packages; CI-enforced. Focus: rule evaluation engines
  (e.g. the conditional-visibility/validation engine specified in
  `docs/modules/03-forms-workflow-builder/low-level-design.md` §7.1), state-machine transitions,
  OPA/Rego policy evaluation, scheduling recurrence expansion, SLA escalation timers.

**Contract Tests:**
- Every service publishes its OpenAPI spec to `platform-contracts`; every consumer runs a
  contract-verification test against the latest compatible version **on every CI run** — Pact-style
  consumer-driven contracts or an OpenAPI-diff gate (OpenAPI-diff is the stated lower-effort default)
  (§7). CI also runs a contract-diff check that fails on undeclared breaking changes (§5). Event
  schemas evolve backward-compatibly within a major version; breaking change = new versioned
  `event_type` (§3).

**Integration Tests:**
- Testcontainers (real Postgres + Kafka + Redis); minimum scope = every state-machine transition
  and every documented error code per module LLD Error Handling table (§7).

**E2E Tests:**
- Mobile: **Maestro** (chosen over Detox for declarative YAML flows + network-condition simulation),
  against a device farm (Firebase Test Lab for Android; macOS runner + iOS Simulator / App Store
  Connect farm for iOS), on every release-candidate build. Three canonical "hard" flows are required:
  (1) full offline fill → reconnect → sync, (2) the multi-level approval flow, (3) cross-template
  forwarding in both `INLINE_CHAIN` and `SPAWN_WORK_ITEM` modes (§7).

**Load Tests:**
- **k6** scripts per module, run against staging before any release touching a module's hot path,
  targeting the exact NFR numbers in that module's functional spec (e.g. `NFR-FORM-04`/`NFR-FORM-05`
  in `docs/modules/03-forms-workflow-builder/functional-spec.md`; Work Orchestration's
  assignment-latency NFR). A regression against a documented NFR fails the release gate (§7).

**Chaos Tests:**
- Quarterly, scripted, against staging: Kafka broker kill, Postgres primary failover, Redis eviction
  storm — verifying each module's "Failure Modes and Degradation" table holds under real fault
  injection (§7).

**Accessibility Tests:**
- Web Admin Console: WCAG 2.1 AA via **axe-core** in CI, plus a manual screen-reader pass before each
  quarterly release (§7); DoD item 7 requires the automated check to pass for any Web Admin story.

## Common Patterns

**Acceptance criteria as executable specifications (documentation-level, in force today):**
Every story in `docs/02-Epics-and-Stories.md` is written in Given/When/Then form, e.g.
`EPIC-01.S-01` ("Given a valid provisioning request, when the tenant is created, then a `Tenant` row
is created with `id`, `status=ACTIVE`, …") and `EPIC-01.S-02` ("Given an attempt to create a cycle …
then the API returns `422` with code `INVALID_HIERARCHY`"). These criteria are the direct source for
future test cases; requirement IDs (`FR-<MOD>-###`, `EPIC-xx.S-yy`) are the traceability keys.

**Error Testing (planned pattern, §7 + DoD item 3):**
```text
For each row in a module's LLD Error Handling Table (e.g.
docs/modules/03-forms-workflow-builder/low-level-design.md §11):
  assert the documented HTTP status + stable error code is returned under the stated
  detection-point condition (e.g. duplicate client_uuid with identical payload → 200
  duplicate:true; different payload → 409 IDEMPOTENCY_KEY_REUSED_DIFFERENT_PAYLOAD).
```

**Offline Testing (planned pattern, DoD item 5):**
Any story touching client-facing behavior must explicitly verify: what happens with no
connectivity, and what happens on reconnect — "didn't think about offline" is not an acceptable
state. Verify against the 72-hour offline grace window rules (`engineering-decisions.md` §1).

**Definition of Done — test-related gates (§11, all modules):** unit + contract + relevant
integration tests green; new error codes tested; k6 result attached to the PR for perf-sensitive
stories; offline behavior verified; audit-log production checked in integration tests; WCAG 2.1 AA
automated check passing for Web Admin stories; and any behavioral deviation from the spec updates
the corresponding `functional-spec.md`/`low-level-design.md` **in the same PR** (DoD item 8).

---

*Testing analysis: 2026-11-27*
