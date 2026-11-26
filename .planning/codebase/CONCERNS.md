# Codebase Concerns

**Analysis Date:** 2026-11-27

**Repo type:** Specification-only repository — no application source code. All "bugs" below are
documentation defects (inconsistent facts between documents); all "test gaps" refer to the absence
of any automated doc-consistency enforcement. Content docs live under `docs/` (135 markdown files,
~48,600 lines); AI tooling lives under `.claude/`, `.zcode/`, `_bmad/` (849 files, ~8.7 MB).

## Tech Debt

**Legacy root-level draft files duplicating Module 03 (confirmed drift):**
- Issue: Four draft files at repo root partially duplicate the canonical Module 03 docs. Verified
  by diff: `high-level-design.md` and `architecture-diagram_1.md` are byte-identical to their
  canonical counterparts, but `functional-spec_1.md` (841 lines) and `low-level-design_1.md`
  (1247 lines) are **stale** — they still carry the superseded "Decision needed:" / "Open
  Questions" text (e.g., `functional-spec_1.md` §11 "Open Questions / Assumptions", the
  label/help-text versioning decision at line ~390, and the `422`-vs-`201` Tier-2 submission
  semantics at `low-level-design_1.md:672`) that has since been rewritten as final "Resolved
  Decisions" in `docs/modules/03-forms-workflow-builder/functional-spec.md:608` and
  `docs/modules/03-forms-workflow-builder/low-level-design.md:195,672,1230`.
- Files: `functional-spec_1.md`, `high-level-design.md`, `low-level-design_1.md`,
  `architecture-diagram_1.md` (repo root)
- Impact: A reader or AI agent landing on the root drafts gets **contradictory answers** to
  questions the canonical docs have already settled (three concrete reversals identified in the
  diff: label-edit versioning, Tier-2 failure HTTP semantics, simulated-execution PII sampling).
  The identical-but-duplicated pair invites silent future divergence.
- Mitigation in place: `README.md:60-62` explicitly labels them "legacy drafts of Module 03 kept
  for history" and names the canonical location.
- Fix approach: Delete the four root drafts (history lives in git) or move them to a clearly
  named `archive/` folder excluded from doc indexes; at minimum add a deprecation banner at the
  top of each pointing to `docs/modules/03-forms-workflow-builder/`.

**Stale module-count claims in top-level docs:**
- Issue: `docs/README.md:6` still says the package covers "the 13 product modules" and
  `docs/00-PRD.md:8,376,614` repeatedly says "13 modules", while the package actually contains
  25 module folders (`docs/modules/01-identity-organization/` …
  `docs/modules/25-field-service-vendor-scheduling/`) and both files list all 25 further down.
- Files: `docs/README.md`, `docs/00-PRD.md`
- Impact: Contradicts the module catalog (`docs/modules/README.md`) and root `README.md:27,31`;
  an agent scoping work from the PRD intro will undercount scope by 12 modules.
- Fix approach: Update the intro paragraphs to "25 modules (13 baseline + 12 competitive-gap)".

**CLAUDE.md references file paths that do not exist:**
- Issue: `CLAUDE.md:18,24,30` routes BMAD output and GSD context to `docs/PRD.md`,
  `docs/architecture.md`, and `docs/epics/` — none of which exist. The actual artifacts are
  `docs/00-PRD.md`, `docs/01-FSD.md`, and `docs/02-Epics-and-Stories.md`.
- Files: `CLAUDE.md` (mirrored in `AGENTS.md`)
- Impact: The mandatory workflow-routing rules point agents at nonexistent sources of truth;
  BMAD personas instructed to write `docs/PRD.md` would create a **competing PRD** alongside
  `docs/00-PRD.md`.
- Fix approach: Rewire the paths in `CLAUDE.md`/`AGENTS.md` to the actual `docs/00-*` names.

**Dead tooling config in agent definition:**
- Issue: `.claude/agents/gsd-codebase-mapper.md:6-11` contains a commented-out PostToolUse hook
  running `npx eslint --fix $FILE`. There is no eslint config and no JS source in the repo, so
  the hook would be a silent no-op if uncommented.
- Files: `.claude/agents/gsd-codebase-mapper.md`
- Impact: Cosmetic; misleading configuration surface.
- Fix approach: Remove the commented block.

**Hardcoded machine-specific Node path in committed hook config:**
- Issue: `.claude/settings.json` invokes hooks via the absolute path
  `/Users/admin/.nvm/versions/node/v25.9.0/bin/node` in 8 hook commands (SessionStart,
  PostToolUse, PreToolUse entries).
- Files: `.claude/settings.json`
- Impact: The hook chain (prompt guard, read guard, workflow guard, context monitor, injection
  scanner) silently fails on any machine without that exact nvm node version — the repo's main
  guardrails are machine-fragile.
- Fix approach: Use `node` from PATH or `"$CLAUDE_PROJECT_DIR"`-relative resolution.

## Known Bugs

**Event name mismatch between Module 09 (producer) and Module 10 (consumer), plus registry gap:**
- Symptoms: `docs/modules/10-agentic-operations/high-level-design.md:123-126` says the Agent
  Runtime consumes an internal `visualai.result_scored` event. Module 09 — the actual producer —
  defines and emits `visualai.result_ready`, `visualai.review_completed`, and
  `visualai.appeal_submitted` (`docs/modules/09-visual-ai/high-level-design.md:104-111`,
  `docs/modules/09-visual-ai/architecture-diagram.md:20-21`); the name `visualai.result_scored`
  appears nowhere in `docs/modules/09-visual-ai/`. Additionally, none of the `visualai.*` events
  are in the canonical event catalogs — `docs/_context/source-spec.md` §18.3 (lines 598-615) or
  `docs/01-FSD.md` §6.3 "Canonical Event Catalog" (lines 1284-1311, confirmed by grep: zero
  `visualai` matches in `docs/01-FSD.md`).
- Files: `docs/modules/10-agentic-operations/high-level-design.md`,
  `docs/modules/09-visual-ai/high-level-design.md`, `docs/01-FSD.md`,
  `docs/_context/source-spec.md`
- Trigger: Any implementer wiring the Agent Runtime's Visual AI trigger, or generating
  event-schema code from §6.3.
- Workaround: Module 10's HLD itself flags it ("not in the platform's canonical list in §18.3 …
  flagged for addition to the canonical event registry as part of EPIC-11/EPIC-10 joint
  delivery"), but the name still disagrees with the producer's own contract.
- Fix: Reconcile the name to `visualai.result_ready` (or add a documented alias) and add the
  three `visualai.*` events to `docs/01-FSD.md` §6.3 and `docs/_context/source-spec.md` §18.3.

**"Module 12" means two different modules in different docs:**
- Symptoms: `docs/modules/07-communications-knowledge/functional-spec.md:61` and `:315` label
  **Agentic Operations** as "Module 12 / EPIC-11", but Agentic Operations is folder
  `docs/modules/10-agentic-operations/`. Everywhere else "Module 12" correctly means Integrations
  & Extensibility (`docs/modules/12-integrations-extensibility/README.md:1` — "# Module 12 —
  Integrations & Extensibility"; used consistently in `docs/modules/13-security-compliance/functional-spec.md:74`,
  `docs/modules/15-time-attendance/functional-spec.md:42`, `docs/_context/new-modules-addendum.md:62`).
- Files: `docs/modules/07-communications-knowledge/functional-spec.md`
- Impact: An agent resolving "Module 12" from module 07's dependency table lands on the wrong
  module's folder.
- Fix: Change the two occurrences to "Agentic Operations (Module 10 / EPIC-11)".

**"Module 14" means two different modules in different docs:**
- Symptoms: `docs/modules/07-communications-knowledge/functional-spec.md:69` and `:318` label
  **Analytics & Control Tower** as "Module 14", but Analytics & Control Tower is folder
  `docs/modules/11-analytics-control-tower/`; "Module 14" everywhere else means Workforce
  Scheduling (`docs/modules/14-workforce-scheduling/`, `docs/01-FSD.md:548`).
- Files: `docs/modules/07-communications-knowledge/functional-spec.md`
- Impact: Same wrong-target resolution risk as above; module 07 is the only outlier (likely
  written against a pre-renumbering scheme).
- Fix: Change to "Analytics & Control Tower (Module 11 / EPIC-12, EPIC-15)".

**Phantom / colliding epic numbers in competitive-gap modules (traceability rule violated):**
- Symptoms: `docs/02-Epics-and-Stories.md` defines exactly EPIC-01 through EPIC-15 (see its
  section headers at lines 16-1291). Modules 14-25 cite epics that don't exist there — and
  **EPIC-16 is claimed by three different modules**: Asset & CMMS
  (`docs/modules/16-asset-maintenance-cmms/functional-spec.md` "EPIC-16 (Asset & Equipment
  Maintenance)"), IoT (`docs/modules/17-iot-sensor-platform/functional-spec.md` and
  `low-level-design.md:3` "EPIC-16 (new)"), and Employee Engagement
  (`docs/modules/18-employee-engagement/functional-spec.md:5` "new EPIC-16 (Engagement)").
  Module 21 self-describes as "EPIC-16-class scope"
  (`docs/modules/21-food-safety-haccp/functional-spec.md:5`), module 22 claims "EPIC-22 (new)"
  (`docs/modules/22-gamification/functional-spec.md:6`), module 25 claims "EPIC-25"
  (`docs/modules/25-field-service-vendor-scheduling/functional-spec.md`). Only
  `docs/modules/11-analytics-control-tower/README.md:7-9` documents the mapping cleanly.
- Files: `docs/02-Epics-and-Stories.md`, `docs/modules/16-*` through `docs/modules/25-*`
- Impact: Violates the binding traceability rule in `docs/_context/conventions.md` (§Traceability:
  every FR traces to `EPIC-xx` story IDs). `docs/modules/README.md:78` acknowledges "epic numbers
  do not align 1:1 with folder numbers", but the EPIC-16 triple-collision is unresolved ambiguity,
  not just non-alignment.
- Fix: Extend `docs/02-Epics-and-Stories.md` with EPIC-16+ for the gap modules, or add a canonical
  epic↔module mapping table to `docs/_context/new-modules-addendum.md` and normalize citations.

**"No ambiguity" claim conflicts with the one flagged open item:**
- Symptoms: `docs/README.md:74` states "there is no remaining build-blocking ambiguity in this
  package", yet `docs/modules/10-agentic-operations/high-level-design.md:125` flags the event
  registry addition as pending.
- Files: `docs/README.md`, `docs/modules/10-agentic-operations/high-level-design.md`
- Impact: Minor credibility gap; an agent trusting the blanket claim won't look for flagged items.
- Fix: Resolve the registry item, or qualify the claim in `docs/README.md`.

## Security Considerations

**Credential hygiene — clean (verified):**
- Risk: Secrets committed in a docs repo pushed to GitHub.
- Files: repo-wide scan for `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `.npmrc`,
  `serviceAccountKey*`, `*credential*`, `*secret*` found **no credential files**. The only name
  match is `.claude/get-shit-done/bin/lib/secrets.cjs` — GSD framework source code (existence
  noted only; contents not read per forbidden-files policy). Strict pattern scan for live key
  formats (`sk-…`, `AKIA…`, `ghp_…`, `xox…`, PEM blocks) across `docs/` and root markdown found
  no hits (earlier loose matches were substrings of "task-"/"risk-").
- Current mitigation: `.gitignore` (`.DS_Store`, `*.zip`); `.git/config` remote is plain HTTPS
  GitHub (`https://github.com/ashishkumar1000/frontline-ops-meta.git`) with no embedded token.
- Recommendations: Keep it that way; if AI tooling ever needs tokens, use a secret manager, not
  files in this repo.

**Spec-level security posture is strong but unenforced:**
- Risk: The docs *prescribe* DPoP-bound tokens, refresh-token rotation, RLS tenancy, OPA policy,
  secret-manager usage (`docs/_context/engineering-decisions.md` §1 "AuthN/AuthZ";
  `docs/modules/13-security-compliance/functional-spec.md:74,184,472`) — but nothing checks
  future edits for contradictions.
- Files: `docs/_context/engineering-decisions.md`, `docs/01-FSD.md` (ADR-002, §8-§9),
  `docs/modules/13-security-compliance/`
- Current mitigation: The register is declared BINDING (`docs/_context/README.md:22-25`) and
  `docs/01-FSD.md:213-230` incorporates it by reference specifically "to avoid the two documents
  drifting apart".
- Recommendations: Add a lightweight CI lint that greps for banned patterns ("TBD", invented
  stack names, event names not in §6.3) — currently zero enforcement exists.

**AI guard hooks partially protect the doc corpus:**
- `.claude/hooks/gsd-read-injection-scanner.js` scans content on Read; `.claude/hooks/gsd-prompt-guard.js`,
  `gsd-read-guard.js`, `gsd-workflow-guard.js` gate Write/Edit. These depend on the
  machine-specific Node path in `.claude/settings.json` (see Tech Debt above), so on other
  machines the corpus is unguarded.

## Performance Bottlenecks

**Context-window load for AI agents working from these docs:**
- Problem: The corpus is large and top-heavy: `docs/01-FSD.md` 1854 lines, `docs/02-Epics-and-Stories.md`
  1405, `docs/_context/source-spec.md` 1053, `docs/00-PRD.md` 703, and deep module docs up to
  `docs/modules/03-forms-workflow-builder/low-level-design.md` 1252 lines (docs/ total: 135
  files, ~48.6k lines).
- Files: `docs/01-FSD.md`, `docs/02-Epics-and-Stories.md`, `docs/_context/source-spec.md`,
  largest module LLDs
- Cause: Build-ready depth (full schemas, endpoint tables, JSON examples) per the depth mandate
  in `docs/_context/conventions.md` §Document depth expectations.
- Improvement path: Already partially mitigated by design — plain-language per-module
  `README.md` files, the `docs/README.md` index, and GSD's doc-classifier/doc-synthesizer agents
  (`.claude/agents/gsd-doc-classifier.md`, `gsd-doc-synthesizer.md`). Keep that pattern: agents
  should enter via `README.md` → module `README.md` → the one deep doc needed, never whole-corpus
  loads.

**Tooling surface dwarfs content:**
- Problem: 849 tooling files (~8.7 MB) in `.claude/` (6.5 MB, incl. 67 commands in
  `.claude/commands/gsd/`, 29 skills, 12 hooks in `.claude/hooks/`), `.zcode/skills/` (29 skills,
  2.0 MB), `_bmad/` (160 KB) versus 135 content docs.
- Files: `.claude/`, `.zcode/`, `_bmad/`
- Cause: Two full AI workflow frameworks (GSD + BMAD) installed per `CLAUDE.md:8`.
- Improvement path: Acceptable for the stated two-layer workflow, but every framework upgrade
  rewrites hundreds of files — pin versions (`.claude/get-shit-done/VERSION`,
  `.claude/gsd-install-state.json`, `.claude/gsd-file-manifest.json` exist for this) and review
  tooling diffs separately from doc diffs.

## Fragile Areas

**Cross-document numbering conventions (module numbers, epic numbers, event names):**
- Files: `docs/modules/07-communications-knowledge/functional-spec.md` (the only file with the
  stale "Module 12/14" usage), `docs/02-Epics-and-Stories.md`, `docs/01-FSD.md` §6.3,
  `docs/_context/source-spec.md` §18.3
- Why fragile: Consistency is maintained purely by convention (`docs/_context/conventions.md`,
  `docs/_context/README.md:19-30`); there is no CI, no linter, no link-checker. The observed
  defects (Module 12/14 drift, EPIC-16 collision, `visualai.result_scored`) are all single-file
  edits that drifted silently.
- Safe modification: When touching any module doc, re-grep the exact identifiers you cite
  (`Module N`, `EPIC-xx`, event names) against `docs/modules/README.md`,
  `docs/02-Epics-and-Stories.md`, and `docs/01-FSD.md` §6.3 before committing.
- Test coverage: None (nothing validates docs).

**Binding registers as single sources of truth:**
- Files: `docs/_context/conventions.md`, `docs/_context/engineering-decisions.md`,
  `docs/_context/new-modules-addendum.md`
- Why fragile: Every one of the 25 module folders defers to these ("a module document must never
  contradict it" — `docs/_context/engineering-decisions.md:3-10`). Any edit here retroactively
  re-binds 125 module docs with no consistency check.
- Safe modification: Treat changes to these three files as platform-wide decisions; update the
  root `README.md` note (`README.md:63-65`) and grep module docs for contradicted terms after any
  change.

**Event registry as manually-synced derived artifact:**
- Files: `docs/01-FSD.md` §6.3 (canonical + "Competitive-gap module events (14–25)" section at
  line ~1312, derived from module docs), `docs/_context/source-spec.md` §18.3
- Why fragile: The FSD says gap events "are derived from each module's own functional-spec/LLD" —
  a hand-maintained rollup; module 09's three `visualai.*` events show producer events can exist
  without ever reaching the catalog.
- Safe modification: When adding an event in a module doc, add it to §6.3 in the same change.

## Scaling Limits

**Human/AI review capacity vs. corpus growth:**
- Current capacity: 25 modules × 5 docs + 3 top-level docs + 4 `_context` files ≈ 48.6k lines of
  markdown, extended once already (13 → 25 modules) via `docs/_context/new-modules-addendum.md`.
- Limit: Consistency review is fully manual; each module addition multiplies cross-reference
  surface (dependency tables, event citations, epic citations). The 13→25 expansion already
  produced the stale-count and phantom-epic defects documented above.
- Scaling path: Automate the checks (grep-based CI lint for TBD markers, module-number mentions,
  event-name membership in §6.3, epic-number membership in `docs/02-Epics-and-Stories.md`, and a
  markdown link checker) before adding more modules.

## Dependencies at Risk

**GSD + BMAD tooling layer:**
- Risk: `.claude/` (GSD, version tracked in `.claude/get-shit-done/VERSION`, install state in
  `.claude/gsd-install-state.json`, 400-line manifest in `.claude/gsd-file-manifest.json`,
  migration journal `.claude/gsd-migration-journal/`) and `.zcode/skills/` + `_bmad/` (BMAD,
  config in `_bmad/config.toml`, manifests in `_bmad/_config/`) are third-party framework
  installs living inside the spec repo.
- Impact: Framework updates overwrite large file sets (849 files), can break hooks
  (`.claude/settings.json` paths), and blur review history for the actual docs.
- Migration plan: None needed if versions are pinned and tooling commits are kept separate from
  doc commits; on conflict, the GSD migration journal (`.claude/gsd-migration-journal/`) records
  prior upgrades.

**Node runtime for hooks:**
- Risk: Hooks in `.claude/settings.json` bind to `/Users/admin/.nvm/versions/node/v25.9.0/bin/node`.
- Impact: Any node upgrade or clone on another machine disables all 12 hooks in `.claude/hooks/`.
- Migration plan: Switch to PATH-resolved `node`.

## Missing Critical Features

**No CI, no tests, no linters, no enforcement of the spec's own consistency rules:**
- Problem: The repo's only quality mechanism is convention plus AI review. There is no workflow
  (no `.github/workflows/`), no test runner for docs, no markdown linter, no link checker. The
  only tests anywhere are Python tests for BMAD skill scripts
  (`.claude/skills/*/scripts/tests/`, e.g. `.claude/skills/bmad-customize/scripts/tests/`).
- Blocks: Guaranteed detection of the drift classes already observed (stale module counts,
  phantom epics, unregistered events, broken relative links across 135 files).

**`visualai.*` events missing from canonical registry:**
- Problem: Producer events `visualai.result_ready`, `visualai.review_completed`,
  `visualai.appeal_submitted` (`docs/modules/09-visual-ai/high-level-design.md:103-111`) are
  absent from `docs/01-FSD.md` §6.3 and `docs/_context/source-spec.md` §18.3; the phantom
  `visualai.result_scored` name is flagged in `docs/modules/10-agentic-operations/high-level-design.md:125`.
- Blocks: Downstream codegen or integration work that treats §6.3 as the complete event contract.

**Canonical epic coverage for modules 14-25:**
- Problem: `docs/02-Epics-and-Stories.md` stops at EPIC-15; gap modules self-assign colliding
  epic numbers (see Known Bugs).
- Blocks: Traceability from the ~500+ FRs of the 12 gap modules to backlog stories, per the rule
  in `docs/_context/conventions.md` §Traceability.

## Test Coverage Gaps

**Doc-consistency validation (entire repo):**
- What's not tested: TBD-marker compliance (the `docs/_context/conventions.md:12` ban — currently
  clean: the only "TBD" hits are the rule text itself and "JTBD" in
  `docs/modules/08-learning/functional-spec.md:257`; zero TODO/FIXME/HACK in `docs/`), module/epic
  numbering validity, event-name membership in the registry, "Decision needed:" resolution state
  (only 2 hits remain, both meta-references in `docs/_context/conventions.md:13` and
  `docs/_context/engineering-decisions.md:7`), and relative-link integrity.
- Files: `docs/` (all 135 files), checkable with simple grep scripts
- Risk: The drift already found (Module 12/14, EPIC-16, stale counts, event mismatch) recurs
  unnoticed on every doc edit.
- Priority: High — this is the repo's only real "code".

**TBD-compliance status (verified clean today):**
- The convention `docs/_context/conventions.md:12-14` forbids "TBD"/"to be decided" in favor of
  "Decision needed:" with a default. As of 2026-11-27 the corpus complies: no unresolved
  "Decision needed"/"Open Questions" sections remain in module docs
  (`docs/modules/03-forms-workflow-builder/functional-spec.md:608` and
  `docs/modules/10-agentic-operations/functional-spec.md:520` are both rewritten "Resolved
  Decisions" sections). The only outstanding flagged item is the event-registry addition above.
- Priority: Keep it that way via the CI lint proposed under Missing Critical Features.

## Repo Hygiene Notes

- Three `.DS_Store` files exist on disk: `.DS_Store`, `docs/.DS_Store`, `docs/modules/.DS_Store`.
  `.gitignore:1` ignores `.DS_Store`, so they should remain untracked; safe to delete locally.
- A `.git` directory exists (branch `main`, GitHub remote in `.git/config`); no git commands were
  run during this analysis (orchestrator handles git).
- `_bmad-output/` is an empty directory (BMAD output target per `CLAUDE.md` — which points at
  `docs/` paths instead; see Tech Debt).
- `.planning/codebase/` sibling docs (`ARCHITECTURE.md`, `CONVENTIONS.md`, `TESTING.md`) were
  written by the other mappers; this file complements them.

---

*Concerns audit: 2026-11-27*
