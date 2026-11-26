# METAREPO.md — How This Meta Repo Works

frontline-ops-meta is a **control plane**: one repo that orchestrates several child repos from a
single place. This page explains the whole mechanism in plain language. The binding rules for AI
agents live in [`CLAUDE.md`](CLAUDE.md) / [`AGENTS.md`](AGENTS.md) — this page is the
explanation, those are the law.

## The problem it solves

The platform will be built as several separate git repos (mobile app, backend, web admin,
platform contracts) that are versioned and deployed **independently**. But development is
AI-driven, and an AI agent works best with **one filesystem and shared context** — it should be
able to read the backend's code while updating the contracts, in one session.

The control plane gives you both:

- **One workspace** where every child repo is checked out side by side, next to the spec package.
- **Separate git histories** — each child repo keeps its own commits, branches, and releases.
  Nothing a child repo does ever appears in this repo's history.

## What lives where

```
frontline-ops-meta/              ← this repo (the control plane)
├── docs/
│   ├── repo-catalog.yaml        ← THE REGISTRY: one entry per child repo
│   ├── repos/<repo-id>/         ← durable notes about each child repo
│   ├── initiatives/             ← notes for work spanning several repos
│   └── …                        ← the product spec package (PRD, FSD, 25 modules)
├── scripts/                     ← bun tooling: workspace setup / status / push
├── hooks/pre-commit             ← safety net (see golden rules below)
├── workspace/
│   ├── core/<role>/<repo-id>/   ← cloned child repos — each an independent git repo
│   └── external/                ← third-party repos, if ever needed
└── .planning/codebase/          ← GSD's baseline map of THIS repo (the spec package)
```

Key mental model: **`workspace/**` is disposable local working space.** It's gitignored here.
Delete it and nothing is lost — `bun run workspace:setup` recreates it from the catalog.
Everything durable is a tracked meta file (`docs/`, `scripts/`, `hooks/`).

## The repo catalog (`docs/repo-catalog.yaml`)

The catalog is the single source of truth for *which* child repos exist and *where* they live.
Today it is **empty** — entries are added as repos come into existence. One entry looks like:

```yaml
- id: frontline-ops-contracts
  path: core/contracts/frontline-ops-contracts
  remote: git@github.com:ashishkumar1000/frontline-ops-contracts.git
  role: contracts
  default_branch: main
  status: active
  docs_dir: docs/repos/frontline-ops-contracts
```

| Field | Meaning |
|---|---|
| `id` | Stable short name. Used by the tooling and by `docs/repos/<id>/` |
| `path` | Where it clones under `workspace/` (relative to the workspace root) |
| `remote` | Git URL cloned from |
| `role` | `mobile` / `backend` / `web` / `contracts` / … — drives the push order |
| `default_branch` | Branch that `workspace:setup` clones |
| `status` | `planned` / `active` / `archived` |
| `docs_dir` | Where this repo's durable notes live inside the meta repo |

## The three commands

Bun only — the repo rejects npm/yarn/pnpm by design (`bun install` enforces it).

```bash
bun run workspace:setup    # clone every catalog entry (skips repos already on disk)
bun run workspace:status   # one-pass git status across all repos, including this one
bun run workspace:push     # fetch → review → push, in the right order
```

`workspace:push` is the orchestrated one. What it actually does:

1. **Fetch** every repo and classify it: in sync / ahead / behind / diverged / dirty / no upstream.
2. **Review** — when more than one repo needs attention, an interactive triage screen lets you
   choose push / skip / push-with-upstream per repo (arrow keys, `v` for details). With
   `--no-interactive` it just pushes everything that is safe to push.
3. **Push in dependency order:** `backend` and `contracts` repos first (the producing side —
   others depend on their API shapes), everything else next, **this meta repo always last**.
4. **Skip, never force.** Diverged or dirty repos are never force-pushed; the tool prints the
   exact fix command instead.

Useful flags: `--dry-run` (show what would happen, change nothing), `--only <id>` and
`--except <id>` (limit scope, mutually exclusive).

## The three golden rules

1. **Commit where it belongs.** A change to a child repo is committed *inside* that child repo
   (`git -C workspace/core/<role>/<repo-id> …`) and pushed to its own remote — never from the
   meta root. The `hooks/pre-commit` hook in this repo rejects any commit that tries to slip
   `workspace/**` files into meta history. Meta-level changes (docs, scripts, catalog, hooks)
   are committed here in `frontline-ops-meta`.
2. **One change across two repos = two commits in two repos.** Same-session edits never mean
   "ships together" — always state explicitly which repo merges/deploys first. Work spanning
   repos gets a note in `docs/initiatives/`.
3. **Order changes by dependency.** Additive change (new field/endpoint, old behavior untouched)
   → producing side (backend/contracts) deploys first, consumers after. Breaking change → three
   steps: producer ships the new shape *alongside* the old → consumers switch → producer removes
   the old shape in a separate change.

## Registering a child repo (when one comes to exist)

1. Create the repo on GitHub (e.g. `frontline-ops-contracts`).
2. Add its entry to [`docs/repo-catalog.yaml`](docs/repo-catalog.yaml).
3. Create `docs/repos/<id>/README.md` — what it does, contracts it owns or consumes, any
   conventions that differ from platform defaults.
4. `bun run workspace:setup` — clones the new repo; existing clones are left untouched.
5. Commit the catalog + docs changes here (that part is a meta change — rule 1 above).

Expected roles, following the target stack in
[`docs/_context/conventions.md`](docs/_context/conventions.md): `mobile`, `backend`, `web`,
`contracts` (the spec's engineering-decisions register already mandates a polyrepo setup with a
dedicated platform-contracts repo — this metarepo is that setup).

## Where durable knowledge lives

- `docs/repos/<id>/` — everything an agent should know **before touching** a child repo:
  architecture, contracts owned/consumed, per-repo conventions.
- `docs/initiatives/` — cross-repo work: which repos are involved, deploy order, acceptance
  criteria, rollout status per repo.
- The spec package (`docs/00-PRD.md` → `docs/01-FSD.md` → `docs/modules/`) — the product truth
  every child repo implements.

## How AI development fits in (BMAD + GSD)

The mandatory two-layer workflow ([rules in `CLAUDE.md`](CLAUDE.md)) maps onto the metarepo like
this:

- **BMAD (planning / discovery / architecture)** runs at the **meta root**. Product-level specs
  are control-plane knowledge — they land in `docs/` (the spec package) or
  `docs/repos/<id>/`, never inside a child repo.
- **GSD (all code execution)** runs **inside the child repo** the work belongs to
  (`workspace/core/<role>/<repo-id>`). A child repo gets its own `.planning/` baseline via
  `/gsd:new-project` run inside it; its commits and its quality gates (tests, lint, typechecks)
  use that repo's own toolchain.
- This repo's `.planning/codebase/` maps the **spec package only** — it is not a baseline for
  child-repo code.
