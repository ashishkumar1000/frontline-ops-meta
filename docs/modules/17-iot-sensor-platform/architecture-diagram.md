# IoT Sensor Platform — Architecture Diagrams

Focused Mermaid diagrams for this module only. See `high-level-design.md` and
`low-level-design.md` for full narrative and schema detail.

## 1. Component / Container Diagram

```mermaid
flowchart TD
    subgraph Gateways [Sensor Gateways - BYOD]
        MQTTDEV[MQTT-capable gateway]
        HTTPDEV[HTTP/webhook gateway]
    end

    subgraph IngestionGateway [Ingestion Gateway]
        BROKER[MQTT Broker - per-tenant topic, mTLS]
        WEBHOOK[HTTP Ingestion API - HMAC-signed]
        DEDUPE[Idempotent Dedupe + Schema Validation]
    end

    STREAM[(Kafka: sensor.reading.raw)]

    subgraph RuleEngine [Rule Evaluation Engine]
        RULECACHE[Rule Definition Cache]
        EVAL[Condition Evaluators]
        DEBOUNCE[Debounce / Hysteresis State]
        HEALTH[Sensor Health Sweep]
    end

    TSDB[(Postgres + TimescaleDB - sensor_readings hypertable)]
    META[(Postgres - sensors, bindings, rules, alert_events)]

    CAPA[Issues and CAPA Service]
    EVID[Evidence Service]
    CMMS[Asset Maintenance / CMMS]
    ANALYTICS[Analytics Pipeline]
    NOTIF[Notification Hub]

    MQTTDEV --> BROKER --> DEDUPE
    HTTPDEV --> WEBHOOK --> DEDUPE
    DEDUPE --> STREAM
    DEDUPE --> TSDB
    STREAM --> EVAL
    RULECACHE --> EVAL
    META --> RULECACHE
    EVAL --> DEBOUNCE
    DEBOUNCE --> CAPA
    DEBOUNCE --> EVID
    HEALTH --> META
    HEALTH --> NOTIF
    DEBOUNCE --> NOTIF
    TSDB --> CMMS
    TSDB --> ANALYTICS
```

This module has three cooperating parts: an **Ingestion Gateway** that terminates both supported
protocols (MQTT for constrained always-connected gateways, HTTP webhook for cloud-aggregated
sources) behind one dedupe/validation/normalize step; a **Time-Series Store**
(TimescaleDB hypertable on the platform's existing Postgres fleet) that is the durable home for
readings and the source for trend queries; and a **Rule Evaluation Engine**, a Kafka-consuming
stream processor that holds per-sensor windowed state and is the only component that calls into
Issues & CAPA to create an Issue. No component writes to another module's tables directly — every
cross-module effect (`Issue`, `Evidence`, notifications) goes through that module's own API.

## 2. Sequence Diagram — Golden Path: Breach to Auto-Created Issue

```mermaid
sequenceDiagram
    participant GW as Sensor Gateway
    participant ING as Ingestion Gateway
    participant TS as TimescaleDB
    participant Eval as Rule Evaluation Engine
    participant Capa as Issues and CAPA Service
    participant Notif as Notification Hub

    GW->>ING: Publish reading (temperature = 10.2C)
    ING->>ING: Dedupe + validate
    ING->>TS: Store reading (write-through)
    ING->>Eval: Normalized reading event (via Kafka)
    Eval->>Eval: Evaluate TEMP-001 (sustained-duration > 8C for 15 min)
    Eval->>Eval: Condition held continuously past duration + debounce
    Eval->>Capa: POST /internal/issues (source=RULE_TRIGGERED)
    Capa-->>Eval: 201 { issue_id }
    Eval->>Notif: notify_role: store_manager
    Notif-->>GW: (no direct reply; manager notified out-of-band)
```

This is the sensor-driven equivalent of the platform's Appendix A golden scenario: a physically
captured value, rather than a manually submitted form field, drives the same
`create_issue`-on-breach mechanism, so everything downstream of Issue creation (SLA clock,
Compliance Monitor Agent repeat-failure detection, CAPA workflow) is identical regardless of source.

## 3. Sequence Diagram — Sensor-Offline Detection

```mermaid
sequenceDiagram
    participant Sweep as Sensor Health Sweep (cron)
    participant PG as Postgres (sensors)
    participant Notif as Notification Hub

    Sweep->>PG: Scan active sensors, compare now() - last_reading_at to offline_threshold
    alt gap exceeds threshold
        Sweep->>PG: health_status = OFFLINE, log sensor_alert_events(SENSOR_OFFLINE)
        Sweep->>Notif: degraded-monitoring notice to Store Manager
    else reading resumed
        Sweep->>PG: health_status = ONLINE, log SENSOR_RECOVERED
    end
```

Offline detection is poll-based rather than event-driven because there is no event to react to when
a sensor stops sending data — the sweep is the only mechanism that can notice an absence. Its
20–90s worst-case detection lag (one sweep cycle) is an accepted trade-off against NFR-IOT-004.

## 4. ER Diagram — Owned Entities

```mermaid
erDiagram
    SENSORS ||--o{ SENSOR_DEVICE_BINDINGS : "binding history"
    SENSORS ||--o{ SENSOR_READINGS : produces
    SENSORS ||--o{ SENSOR_ALERT_EVENTS : raises
    SENSOR_ALERT_RULES ||--o{ SENSOR_ALERT_EVENTS : fires
    SENSOR_ALERT_RULES }o--o| SENSOR_ALERT_RULES : superseded_by

    SENSORS {
        uuid id PK
        uuid tenant_id
        enum sensor_type
        enum health_status
        boolean battery_low
        enum status
    }
    SENSOR_DEVICE_BINDINGS {
        uuid id PK
        uuid sensor_id FK
        uuid location_id FK
        uuid asset_id FK
        timestamptz valid_from
        timestamptz valid_to
    }
    SENSOR_READINGS {
        uuid sensor_id PK-FK
        timestamptz server_received_at PK
        double value_numeric
        boolean value_boolean
        uuid location_id
        uuid asset_id
    }
    SENSOR_ALERT_RULES {
        uuid id PK
        text rule_id
        int version
        enum condition_type
        jsonb condition
        enum status
    }
    SENSOR_ALERT_EVENTS {
        uuid id PK
        uuid sensor_id FK
        uuid rule_id FK
        enum event_type
        uuid issue_id
        uuid evidence_id
    }
```

`SensorReading` is deliberately the only high-volume table here; every other entity is
low-cardinality configuration or event metadata, which is why only `sensor_readings` needs
hypertable partitioning while the rest live as ordinary Postgres tables.

## 5. State Diagram — Sensor Health Status

```mermaid
stateDiagram-v2
    [*] --> OFFLINE : sensor registered, no reading yet
    OFFLINE --> ONLINE : first/next reading arrives within expected_interval
    ONLINE --> DEGRADED : gap > 1.5x expected_interval, still < offline_threshold
    DEGRADED --> ONLINE : reading arrives, gap back within expected_interval
    DEGRADED --> OFFLINE : gap exceeds offline_threshold
    ONLINE --> OFFLINE : gap exceeds offline_threshold directly (missed DEGRADED window)
    OFFLINE --> ONLINE : reading resumes (sensor.recovered)

    state BATTERY_LOW <<choice>>
    ONLINE --> BATTERY_LOW : reported battery_pct < threshold
    DEGRADED --> BATTERY_LOW : reported battery_pct < threshold
    BATTERY_LOW --> ONLINE : battery replaced/recharged, next reading battery_pct recovers
    note right of BATTERY_LOW
        battery_low is an orthogonal flag on the
        sensor row, not a substitute health_status
        value - a sensor can be ONLINE and battery_low
        simultaneously; the diagram shows it as a
        parallel branch, not a mutually exclusive state.
    end note
```

`health_status` itself is a strict three-state cycle (`ONLINE -> DEGRADED -> OFFLINE -> ONLINE`)
driven entirely by reading recency; `BATTERY_LOW` is modeled as an orthogonal boolean flag rather
than a fourth mutually-exclusive state, because a sensor can simultaneously be reporting normally
(`ONLINE`) and running low on battery — collapsing the two into one enum would force an artificial
priority ordering between "not reporting" and "low battery" that the product does not need.
