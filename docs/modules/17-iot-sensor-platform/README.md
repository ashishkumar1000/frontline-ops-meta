# Module 17 — IoT Sensor Platform

> Connects physical sensors — temperature, humidity, door, vibration — so readings stream into the
> platform automatically instead of an associate writing "cooler = 10.2°C" on a clipboard. The docs
> call it "the platform's automatic, un-gameable evidence source": alert rules run continuously over
> the reading stream, and a breach (say, a cooler above 8°C for 15 minutes) opens an Issue on its
> own, with the sensor data attached as proof. IoT stands for Internet of Things — the networked
> physical devices this module registers, watches, and evaluates.

| | |
|---|---|
| **Module code** | **IOT** (FR-IOT-###) |
| **Owning service** | IoT Sensor Platform (Ingestion Gateway + Time-Series Store + Rule Evaluation Engine) |
| **Tier** | 2 (competitive gap closure — `new-modules-addendum.md` row 17) |

## In plain terms
- An admin registers each sensor, gets it a credential, and binds it to a location — and optionally to a specific piece of equipment (an Asset from Module 16) for maintenance correlation.
- Readings arrive continuously from sensor gateways over MQTT (a lightweight messaging protocol for connected devices) or signed HTTP webhooks; duplicates from retrying gateways are silently deduplicated.
- Admins write alert rules in three shapes: a simple threshold ("above 8°C"), a rate of change ("rose 3°C in 10 minutes" — a dying-compressor signature), or a sustained duration ("above 8°C for 15 straight minutes"), the last being the recommended anti-flapping form.
- A confirmed breach auto-creates an Issue through the exact same path as a manually reported one — same lifecycle, same SLA clock, same mandatory corrective-action gate — with the reading attached as Evidence.
- The platform also watches the sensors themselves: a sensor that stops reporting goes OFFLINE and raises its own alert, because "no data" must never be mistaken for "everything is fine."
- Trend charts and downsampled history (raw/1m/1h/1d) feed predictive maintenance in Module 16 and long-term dashboards in Analytics.

## Key concepts
| Concept | What it means |
|---|---|
| `Sensor` | A registered device: type (temperature, humidity, door, vibration, or generic), unit, credential, and health status. Tenants bring their own hardware; the platform never ships sensors. |
| `SensorReading` | One timestamped value from one sensor — a high-volume time-series store (Postgres with the TimescaleDB extension), not ordinary rows. |
| `SensorDeviceBinding` | The effective-dated link from a sensor to one location and, optionally, one Asset; rebinding keeps historical readings attributed to where they were taken. |
| `SensorAlertRule` | A versioned rule (threshold / rate-of-change / sustained-duration) scoped to a sensor type, a specific sensor, or a tag group — a generalization of the original spec's `TEMP-001` example rule. |
| Rule breach → auto-Issue | A confirmed breach calls Issues & CAPA's internal create path (`source=RULE_TRIGGERED`), reusing that module's lifecycle instead of any parallel exception machinery. |
| Sensor health status | The `ONLINE → DEGRADED → OFFLINE` state machine (plus a low-battery flag), with a distinct "degraded monitoring" notice so managers know silence may mean the sensor is down. |
| Evidence from readings | A triggering reading — or a whole breach window — snapshot-attached as `Evidence` on the resulting Issue or a work submission. |

## How it connects to the rest
- **Issues & CAPA (Module 06)** — the primary consumer relationship: rule breaches auto-create Issues via the existing internal creation path and unique-index idempotency pattern; the docs state this module must never implement its own exception-management state machine.
- **Evidence (Module 05)** — readings and breach windows are attached as `Evidence` via the standard Evidence API; object bytes live in Evidence's storage, not here.
- **Asset & Equipment Maintenance (Module 16)** — a sensor can be bound to an `Asset`, and Module 16 reads this module's trend/anomaly query API (not the raw stream) to score predictive maintenance and propose work orders.
- **Food Safety & HACCP (Module 21) and Analytics (Module 11)** — this module is the sensor-data foundation Module 21 uses for automated cold-chain temperature logging; downsampled rollups and breach/health events feed Analytics dashboards beyond this module's operational retention window.
- **Identity & Organization (Module 01)** — `OrganizationNode`/`UserScope` govern who can register sensors and author rules at which location, and per-tenant MQTT topics and credentials keep tenant isolation enforced at the broker level.

## Deep-dive documents
| Doc | What's inside |
|---|---|
| [functional-spec.md](./functional-spec.md) | Numbered requirements FR-IOT-001–052: sensor registration/binding, MQTT and webhook ingestion, the three alert-rule condition types, breach-to-Issue creation, sensor health monitoring, and the trend/anomaly API. |
| [high-level-design.md](./high-level-design.md) | The three-part architecture (Ingestion Gateway, Time-Series Store, Rule Evaluation Engine), why TimescaleDB on the existing Postgres fleet was chosen over a separate time-series database, scaling, and alert-storm handling. |
| [low-level-design.md](./low-level-design.md) | Postgres/TimescaleDB schema, ingestion and admin endpoints, the golden breach-to-Issue sequence, rule-evaluation and debounce pseudocode, and the `INCONCLUSIVE` window handling across outages. |
| [architecture-diagram.md](./architecture-diagram.md) | Mermaid diagrams: component/container view, breach-to-Issue and offline-detection sequences, the owned-entities ER model, and the sensor health state machine. |
