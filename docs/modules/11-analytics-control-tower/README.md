# Module 11 — Analytics & Control Tower

> This module turns the platform's raw operational activity (tasks, audits, issues, agent actions) into KPIs, dashboards, alerts, and forward-looking risk predictions. It answers the seven questions every operator has — what is wrong, where, how serious, why, what has been done, what needs me, and is it working — at three levels: Executive, Regional/District Manager, and Store/Site Manager. Its "Control Tower" screens are where leaders spot problems and drill down from a company-wide number to the specific store, work item, and evidence behind it.

| | |
|---|---|
| **Module code** | `11-analytics-control-tower` (folder identifier; docs map it to EPIC-12 Control Tower + EPIC-15 Analytics & ML) |
| **Owning service** | Analytics Platform (one logical module: streaming ingestion, lakehouse, metrics semantic layer, dashboard API, ML forecasting service) |
| **Epics** | EPIC-12 (Control Tower), EPIC-15 (Analytics & ML) |

## In plain terms
- Executives see one Enterprise Health Score (0–100) plus live counters: critical exceptions, locations at risk, repeated failure patterns, and unacknowledged critical issues.
- Regional managers get a ranked heatmap of their locations by health, an exception queue, and a territory risk forecast with the top causes named in plain language.
- Store managers see their own location's trends against region/brand benchmarks, their verification backlog, and a "what needs my attention" list that ranks items instead of just showing them chronologically.
- Anyone can drill down from an enterprise KPI dip to the exact locations, work items, issues, and photo/form evidence behind it — each hop re-checks what the viewer is allowed to see.
- Admins set KPI thresholds and who gets alerted (in-app, email, Slack/Teams) when a metric breaches them; reports can be exported (PDF/CSV/XLSX) or scheduled as recurring deliveries.
- Predictive models score each location's risk daily, forecast KPIs 7/30 days ahead, and cluster repeat failures across locations — every prediction ships with a recommended action a manager can accept (which creates a real task) or dismiss.

## Key concepts
| Concept | What it means |
|---|---|
| **VOER (Verified Operational Execution Rate)** | The North-Star KPI: the share of expected work items that were actually completed *and* verified on time. Computed in the gold layer and segmented by location, workflow, risk band, evidence type, and verification method. |
| **Enterprise Health Score** | A single 0–100 composite (weighted VOER, repeat-failure, on-time completion, critical-issue density) shown at the top of the Executive Control Tower; weights are tenant-configurable and versioned. |
| **Control Tower views** | The three persona-scoped dashboards: Executive, Regional/District Manager, and Store/Site Manager (Location Health), each with fixed required widgets. |
| **Drill-down navigation** | The fixed path from an aggregate metric down to evidence (enterprise → region → location → work item type → records → evidence); the final hop links to the owning module's record view. |
| **Lakehouse (bronze/silver/gold)** | The three-layer data store: raw as-received copies (bronze), cleaned and conformed data (silver), and business-ready pre-aggregated facts (gold). Dashboards query only gold. |
| **Metrics semantic layer** | One governed, versioned definition of every KPI, so dashboards, exports, BI tools, and the AI agent all compute the same number the same way. |
| **Location Risk Score** | A daily 0–100 score per location (LOW/MEDIUM/HIGH/CRITICAL band) with its top 2–3 drivers stated in plain language, not raw model output. |
| **Freshness tiers** | Data latency classes per KPI — near-real-time (<15 min) for critical counters, T-1 (daily batch) for most — with honest UI labels when data is partial or stale. |

## How it connects to the rest
- It consumes nearly every other module's events and database changes (via CDC — change data capture, a stream of every row change): Work Orchestration (02) and Issues/CAPA (06) feed the core KPIs, Visual AI (09) feeds risk drivers, Communications/Learning (07/08) feed adoption metrics, Identity/Org (01) provides the hierarchy snapshots used for rollups.
- It has the platform's one true bidirectional relationship with Agentic Operations (10): agent events feed the Automation Rate and AI Override Rate KPIs, while its Metrics Query API is the data source the Executive Insights Agent retrieves from.
- It produces the `analytics.threshold_breached` event, consumed by the Notification Hub (which delivers the actual alerts) and optionally by the Agent Runtime (10) as an automation trigger.
- Integration Hub (12) builds its BI connectors (Power BI/Tableau/Snowflake-class) on this module's Metrics Query API and semantic layer.
- It deliberately owns only derived data: the source services stay the system of record, and a recommended action becomes a real task/issue only when a manager explicitly accepts it.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | FR-ANLY-001–0xx requirements: exact formulas and segmentation for every KPI (starting with VOER, FR-ANLY-001), the three Control Tower views, drill-down, alerting, predictive risk, and exports. |
| [high-level-design.md](./high-level-design.md) | The five Analytics Platform responsibilities (ingestion, lakehouse, semantic layer, dashboard API, ML pipeline), the component diagram, data ownership, scaling, and the near-real-time bypass path. |
| [low-level-design.md](./low-level-design.md) | Gold-layer star schema (dimensions, facts, ML tables), metrics query REST endpoints, CDC-to-dashboard sequence diagrams, and pseudocode for VOER, anomaly detection, and pattern clustering. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component view, ingestion-to-dashboard and drill-down sequences, the gold-layer ER diagram, and the bronze→silver→gold data-quality flow with validation gates. |
