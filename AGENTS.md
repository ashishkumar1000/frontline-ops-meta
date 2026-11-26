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
