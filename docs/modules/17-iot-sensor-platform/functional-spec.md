# IoT Sensor Platform — Functional Specification

Corresponds to **EPIC-16 (new)** — see `new-modules-addendum.md` Module Map row 17. Tier-2
competitive gap closure, promised in the platform's own PRD Capability Map (source-spec.md §5,
"IoT/Assets": Phase 2 = Yes, Phase 3 = Advanced, success signal = "Preventive intervention rate").
Reference: `source-spec.md` (Appendix B — Example Rule Specification; §5 Capability Map),
`conventions.md` (tech stack, multi-tenancy, API/event conventions, state machines),
`new-modules-addendum.md` (Module 17 row), `06-issues-capa/low-level-design.md` (Issue creation
path this module plugs into).

## 1. Purpose and Scope

### 1.1 Purpose

Sensors are the platform's automatic, un-gameable evidence source: instead of an associate manually
recording "cooler temperature = 10.2°C" (source-spec.md Appendix A's golden scenario), a physical
sensor gateway reports it continuously. The IoT Sensor Platform generalizes Appendix B's single
`TEMP-001` temperature rule into a reusable pattern — **any** sensor type, **any** condition shape
(threshold, rate-of-change, sustained-duration), evaluated continuously against a high-volume
reading stream, with a rule breach auto-creating an `Issue` through the exact same mechanism
Appendix A describes for the manually-captured temperature exception. It closes the "IoT/Assets"
capability gap and is the sensor-data foundation that Module 16 (Asset Maintenance/CMMS) uses for
predictive maintenance and that Module 21 (Food Safety & HACCP) uses for automated CCP temperature
logging.

### 1.2 In Scope

- Registration and lifecycle management of `Sensor` records and their binding to an
  `OrganizationNode` (location) and, optionally, an `Asset` (Module 16).
- Ingestion of sensor readings over MQTT (constrained/always-connected gateways) and HTTP webhook
  (retrofit/cloud-native gateways), at high volume, multi-tenant, with idempotent de-duplication.
- Authoring and versioning of `SensorAlertRule`s: threshold, rate-of-change, and sustained-duration
  conditions, generalizing the Appendix B rule-spec shape to arbitrary sensor types and fields.
- Continuous rule evaluation against the live reading stream, with debounce/hysteresis to prevent
  alert flapping.
- Auto-creation of an `Issue` on rule breach, calling into the Issues & CAPA Service's `create_issue`
  internal path exactly as the Form Platform does for a manually-submitted temperature reading
  (06-issues-capa/low-level-design.md §2.1 "internal variant").
- Sensor health monitoring: offline detection, battery-low detection, reporting-gap detection,
  surfaced as their own alert class distinct from a data-value breach.
- Historical trend/time-series query API feeding Module 16's predictive-maintenance models and
  Module 11 (Analytics) rollups.
- Attaching a sensor reading (or a rolling window of readings) as `Evidence` on an Issue or a
  WorkItem submission.

### 1.3 Out of Scope

- **Manufacturing, provisioning, or physically shipping sensor hardware.** This module assumes a
  **BYOD-style sensor gateway model**: the tenant (or an approved hardware partner) owns and deploys
  physical sensors and gateways; this module only registers, binds, ingests from, and evaluates
  rules against them. Device firmware, calibration procedures, and physical installation are outside
  this module's scope (may be documented in tenant onboarding runbooks, not here).
- Sensor-side firmware provisioning protocols beyond credential issuance (MQTT client cert / HTTP
  webhook secret) needed for this platform to authenticate an inbound connection.
- Video/image-based sensing — that is Visual AI (Module 09).
- Long-term (multi-year) analytical modeling of sensor data beyond rollups/exports — that is
  Analytics (Module 11); this module owns operational-window retention and hands off to the
  lakehouse for deep history (see NFR-IOT-005).
- Vendor/technician dispatch triggered by a predictive-maintenance signal — that is Module 16
  (`MaintenanceWorkOrder`) and Module 25 (Field Service), which consume this module's trend data but
  own their own workflows.

## 2. Personas

| Persona | Primary job | Primary product surface |
|---|---|---|
| Operations Admin | Register sensors, bind them to locations/assets, author and publish alert rules, configure debounce/reporting-gap thresholds | Web Admin — Sensor Console |
| Store/Site Manager | Receive sensor-triggered alerts (offline, battery-low, rule breach) for their location, act on resulting Issues | Mobile App + Manager Workspace |
| Maintenance Technician | View historical sensor trend on a bound Asset to plan or justify preventive maintenance | Mobile App + Module 16 Asset Detail |
| Compliance Monitor Agent | Consume sensor-derived Issues and reading history as context for repeat-failure detection (same `get_recent_failures` pattern as Appendix A) | Agent Runtime |

## 3. Functional Requirements

### 3.1 Sensor Registration and Binding

- **FR-IOT-001**: An Operations Admin can register a `Sensor` with: `sensor_type`
  (`TEMPERATURE`, `HUMIDITY`, `DOOR_OPEN`, `VIBRATION`, `GENERIC_NUMERIC`, `GENERIC_BOOLEAN`),
  `external_device_id` (vendor/gateway-assigned identifier, unique per tenant), `display_name`,
  `unit` (e.g. `CELSIUS`, `PERCENT_RH`, `BOOLEAN`, `MM_PER_S2`), an ingestion credential
  (MQTT client certificate or HTTP webhook shared secret, minted at registration), and an optional
  `expected_interval_seconds` (defaults to 300 seconds for continuous-signal types, `NULL`/disabled
  for event-driven boolean types — see §8 for the full default-value rationale).
- **FR-IOT-002**: A `Sensor` must be bound to exactly one `OrganizationNode` (location) at a time via
  a `SensorDeviceBinding`, and may optionally also be bound to one `Asset` (Module 16) for
  predictive-maintenance correlation. A sensor with no active binding cannot have its readings
  evaluated against location- or asset-scoped alert rules (readings are still stored, flagged
  `UNBOUND`, for later reconciliation).
- **FR-IOT-003**: Rebinding a sensor to a new location or asset closes the current
  `SensorDeviceBinding` (`valid_to = now()`) and opens a new one; historical readings retain their
  original `location_id`/`asset_id` as recorded at ingestion time (effective-dated, per platform
  convention P7/P10 — see Edge Cases §5.4).
- **FR-IOT-004**: Sensor types are extensible: `GENERIC_NUMERIC`/`GENERIC_BOOLEAN` allow a tenant to
  onboard a sensor class not in the built-in list without a platform code change, by declaring a
  `value_schema` (numeric range + unit, or boolean) at registration.
- **FR-IOT-005**: An Operations Admin can deactivate (`DECOMMISSIONED`) a sensor; deactivated sensors
  stop being evaluated but keep their historical readings for audit/trend purposes.

### 3.2 Ingestion

- **FR-IOT-010**: The platform accepts sensor readings via an **MQTT** topic per tenant
  (`tenants/{tenant_id}/sensors/{sensor_id}/readings`), for gateways maintaining a persistent
  connection, authenticated by per-sensor client certificate issued at FR-IOT-001.
- **FR-IOT-011**: The platform accepts sensor readings via an **HTTP webhook**
  (`POST /api/v1/iot/sensors/{sensor_id}/readings`), signed with the sensor's shared secret (HMAC,
  same signing convention as platform outbound webhooks per source-spec §15.1), for gateways that
  push in batches or cannot hold an MQTT session.
- **FR-IOT-012**: Both ingestion paths accept a single reading or a **batch** of readings (for
  gateways that buffer during connectivity gaps and replay on reconnect).
- **FR-IOT-013**: Every ingested reading is de-duplicated on `(tenant_id, sensor_id,
  device_reading_id)` where `device_reading_id` is a gateway-supplied idempotency token (falling
  back to `(sensor_id, device_reported_at, value)` composite dedupe when a gateway does not supply
  one — see Edge Cases §5.2).
- **FR-IOT-014**: Ingestion records both `device_reported_at` (device clock, untrusted) and
  `server_received_at` (authoritative, per platform convention "client clock cannot be trusted for
  authoritative ordering" — source-spec §19.4). Rule evaluation and trend charts default to
  `server_received_at` ordering; `device_reported_at` is retained for forensic/clock-skew analysis
  (Edge Cases §5.3).

### 3.3 Alert Rule Authoring

- **FR-IOT-020**: An Operations Admin authors a `SensorAlertRule` generalizing Appendix B's shape:
  `rule_id`, `sensor_type` or explicit `sensor_id`/`sensor_tag` scope, `trigger` (always
  `sensor_reading_ingested` for this module), `condition`, and `actions`. Example (threshold,
  directly generalizing `TEMP-001`):
  ```yaml
  rule_id: TEMP-001
  scope: { sensor_type: TEMPERATURE, location_scope: "all" }
  trigger: sensor_reading_ingested
  condition:
    type: THRESHOLD
    field: value
    operator: ">"
    value: 8
  actions:
    - set_risk: HIGH
    - require_field: corrective_action
    - create_issue: true
    - notify_role: store_manager
  ```
- **FR-IOT-021**: Three condition types are supported, matching the FR set below:
  1. **Threshold** — `value {operator} {threshold}` evaluated on the latest reading (Appendix B's
     original pattern).
  2. **Rate-of-change** — `abs(value[t] - value[t - window]) / window {operator} {threshold}`, e.g.
     "temperature rose more than 3°C in 10 minutes" (compressor failure signature).
  3. **Sustained-duration** — the threshold condition must hold **continuously** for at least
     `duration` before the rule fires, e.g. "temperature > 8°C for 15 consecutive minutes" — this is
     the debounce-by-design condition type (see §3.5 and LLD §4 pseudocode) and is the recommended
     replacement for a bare threshold on any noisy sensor type.
- **FR-IOT-022**: A rule's `scope` may target a `sensor_type` (applies to all sensors of that type
  tenant-wide), a specific `sensor_id`, or a `sensor_tag` (admin-defined grouping, e.g. "walk-in
  coolers"); more specific scope overrides less specific when multiple rules could match the same
  reading (see LLD §4.1 precedence rule).
- **FR-IOT-023**: `SensorAlertRule`s are versioned like `TemplateVersion` (`DRAFT -> ACTIVE ->
  SUPERSEDED -> RETIRED`); a rule change does not retroactively re-evaluate past readings, and an
  in-flight sustained-duration evaluation window continues under the rule version active when the
  window opened (avoids a rule edit silently invalidating an in-progress breach detection).
- **FR-IOT-024**: A rule's `actions` reuse the Appendix B action vocabulary: `set_risk`,
  `require_field`, `create_issue`, `notify_role`, extended with `create_predictive_maintenance_flag`
  (Module 16 hand-off) and `attach_as_evidence` (attach the triggering reading/window to the
  resulting Issue automatically).

### 3.4 Rule Breach → Auto Issue Creation

- **FR-IOT-030**: When a rule with `create_issue: true` breaches, this module calls the Issues & CAPA
  Service's internal creation path (`POST /internal/issues`, `06-issues-capa` §2.1) with
  `source = RULE_TRIGGERED`, `source_ref = {rule_id, sensor_id, reading_id or window_id}`,
  `severity` seeded from `set_risk`, and `initial_finding` populated from the triggering reading —
  functionally identical to the Form Platform's temperature-exception path in Appendix A, so a
  sensor-triggered issue flows through the same Issue lifecycle, SLA clock, Compliance Monitor Agent
  repeat-failure detection, and CAPA steps as a manually-submitted one.
- **FR-IOT-031**: Idempotent issue creation on breach: the same `(tenant_id, rule_id,
  sensor_id, breach_window_id)` tuple must not create a second Issue on evaluator retry/replay,
  reusing the unique-partial-index pattern already defined for `RULE_TRIGGERED` issues
  (06-issues-capa LLD §1.1).
- **FR-IOT-032**: `require_field: corrective_action` on a sensor-triggered issue surfaces the same
  mandatory-corrective-action UI gate on the resulting Issue as a form-submitted breach; there is no
  parallel/duplicate mandatory-field mechanism for sensor-originated issues.
- **FR-IOT-033**: `attach_as_evidence` creates an `Evidence` record whose payload is a snapshot of
  the triggering reading (or the full sustained-duration window's readings, downsampled if large)
  and links it to the created Issue, satisfying the "sensor reading as evidence" dependency.

### 3.5 Sensor Health Monitoring

- **FR-IOT-040**: A sensor's `health_status` state machine is `ONLINE -> DEGRADED -> OFFLINE ->
  ONLINE`, with an orthogonal `BATTERY_LOW` flag (see `architecture-diagram.md` state diagram).
  `DEGRADED` = reporting slower than `expected_interval` but within `offline_threshold`; `OFFLINE` =
  no reading received within `offline_threshold` (tenant/sensor-type configurable, default: 3x the
  sensor's expected reporting interval).
- **FR-IOT-041**: An `OFFLINE` transition raises a `sensor.offline` alert (distinct alert class from
  a data-value rule breach) and, if the sensor is bound to a location/asset with an active
  temperature-class or safety-class alert rule, additionally raises a **degraded-monitoring
  notice** on that location so a manager knows the absence of an alert no longer means "in range" —
  it may mean "not reporting" (see Edge Case §5.1).
- **FR-IOT-042**: Battery-level telemetry (when the gateway reports it) below a configurable
  threshold (default 15%) sets `battery_low = true` and raises a low-priority notification to the
  Operations Admin, without affecting the sensor's `health_status` state machine on its own.
- **FR-IOT-043**: A recovered sensor (reading resumes after `OFFLINE`) transitions back to `ONLINE`
  and raises a `sensor.recovered` event; it does not retroactively evaluate rules against the gap
  (see Edge Case §5.1) unless the gateway replays buffered readings from the outage window, in which
  case those replayed readings are evaluated normally, in device-timestamp order, against the rule
  version active at each reading's `device_reported_at`.

### 3.6 Historical Trend / Predictive Maintenance Feed

- **FR-IOT-050**: A trend/history query API (`GET /api/v1/iot/sensors/{id}/readings`) returns
  time-bucketed (raw, or downsampled at `1m`/`1h`/`1d` resolution) readings for a sensor over a
  requested window, used by the mobile/web trend chart and by Module 16's predictive-maintenance
  scoring job.
- **FR-IOT-051**: For a sensor bound to an `Asset`, the trend API additionally exposes
  rule-breach-annotated overlays (breach windows highlighted) and a rolling anomaly-score field
  (statistical, not LLM-based, per source-spec §13.3's "use statistical models for anomaly
  detection") that Module 16 can read to decide when to propose a `MaintenanceWorkOrder`.
- **FR-IOT-052**: Readings older than the operational retention window (NFR-IOT-005) are downsampled
  and/or exported to the Analytics lakehouse rather than deleted outright, per source-spec's data
  lifecycle (`CREATE -> ACTIVE -> ARCHIVED -> RETENTION HOLD -> DELETE/DESTROY`).

## 4. UX States

| State | Surface | Behavior |
|---|---|---|
| Sensor-offline badge | Manager Workspace sensor list, Asset detail | Red "Offline since {time}" badge; tapping shows last-known-good reading and the degraded-monitoring notice text (FR-IOT-041). |
| Alert-rule-triggered banner | Manager Workspace, Mobile push | Banner names the sensor, the breached condition in plain language ("Walk-in Cooler #2: 10.2°C, above 8°C threshold for 15 min"), and deep-links to the auto-created Issue. |
| Sensor battery-low warning | Sensor Console (admin), Asset detail | Amber badge, non-blocking, batched into a daily digest rather than an immediate push (avoids alert fatigue for a non-urgent signal). |
| Ingestion-lag warning | Sensor Console (admin) | Shown when `server_received_at - device_reported_at` for a sensor's recent readings exceeds a lag threshold (gateway backlog or connectivity issue distinct from full offline) — informs the admin before the sensor crosses into `DEGRADED`/`OFFLINE`. |
| Empty state | Sensor Console | "No sensors registered" with a call-to-action to register the first sensor and a link to the gateway pairing/credential-issuance flow. |
| Loading | Trend chart | Skeleton chart while the (potentially large) downsampled query resolves; query is paginated/bucketed server-side so the UI never blocks on a raw multi-million-row fetch. |
| Permission-denied | All admin screens | Standard `403 SCOPE_DENIED` pattern — a Store Manager without sensor-admin scope at a location sees alerts but not the rule-authoring UI. |

## 5. Validation and Edge Cases

- **5.1 Sensor reporting gap vs. genuine breach ambiguity.** A gap in readings is not evidence of
  "in range" — the system must not silently treat "no data" as "compliant." The `OFFLINE` alert
  (FR-IOT-041) is deliberately a *separate, mandatory* alert class from a value breach, and any
  sustained-duration rule whose evaluation window spans an `OFFLINE` gap is held `INCONCLUSIVE`
  rather than resolved false — see LLD §4.3 pseudocode. A manager cannot "clear" an
  `INCONCLUSIVE` window without acknowledging it explicitly (distinct from acknowledging a normal
  breach).
- **5.2 Duplicate readings from a retrying gateway.** Both ingestion paths de-duplicate on
  `device_reading_id` (or the fallback composite key) per FR-IOT-013; a duplicate returns `200 OK`
  with the original reading's server identity, matching the platform's idempotent-submission
  convention (source-spec §19.4) rather than erroring or double-counting toward a rate/threshold
  calculation.
- **5.3 Clock-skew on device-reported timestamps.** `device_reported_at` is never used for
  authoritative ordering, rule evaluation windowing, or SLA-relevant timing — only
  `server_received_at` is (FR-IOT-014). A reading whose `device_reported_at` is more than a
  configurable skew tolerance (default 24h) away from `server_received_at` is flagged
  `CLOCK_SKEW_SUSPECTED` for admin visibility but still ingested and evaluated on
  `server_received_at`.
- **5.4 Sensor reassigned between locations/assets.** Per FR-IOT-003, historical readings keep the
  `location_id`/`asset_id` that was active in the sensor's `SensorDeviceBinding` at
  `server_received_at` of that specific reading (effective-dated, matching the platform's
  organization-reassignment convention in source-spec's QA acceptance table: "Location moves between
  regions without changing historical attribution for prior events"). A trend chart spanning a
  reassignment shows a visible binding-change marker rather than silently concatenating two
  physically different contexts.
- **5.5 Alert flapping / debounce.** A raw threshold condition on a noisy sensor (e.g. a door sensor
  bouncing near a boundary, or brief compressor-cycle temperature spikes) must not create an Issue
  per bounce. Mitigations: (a) the sustained-duration condition type (FR-IOT-021.3) is the
  recommended default for anything but a hard boolean event; (b) every rule (including plain
  threshold) has a mandatory `debounce_seconds` (default 60s) — a value must stay past the boundary
  for at least `debounce_seconds` before the rule transitions from `CANDIDATE` to `BREACHED`; (c) a
  rule that already has an open, unresolved Issue for the same `(rule_id, sensor_id)` does not create
  a second Issue on a re-breach — it re-attaches new evidence to the existing Issue instead (see LLD
  §4.4 debounce pseudocode).
- **5.6 Unbound sensor readings.** Readings from a sensor with no active binding are stored (not
  dropped — avoids silent data loss the admin later needs) but flagged `UNBOUND` and excluded from
  location/asset-scoped rule evaluation until bound; `sensor_type`-scoped tenant-wide rules still
  apply if the tenant explicitly configured them to (rare; default is location-scoped evaluation
  only).
- **5.7 Out-of-range / malformed values.** A reading whose value fails `value_schema` validation
  (FR-IOT-004) is rejected at ingestion (`400 VALIDATION_ERROR`) rather than stored and silently
  excluded from evaluation, so a misconfigured gateway is visible immediately rather than producing a
  quiet data gap.

## 6. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-IOT-001 | Ingestion throughput | Architecture must sustain sustained ingestion of at least 50,000 readings/second tenant-aggregate at MVP scale (aligned to source-spec §23 "millions of work items per day" scale target extended to sensor readings, which are higher-frequency than work items), horizontally scalable by adding stream-processor and hypertable chunk partitions, not a hard architectural ceiling. |
| NFR-IOT-002 | Alert latency | p95 time from a breaching reading's `server_received_at` to the resulting Issue's `issue.created` event ≤ 30 seconds for threshold/rate-of-change rules; for sustained-duration rules, latency is measured from the moment the duration condition is satisfied (not from reading arrival), same 30s p95 target. |
| NFR-IOT-003 | Ingestion availability | Ingestion endpoints (MQTT broker, HTTP webhook) target the platform's core-API 99.9%+ availability bar; a brief ingestion outage must not lose data for gateways that buffer-and-replay (FR-IOT-012). |
| NFR-IOT-004 | Sensor-health detection latency | `OFFLINE` transition detected within one `offline_threshold` window of the last expected reading (sweep job, LLD §6), independent of ingestion pipeline health for sensors that have simply stopped reporting. |
| NFR-IOT-005 | Retention | Raw readings retained at full resolution for 90 days (tenant-configurable) in the operational time-series store; downsampled (1-minute or coarser) retained for 2 years; full raw history beyond 90 days is available only via the Analytics lakehouse export, not the operational store. |
| NFR-IOT-006 | Tenant isolation | No cross-tenant reading leakage under any query, cache key, MQTT topic, or webhook path — per platform Multi-tenancy Rule; see HLD §6. |

## 7. Dependencies

- **Evidence (Module 05)** — a sensor reading or breach window can be attached as `Evidence` on an
  Issue or WorkItem submission (FR-IOT-033).
- **Issues & CAPA (Module 06)** — rule breach auto-creates an `Issue` via the same internal creation
  path and lifecycle as Appendix A's golden scenario (FR-IOT-030–032); this module never implements
  its own parallel exception-management state machine.
- **Asset & Equipment Maintenance / CMMS (Module 16)** — optional `SensorDeviceBinding` to an `Asset`;
  trend/anomaly data feeds Module 16's predictive-maintenance scoring, which may in turn create a
  `MaintenanceWorkOrder`.
- **Identity & Organization (Module 01)** — `OrganizationNode`/`UserScope` govern which admin can
  register sensors or author rules at which location, and which manager receives which alert.
- **Analytics & Control Tower (Module 11)** — consumes downsampled time-series rollups and
  breach/health events for enterprise dashboards and long-term trend analysis beyond this module's
  operational retention window.
- **Agentic Operations (Module 10)** — the Compliance Monitor Agent's `get_recent_failures` tool
  treats sensor-triggered Issues identically to any other `RULE_TRIGGERED` Issue; no new agent tool
  is required at MVP (a future `get_sensor_trend` tool is deferred to Phase 3 — see §8).
- **Integrations & Extensibility (Module 12)** — a large tenant may prefer to receive sensor readings
  through an existing IoT/MQTT vendor pipeline rather than direct gateway connection; the Integration
  Hub's webhook connector pattern is the supported bridge (out of scope to build a bespoke connector
  per vendor at MVP).

## 8. Resolved Decisions

- **The time-series store is the TimescaleDB extension on the existing Postgres fleet, not a
  dedicated time-series database, for MVP and for the foreseeable roadmap** (adopted in HLD §8 and
  §5). This keeps the platform on one operational database technology (per conventions.md's
  Postgres-first stance) and supports the hypertable partitioning/compression/continuous-aggregate
  features NFR-IOT-001/005 require without a new operational system to run, back up, and secure per
  tenant. **Firm revisit trigger** (not an open-ended "someday"): re-evaluate a dedicated stream-native
  time-series store only when sustained per-tenant or platform-wide write throughput materially and
  persistently exceeds the NFR-IOT-001 50,000 readings/second target even after adding hypertable
  space-partitions — until that concrete threshold is hit, TimescaleDB is the permanent answer, not a
  placeholder.
- **Default sensor reporting/sampling interval (`expected_interval_seconds`) is 300 seconds (5
  minutes) for continuous-signal sensor types** (`TEMPERATURE`, `HUMIDITY`, `VIBRATION`,
  `GENERIC_NUMERIC`) when an Operations Admin does not explicitly set one at registration — this is
  the platform fallback, not a requirement to always set it explicitly. Rationale: 5 minutes is
  frequent enough to catch a meaningful temperature/humidity excursion well within a typical
  sustained-duration rule window (e.g. the 15-minute example in FR-IOT-021.3) while keeping default
  ingestion volume for a large sensor fleet well within the NFR-IOT-001 throughput budget; a tenant
  with a faster-cycling process (e.g. a blast chiller) sets a shorter interval explicitly per sensor.
  Event-driven boolean sensor types (`DOOR_OPEN`, `GENERIC_BOOLEAN`) default to no periodic-reporting
  expectation (`expected_interval_seconds = NULL`, health/offline monitoring disabled by default for
  that sensor) since these sensors report on state change, not on a clock, and a fixed interval would
  produce false `OFFLINE` alerts during genuinely idle (door-closed) periods; an admin who wants
  heartbeat monitoring for a boolean sensor sets an explicit interval, at which point offline detection
  activates identically to a continuous-signal sensor.
- **Sensor gateways are BYOD/tenant-provided; this platform never ships firmware or physical
  hardware** (§1.3), permanently for this module's scope. A bundled hardware offering, if ever pursued,
  is a separate initiative outside this module.
- **MQTT QoS 1 (at-least-once) is the supported delivery guarantee for the MQTT ingestion path.**
  QoS 2 (exactly-once) is not implemented: application-layer de-duplication (FR-IOT-013) already
  provides idempotency, so QoS 2's additional broker-side handshake overhead would add latency and
  complexity for no additional correctness benefit.
- **A `get_sensor_trend`/`get_sensor_health` agent tool is deferred to Phase 3, not built at MVP.**
  Module 16 consumes trend data directly via this module's API at MVP, which is sufficient until a
  dedicated "Predictive Maintenance Agent" concept is scoped; building an agent tool ahead of that
  agent's own design would risk a mismatched interface. This aligns with the Capability Map's own
  "Advanced" (Phase 3) maturity marker for IoT/Assets.
- **One ingestion credential per sensor, not a shared gateway-level credential, ships for MVP and
  remains the default going forward unless field data shows credential-management friction.**
  Per-sensor credentials give simple, precise revocation (compromising or retiring one sensor never
  requires rotating a shared certificate touching many other sensors), which outweighs the smaller
  operational convenience of fewer certificates for a multi-sensor gateway. A gateway-level credential
  model is revisited only if real deployments demonstrate the per-sensor model's rotation/provisioning
  overhead is a material adoption blocker.
