# AGENTS.md — Frontline Ops Platform (Spec Repository)

This repository is a **build-ready specification package** (markdown docs, no application source
code yet) for a mobile-first B2B SaaS frontline-operations platform. Documentation index:
`README.md` → `docs/README.md` → `docs/modules/README.md` (25 modules). Shared rules live in
`docs/_context/conventions.md` and the **binding** `docs/_context/engineering-decisions.md`.

This workspace uses a two-layer AI engineering workflow: **BMAD** (planning/discovery) + **GSD**
(code execution). The routing rules below are mandatory for this agent and all subagents.

---
## Workflow Routing Rules: BMAD vs. GSD (Mandatory)

We use a strict two-stage workflow. Never mix their responsibilities:

### 1. When to Use BMAD (Planning, Discovery & Major Architecture Only)
- **Trigger:** Starting a major new module/epic, redesigning system architecture, or working from a vague idea where requirements and edge cases are not yet locked down.
- **What to do:** Invoke BMAD's **Analyst, PM, or Architect** personas only to interrogate requirements and output formal specs into `docs/PRD.md`, `docs/architecture.md`, or `docs/epics/`.
- **Hard Stop:** NEVER use BMAD's developer/implementation agents to write application code. Once specs are saved in `docs/`, stop BMAD.

### 2. When to Use GSD (All Code Execution, Features & Refactors)
- **Trigger:** Any time code needs to be written, refactored, debugged, or implemented—whether from a new BMAD spec or an existing feature request.
- **What to do:** Run the isolated GSD execution loop:
  1. `/gsd:discuss-phase [N]` — Clarify edge cases against `docs/architecture.md` and `.planning/codebase/`.
  2. `/gsd:plan-phase [N]` — Break work into atomic, verifiable tasks.
  3. `/gsd:execute-phase [N]` — Execute tasks in fresh, isolated context windows to prevent context rot.
  4. `/gsd:verify-work [N]` — Run automated tests, typechecks, and linting before marking complete.

### 3. Quality Guardrails
- Treat `docs/architecture.md`, `docs/PRD.md`, and `.planning/codebase/` as the strict source of truth.
- Never mark a GSD task complete without passing unit/integration tests, linter, and typechecks.
- Keep git commits atomic and scoped to the active GSD task.

---
## Control-Plane Rules (Metarepo, Structural — Mandatory)

This repo is the **control plane** for the Frontline Ops child repos. No child repos exist yet;
when they do, they are registered in `docs/repo-catalog.yaml` and cloned by
`bun run workspace:setup` into `workspace/core/<role>/<repo-id>` as **independent git repos**.

- **Commit in the repo the change belongs to — never from the meta root.** Files under
  `workspace/core/**` or `workspace/external/**` → run git inside that child repo (`git -C <path>`),
  push to its own remote. Files under `docs/`, `scripts/`, or other meta-only paths → commit here
  in `frontline-ops-meta`. Never `git add -A && git commit` from the meta root — child repos are
  gitignored here on purpose; the `hooks/pre-commit` net rejects violations.
- **A change spanning two repos = two commits in two repos**, optionally noted together under
  `docs/initiatives/`. Same-session edits never mean "ships together" — always state which repo
  merges/deploys first.
- **Cross-repo change ordering.** Child repos deploy independently. Additive change (new
  field/endpoint, existing behavior untouched) → producing side (backend/contracts) merges/deploys
  first, consumers after. Breaking change → three steps: (1) producer adds the new shape alongside
  the old, (2) consumers switch, (3) producer removes the old shape in a separate change.
- **Registering a repo:** add an entry to `docs/repo-catalog.yaml` (id, path, remote, role,
  default_branch, status, docs_dir) + create `docs/repos/<id>/`; re-run `bun run workspace:setup`.
  Per-repo durable notes live in `docs/repos/<id>/`; cross-repo work notes in `docs/initiatives/`.
- **Push orchestration:** `bun run workspace:push` pushes backend/contract repos first, then
  others, then this meta repo last. It never force-pushes; diverged/dirty repos are skipped with
  the fix command shown.
- **Bun only** for this repo's tooling — never npm/yarn/pnpm; never commit their lockfiles.

---
## Where BMAD & GSD Run (Metarepo Mapping — Mandatory)

The routing rules above were written for a single repo. In this metarepo they apply like this:

- **BMAD (planning, discovery, architecture) runs at this meta root.** Product-level specs are
  control-plane knowledge — they land in `docs/` (the spec package, e.g. `docs/00-PRD.md`,
  `docs/01-FSD.md`, module folders) or `docs/repos/<id>/` for per-child planning. Never inside a
  child repo.
- **GSD (all code execution) runs inside the child repo the work belongs to**
  (`workspace/core/<role>/<repo-id>`). On first use of a child repo, run `/gsd:new-project`
  *inside it* so it gets its own `.planning/` baseline; commits and quality gates (tests, lint,
  typechecks) run with that repo's own toolchain.
- This meta repo's `.planning/codebase/` maps the **spec package only** — it is not a baseline
  for child-repo code, and must not be used as the source of truth for it.
- Meta-root commits stay scoped to control-plane files (`docs/`, `scripts/`, `hooks/`, catalog);
  child-repo work is committed inside the child repo (see Control-Plane Rules above).
