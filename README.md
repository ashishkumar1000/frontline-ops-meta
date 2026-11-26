# Frontline Operations & Agentic Ops Platform

A complete, build-ready specification package for a **mobile-first B2B SaaS platform** that runs
frontline operations — the daily work done at stores, restaurants, warehouses, facilities, and
factory floors — from one governed system instead of a pile of disconnected tools.

> **This repo contains specifications, not source code.** The documents are written at build-ready
> depth (concrete schemas, API contracts, state machines) so engineering teams can implement
> directly from them. If you're new here, everything below links to a plain-language README first.

## The big idea, in three lines

1. Standards, safety rules, and brand promises are made **at headquarters**, but the work that
   fulfils them happens **at the edge** — on a phone, often offline, usually invisible to leadership.
2. This platform closes that gap with one loop: **standard → work → evidence → exception →
   correction → re-verification → learning**.
3. AI is built in — but always governed by deterministic policy ("policy before AI"), because
   ungoverned AI in a regulated, safety-relevant domain is a liability, not a feature.

## Where to go (this README is the index)

| If you want… | Go to | What's there |
|---|---|---|
| **The product story** | [`docs/00-PRD.md`](docs/00-PRD.md) | Master PRD v3.0: vision, users, mobile-first strategy, roadmap, metrics |
| **The technical architecture** | [`docs/01-FSD.md`](docs/01-FSD.md) | Platform-wide functional spec: architecture, domain model, offline sync, AI governance, security |
| **The engineering backlog** | [`docs/02-Epics-and-Stories.md`](docs/02-Epics-and-Stories.md) | 15 epics decomposed into 154 user stories with acceptance criteria |
| **All 25 modules, explained simply** | [`docs/modules/README.md`](docs/modules/README.md) | The module catalog — one-line description of every module, linking to each module's plain-language README |
| **The rules every doc follows** | [`docs/_context/README.md`](docs/_context/README.md) | Shared conventions, the **binding** engineering-decisions register, source baseline, modules addendum |
| **A file-by-file map of everything** | [`docs/README.md`](docs/README.md) | The detailed documentation index |

## The 25 modules at a glance

Full one-line descriptions live in the [module catalog](docs/modules/README.md). The short version:

- **The foundation (01–05):** identity & org structure · work scheduling & assignment · no-code form
  builder · the offline-first mobile app · the tamper-proof evidence vault
- **Close the loop (06–09):** issue tracking & CAPA · targeted communications & knowledge ·
  auto-assigned microlearning · AI photo compliance checks
- **See and automate (10–13):** governed AI agents · executive dashboards & predictions · ERP/HR
  integrations · security, privacy & compliance
- **People & assets (14–18):** workforce scheduling · time & attendance · equipment maintenance
  (CMMS) · IoT sensors · employee engagement
- **Domain packs (19–25):** inventory & procurement · EHS safety · food safety (HACCP) ·
  gamification · store launches · app marketplace · third-party vendor management

## Implementing from these docs? Read in this order

1. [`docs/00-PRD.md`](docs/00-PRD.md) — what we're building and why
2. [`docs/_context/engineering-decisions.md`](docs/_context/engineering-decisions.md) — **binding**
   platform-wide decisions (auth, versioning, CI/CD, testing gates, Definition of Done). Module docs
   assume it and never repeat it.
3. [`docs/01-FSD.md`](docs/01-FSD.md) — the architecture every module builds on
4. [`docs/02-Epics-and-Stories.md`](docs/02-Epics-and-Stories.md) — the backlog
5. The module you're working on — start at its `README.md`, then functional spec → HLD → LLD, with
   `architecture-diagram.md` as the visual companion. All diagrams are Mermaid and render natively
   on GitHub.

## Repo notes

- Root-level files like `functional-spec_1.md` and `high-level-design.md` are **legacy drafts of
  Module 03** kept for history — the canonical versions live in
  [`docs/modules/03-forms-workflow-builder/`](docs/modules/03-forms-workflow-builder/).
- Target tech stack (React Native, Kotlin/Spring Boot, PostgreSQL + RLS, Kafka, Temporal, OPA) is
  fixed in [`docs/_context/conventions.md`](docs/_context/conventions.md) — new docs must not invent
  alternatives.
