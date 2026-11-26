# _context — Shared Reference Material

This folder is the shared foundation every other document in the package is written against. It holds
the original product specification the whole platform is built from, the house rules every document
must follow, the already-made engineering decisions that apply everywhere, and the scope addendum for
the twelve newer modules. Read what you need here before opening any module document in isolation —
the module docs assume this material and never repeat it.

## What's in here

| File | What it is | When to read it |
|---|---|---|
| [`source-spec.md`](./source-spec.md) | The extracted text of the original version 2.0 product requirements and functional specification — the full picture of what the platform is and does: vision, personas, product principles, capabilities, work item types, forms and workflows, offline mobile behavior, the core domain entities, lifecycle state machines, API and event formats, AI governance, and the delivery roadmap. It is the single source of truth for scope, terminology, and what already exists (modules 01–13). | First, before writing or reviewing any module document — module docs extend and deepen it, they never contradict it. |
| [`conventions.md`](./conventions.md) | The shared "house rules" every document must follow: the agreed technology stack (React Native mobile app, Kotlin/Spring Boot backend services, PostgreSQL, Kafka event backbone, Temporal for workflow scheduling, and so on), the tenant-scoping rule that every piece of data carries a tenant ID, the canonical list of domain entities, API and error-message formats, the fixed lifecycle state machines, the Mermaid diagram format, how deep each of the four document types in a module folder should go, and the numbered-requirement style that keeps everything traceable. | Before writing or editing any document — every document must stay consistent with it, and no document may invent a different stack or format. |
| [`engineering-decisions.md`](./engineering-decisions.md) | The binding, platform-wide Engineering Decisions Register. It records the already-made choices a build team would otherwise have to ask about: how sign-in and tokens work (OAuth 2.1 / OpenID Connect with device-bound tokens, refresh-token rotation, a 72-hour offline grace window), which phone models and OS versions are supported, API versioning and deprecation policy, code repository layout, CI/CD pipelines, environments, test gates and coverage requirements, secrets and encryption, observability, coding standards, the Definition of Done for every user story, and the EU AI Act compliance posture for AI that affects individual employees (mandatory human approval, logging, transparency, and risk-assessment requirements). | Before any implementation or module design work. It is binding: where a module document is silent on something covered here, this register's answer is the answer. |
| [`new-modules-addendum.md`](./new-modules-addendum.md) | The scope addendum for modules 14–25: the twelve competitive-gap modules (workforce scheduling, time & attendance, asset maintenance, IoT sensors, employee engagement, inventory and procurement, safety/EHS, food safety/HACCP, gamification, store launch, app marketplace, field service) that were not part of the original baseline. It maps each new module's entities, its requirement-ID shortcode, and exactly how it plugs into the platform primitives the first thirteen modules already define. | When working on any module 14–25, or when touching a shared primitive (work items, evidence, issues, templates, scoping) that a new module reuses. |

## The rules

- **Consistency with `conventions.md` is mandatory.** Every document in the package uses the same
  tech stack, entity names, API/event formats, state machines, and diagram style defined there — and
  extends `source-spec.md` rather than contradicting it.
- **`engineering-decisions.md` is BINDING and applies platform-wide.** These decisions are settled;
  a document never needs to re-litigate them. Where a module document is silent on something the
  register covers, the register's answer is the answer.
- **Module documents assume this folder and never repeat it.** The four deep-dive docs in each
  module folder cover only what is specific to that module; cross-cutting material lives here, once.
- **New shared material belongs here, not in module folders.** If a decision, convention, or piece of
  context turns out to apply to more than one module, add it to the relevant file in this folder so
  every module inherits it automatically.
