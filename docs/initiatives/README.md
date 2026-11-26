# docs/initiatives/ — Cross-Repo Work Notes

Durable notes for work that **spans more than one child repo** (or spans a
child repo and this meta repo). One markdown file per initiative, named after
the work (e.g. `2027-q1-mobile-bootstrap.md`).

A cross-repo change is never one atomic release: each affected repo gets its
own commits, in the order documented in `CLAUDE.md` (producing side first).
The initiative note is what survives across sessions and tells the next agent:

- which repos are involved and which merges/deploys first
- the shared acceptance criteria
- rollout status per repo (what shipped, what's pending)

Empty today — no child repos exist yet, so nothing spans repos.
