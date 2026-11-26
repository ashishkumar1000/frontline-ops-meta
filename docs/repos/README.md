# docs/repos/ — Durable Notes per Child Repo

One folder per registered child repo, named after the catalog `id`
(`docs/repos/<repo-id>/`). These notes are the **durable cross-session memory**
about each child repo: the meta repo is where an agent looks before touching a
child, because the child repo itself only knows its own code.

No child repos exist yet — this folder is convention only until the first
entry is registered in [`repo-catalog.yaml`](./repo-catalog.yaml).

## What goes in each repo folder

When a child repo is registered, create `docs/repos/<repo-id>/` with:

- `README.md` — architecture summary, key decisions, cross-repo contracts this
  repo owns or consumes, conventions that differ from platform defaults
- `bmad-history/` *(optional)* — BMAD planning/implementation artifacts for
  that repo's stories, if you want them archived alongside the meta context

## Rules

- Keep notes factual and current — stale notes are worse than no notes.
- Contracts between two repos (shared API shapes, event names, deploy order)
  live here **and** are noted in `docs/initiatives/` if a piece of work spans
  both repos.
- The catalog entry (`repo-catalog.yaml`) is the registration source of truth;
  this folder is the knowledge source of truth.
