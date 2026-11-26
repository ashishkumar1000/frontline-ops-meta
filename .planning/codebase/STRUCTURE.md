# Codebase Structure

**Analysis Date:** 2026-11-27

## Critical Context

This is a **specification-only repository**: 768 markdown files, zero application source code, no
package manifests, no build/test configuration, no runtime entry points. The structure below is the
actual layout of the documentation package and its AI tooling. Names like `svc-work-orchestration`
or `mobile-app` appear only as *specified future repos* inside
`docs/_context/engineering-decisions.md` §4 — they do not exist here.

## Directory Layout

```text
frontline-ops-meta/                       # repo root (spec package + AI tooling)
├── README.md                             # repo index & reading order (entry point)
├── CLAUDE.md                             # mandatory BMAD-vs-GSD AI routing rules
├── .gitignore                            # excludes .DS_Store, *.zip, *.zip.tmp
├── architecture-diagram_1.md             # LEGACY Module 03 draft (superseded)
├── functional-spec_1.md                  # LEGACY Module 03 draft (superseded)
├── high-level-design.md                  # LEGACY Module 03 draft (superseded)
├── low-level-design_1.md                 # LEGACY Module 03 draft (superseded)
├── docs/                                 # CANONICAL specification package
│   ├── README.md                         # file-by-file doc index + reading order
│   ├── 00-PRD.md                         # Master PRD v3.0 (703 lines)
│   ├── 01-FSD.md                         # Platform FSD v2.1 (1854 lines; arch, ADRs, domain model)
│   ├── 02-Epics-and-Stories.md           # Backlog: 15 epics → 154 stories (1405 lines)
│   ├── _context/                         # shared reference layer (assumed, never repeated)
│   │   ├── README.md                     # layer index + binding rules
│   │   ├── source-spec.md                # original v2.0 PRD/FSD baseline (1053 lines)
│   │   ├── conventions.md                # house rules: stack, entities, formats (138 lines)
│   │   ├── engineering-decisions.md      # BINDING decisions register (302 lines)
│   │   └── new-modules-addendum.md       # modules 14–25 scope & integration map
│   └── modules/                          # 25 module folders + catalog
│       ├── README.md                     # module catalog (one-liner per module)
│       ├── 01-identity-organization/     # ┐
│       ├── 02-work-orchestration/        # │ 25 folders, each with EXACTLY:
│       ├── …                             # │ README.md · functional-spec.md
│       ├── 10-agentic-operations/        # │ high-level-design.md
│       ├── …                             # │ low-level-design.md
│       └── 25-field-service-vendor-scheduling/  # ┘ architecture-diagram.md
├── .claude/                              # AI engineering tooling (GSD + BMAD skills)
│   ├── agents/                           # 33 gsd-* subagent definitions
│   ├── commands/gsd/                     # 67 /gsd:* slash commands
│   ├── get-shit-done/                    # GSD framework v1.42.3
│   │   ├── VERSION                       # "1.42.3"
│   │   ├── bin/                          # gsd-tools.cjs + helpers
│   │   ├── contexts/                     # dev.md, research.md, review.md
│   │   ├── references/                   # ~55 framework reference docs
│   │   ├── templates/                    # doc templates incl. templates/codebase/
│   │   └── workflows/                    # ~80 workflow definitions
│   ├── hooks/                            # 11 GSD hook scripts (js/sh)
│   └── skills/                           # 36 bmad-* skills
├── .zcode/
│   └── skills/                           # identical copy of the 36 bmad-* skills
├── _bmad/                                # BMAD runtime configuration
│   ├── config.toml / config.user.toml
│   ├── _config/                          # manifest.yaml + CSV manifests
│   ├── bmm/ · core/                      # BMAD module configs + v6-shims
│   ├── custom/                           # config overrides
│   ├── render/                           # (empty render output)
│   └── scripts/                          # 5 Python helper scripts (tooling only)
├── _bmad-output/                         # BMAD output drop zone (currently empty)
└── .planning/                            # GSD planning artifacts
    └── codebase/                         # codebase maps (ARCHITECTURE.md, STRUCTURE.md)
```

## Directory Purposes

**`docs/`:**
- Purpose: The canonical, build-ready specification package — the product this repo exists to hold.
- Contains: Three top-level spec documents, the shared reference layer, and 25 module folders.
- Key files: `docs/00-PRD.md`, `docs/01-FSD.md`, `docs/02-Epics-and-Stories.md`, `docs/README.md`

**`docs/_context/`:**
- Purpose: Shared foundation every other document is written against; holds the binding rules.
- Contains: Baseline spec, conventions, binding engineering decisions, modules 14–25 addendum, layer README.
- Key files: `docs/_context/engineering-decisions.md` (BINDING), `docs/_context/conventions.md` (stack/entity/format rules), `docs/_context/source-spec.md` (scope/terminology baseline)

**`docs/modules/`:**
- Purpose: Per-module build-ready specifications for all 25 platform modules, plus the catalog.
- Contains: `README.md` catalog + 25 identically-shaped folders (125 spec files, ~42,900 lines).
- Key files: `docs/modules/README.md`; per folder `README.md`, `functional-spec.md`, `high-level-design.md`, `low-level-design.md`, `architecture-diagram.md`

**`.claude/`:**
- Purpose: AI engineering tooling for this workspace: GSD execution framework + BMAD skills + hooks.
- Contains: 33 subagent defs (`.claude/agents/`), 67 commands (`.claude/commands/gsd/`), GSD framework v1.42.3 (`.claude/get-shit-done/`), 11 hooks (`.claude/hooks/`), 36 skills (`.claude/skills/`).
- Key files: `.claude/agents/gsd-codebase-mapper.md` (wrote this map), `.claude/get-shit-done/VERSION`, `.claude/commands/gsd/map-codebase.md`

**`.zcode/skills/`:**
- Purpose: Same 36 BMAD skills exposed to the ZCode CLI; byte-identical to `.claude/skills/` (verified via `diff -rq`).
- Contains: Mirror of the BMAD skill tree only (no agents/commands duplication).
- Key files: `.zcode/skills/bmad-architecture/SKILL.md` (representative)

**`_bmad/`:**
- Purpose: BMAD framework runtime configuration and helper scripts.
- Contains: `config.toml` / `config.user.toml`, manifest CSVs in `_bmad/_config/`, module configs in `_bmad/bmm/` and `_bmad/core/` with `v6-shims/`, custom overrides in `_bmad/custom/`, 5 Python scripts in `_bmad/scripts/` (`config_utils.py`, `memlog.py`, `render_skill.py`, `resolve_config.py`, `resolve_customization.py`). These scripts are tooling, not application code.
- Key files: `_bmad/config.toml`, `_bmad/_config/manifest.yaml`

**`_bmad-output/`:**
- Purpose: Drop zone for BMAD-generated artifacts.
- Contains: Nothing (empty).
- Key files: none

**`.planning/`:**
- Purpose: GSD planning artifacts (created by `/gsd:*` commands and their subagents).
- Contains: `codebase/` with this map. Expected future home for phase plans and milestones per `CLAUDE.md`'s GSD loop.
- Key files: `.planning/codebase/ARCHITECTURE.md`, `.planning/codebase/STRUCTURE.md`

## Key File Locations

**Entry Points:**
- `README.md`: Human entry point — repo index, product summary, canonical reading order.
- `CLAUDE.md`: AI agent entry point — auto-loaded workspace rules; mandatory BMAD/GSD routing.
- `docs/README.md`: Documentation index — file-by-file map with "How to read this package" order.
- `docs/modules/README.md`: Module catalog — the way into any of the 25 modules.
- `docs/_context/README.md`: Shared-reference-layer index and its binding rules.
- Note: **No code entry points exist** (no `main`, no server bootstrap, no deployable).

**Configuration:**
- `CLAUDE.md`: Workspace-level AI behavior configuration (routing, guardrails, source-of-truth designation).
- `_bmad/config.toml`, `_bmad/config.user.toml`: BMAD framework configuration.
- `_bmad/_config/manifest.yaml`, `_bmad/_config/files-manifest.csv`, `_bmad/_config/skill-manifest.csv`: BMAD installation manifests.
- `.claude/get-shit-done/VERSION`: GSD framework version (1.42.3).
- `.gitignore`: Ignores `.DS_Store`, `*.zip`, `*.zip.tmp`. No `.env` or secret files present anywhere in the tree.

**Core Logic:**
- `docs/01-FSD.md`: The platform's "core" — target architecture (§1), ADR-001…009 (§2), service inventory (§3), domain model (§4), state machines (§5), API/event contracts (§6), offline sync (§7), AI governance (§8), security (§9).
- `docs/_context/conventions.md`: The rules every document must obey (stack, scoping, entities, API shapes, Mermaid format, depth expectations).
- `docs/_context/engineering-decisions.md`: The binding register (auth, repos, CI/CD, testing gates, DoD, EU AI Act).
- `docs/modules/02-work-orchestration/`: The spec for the platform's execution engine (WorkItem lifecycle) that every other module builds on.
- `docs/modules/01-identity-organization/`: The spec for the gating module (tenants, org tree, scoping) every module consumes.

**Testing:**
- Not applicable — no tests, test configs, or test runners exist in the repo. Testing *requirements for the future implementation* are specified in `docs/_context/engineering-decisions.md` §7 ("Testing Strategy and Gates").

## Naming Conventions

**Files:**
- Numbered top-level specs: `NN-ShortName.md` — `docs/00-PRD.md`, `docs/01-FSD.md`, `docs/02-Epics-and-Stories.md`
- Fixed module quartet (exact names, lowercase kebab-case): `functional-spec.md`, `high-level-design.md`, `low-level-design.md`, `architecture-diagram.md`
- Legacy root drafts use a `_1` suffix: `functional-spec_1.md`, `low-level-design_1.md`, `architecture-diagram_1.md` (superseded — do not edit)
- GSD subagents: `gsd-<role>.md` (e.g., `.claude/agents/gsd-codebase-mapper.md`); GSD commands: `<verb-or-noun>.md` under `.claude/commands/gsd/` (e.g., `map-codebase.md`, `execute-phase.md`)
- BMAD skills: `bmad-<topic>/SKILL.md` (e.g., `.claude/skills/bmad-architecture/SKILL.md`)
- Codebase map docs: UPPERCASE.md in `.planning/codebase/` (e.g., `ARCHITECTURE.md`, `STRUCTURE.md`)

**Directories:**
- Module folders: `NN-kebab-case-name/` with zero-padded two-digit prefix — `01-identity-organization/` through `25-field-service-vendor-scheduling/` (number = module number, NOT epic number; epic↔module mapping is N:1 and lives in `docs/README.md` table)
- GSD framework subdirs are lowercase singular: `bin/`, `contexts/`, `references/`, `templates/`, `workflows/`
- Underscore prefix marks BMAD-owned trees: `_bmad/`, `_bmad-output/`

**In-document identifiers (for writing new spec content):**
- Functional requirements: `FR-<MOD>-NNN` (e.g., `FR-WRK-001`), per `docs/_context/conventions.md`
- Epics/stories: `EPIC-xx` and `EPIC-xx.S-yy`, owned by `docs/02-Epics-and-Stories.md`
- Entities: exact canonical names from `docs/_context/conventions.md` (`WorkItem`, `TemplateVersion`, …) — never renamed
- States: exact state names from `docs/01-FSD.md` §5 (e.g., `PENDING_REVIEW`, `SUPERSEDED`)

## Where to Add New Code

There is no code to add here — implementation belongs in the future polyrepo layout specified in
`docs/_context/engineering-decisions.md` §4 (`platform-contracts`, `svc-*` repos, `mobile-app`,
`web-admin`, `infra`). What this repo accepts is **new specification and planning content**:

**New Module Specification:**
- Implementation: `docs/modules/NN-<kebab-name>/` — create the folder with exactly the fixed
  5-file set (`README.md`, `functional-spec.md`, `high-level-design.md`, `low-level-design.md`,
  `architecture-diagram.md`), matching the depth contract in `docs/_context/conventions.md`
  ("Document depth expectations per file"). Use the next sequential `NN` (currently `26`).
- Registration (required): add a row to `docs/modules/README.md`, a row to the module table in
  `docs/README.md`, and update the module summary in `README.md`.
- If it's a competitive-gap module: also record scope/entity/integration mapping in
  `docs/_context/new-modules-addendum.md`, reusing core primitives rather than new engines.

**New Shared Context (applies to 2+ modules):**
- Primary code: the relevant file in `docs/_context/` — extend `conventions.md` for
  stack/format/entity rules, `engineering-decisions.md` for binding build decisions,
  `new-modules-addendum.md` for module 14–25 scope. Never put shared material in a module folder
  (rule: `docs/_context/README.md`).

**New Epic/Stories:**
- Primary code: `docs/02-Epics-and-Stories.md` — append with the next `EPIC-xx` number and
  `EPIC-xx.S-yy` stories; keep the epic↔module mapping table in `docs/README.md` consistent.

**New Platform-Wide Architecture Decision:**
- Primary code: `docs/01-FSD.md` §2 — add `ADR-010`+ in the existing ADR format; if it changes the
  stack, it must also update `docs/_context/conventions.md` (and never contradict
  `engineering-decisions.md`).

**New GSD Command / Agent:**
- Commands: `.claude/commands/gsd/<name>.md`; subagents: `.claude/agents/gsd-<role>.md`; framework
  workflow content: `.claude/get-shit-done/workflows/`. Note these are vendor-managed framework
  files (GSD v1.42.3 with update hooks in `.claude/hooks/gsd-check-update.js`) — prefer project
  content in `docs/`/`.planning/` over modifying framework internals.

**New BMAD Skill:**
- Implementation: create `bmad-<topic>/SKILL.md` under `.claude/skills/` AND mirror it under
  `.zcode/skills/` — the two trees are kept identical (verified by `diff`).

**GSD Planning Artifacts:**
- Plans, milestones, and future codebase maps: `.planning/` (e.g., `.planning/codebase/`,
  future `.planning/phases/`-style dirs created by `/gsd:*` commands).

**Utilities:**
- Shared helpers: none exist; the only scripts are BMAD framework tools in `_bmad/scripts/`. Do not
  add project scripts without a home — `.planning/` or a new top-level `scripts/` would need to be
  established deliberately.

## Special Directories

**`.git/`:**
- Purpose: Version control metadata. Note: the workspace harness reports this as not-a-git-repo,
  but the `.git` directory is present; do not run git mutations from mapper subagents (orchestrator owns git).
- Generated: No. Committed: No (metadata).

**`.claude/get-shit-done/`:**
- Purpose: Vendored GSD framework (v1.42.3) — commands' shared engine, templates (including the
  codebase-map templates this document follows in spirit), references, hooks.
- Generated: No. Committed: Yes. Self-updating via `.claude/hooks/gsd-check-update.js` +
  `.claude/get-shit-done/bin/check-latest-version.cjs`.

**`.claude/skills/` and `.zcode/skills/`:**
- Purpose: BMAD skill library, duplicated per CLI host; must remain identical.
- Generated: Installed by BMAD installer (`.claude/get-shit-done/bin/lib/installer-migrations/`
  and `_bmad/_config/files-manifest.csv` track provenance). Committed: Yes.

**`_bmad-output/` and `_bmad/render/`:**
- Purpose: Output zones for BMAD-generated artifacts and rendered skills.
- Generated: Yes (by BMAD runs). Committed: Yes (currently empty).

**`.planning/`:**
- Purpose: GSD planning workspace — codebase maps, future phase plans.
- Generated: Yes (by GSD commands/subagents). Committed: Yes — these docs are consumed by later
  `/gsd:plan-phase` and `/gsd:execute-phase` runs per `CLAUDE.md`.

**Root legacy drafts (`*_1.md`, `high-level-design.md`):**
- Purpose: Historical Module 03 drafts kept for provenance only.
- Generated: No. Committed: Yes. Treat as read-only history; canonical content is in
  `docs/modules/03-forms-workflow-builder/` (per `README.md` "Repo notes").

---

*Structure analysis: 2026-11-27*
