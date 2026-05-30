# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Utility Asset Management · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable domain event stored in an append-only event store. The current state of any asset, work order, or inspection is derived by replaying the sequence of events that have affected it. Materialised read models (projections) are maintained in denormalised PostgreSQL tables optimised for specific query patterns: map views, work order dashboards, compliance reports, and AI feature extraction.

The architecture follows Command Query Responsibility Segregation (CQRS): write operations produce events that are appended to the event store, while read operations query purpose-built projections that are rebuilt or updated asynchronously from the event stream. This naturally produces a complete, tamper-evident audit trail -- every change to every asset, work order, and inspection is permanently recorded with the user, timestamp, and full payload. For regulated utilities subject to NERC CIP, ISO 55000, and AWWA audits, this is not a feature bolted onto the side of a relational schema; it is the foundational architecture.

Event sourcing also enables powerful temporal queries ("what was the condition of transformer T-4401 on 15 March 2024?") and supports AI/ML workflows that need access to the full history of state transitions for training failure prediction models. The trade-off is increased system complexity: developers must understand event replay, projection rebuilds, and eventual consistency between the write store and read models.

**Best for:** Utilities with strict regulatory audit requirements (NERC CIP, ISO 55000), organisations that need temporal "as-of" queries for litigation or insurance claims, and AI-heavy deployments where the full asset state history is a training dataset.

**Trade-offs:**
- (+) Complete, immutable audit trail is the architecture, not an afterthought
- (+) Temporal queries ("as of date X") are trivial -- just replay events up to that date
- (+) Full state history available for AI/ML model training without ETL
- (+) Event stream can be consumed by multiple read models, analytics pipelines, and external systems
- (+) Schema evolution is easier -- new event types can be added without altering existing data
- (-) Higher system complexity: developers must understand event replay and projections
- (-) Eventual consistency between event store and read models requires careful handling
- (-) Projection rebuilds can be slow for large event volumes
- (-) Debugging requires understanding event sequences, not just current state
- (-) Storage requirements are higher (all historical events retained)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IEC 61970/61968 (CIM) | Event payloads use CIM-aligned field names; projection tables mirror CIM class structure |
| ISO 55000:2024 | Full asset lifecycle evidence is inherent -- every state transition is an immutable event |
| NERC CIP-002 through CIP-015 | Audit trail is the core architecture; BES cyber asset categorisation changes are events with full traceability |
| ISO 3166-1/2 | Jurisdiction codes in event payloads and projection tables |
| OGC / RFC 7946 | GeoJSON geometries stored as JSONB within event payloads; PostGIS projections for spatial queries |
| W3C SOSA/SSN | Sensor observations modeled as events; telemetry event stream supports threshold alerting |
| OCSF (Open Cybersecurity Schema) | Security-relevant events (access, configuration changes) follow OCSF event categories |
| OpenAPI 3.1 | Read model projections designed for direct OpenAPI schema mapping |

---

## Event Store (Core Write Model)

```sql
-- Event store: the single source of truth for all state changes
CREATE TABLE event_store (
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                     -- Aggregate root ID (asset_id, work_order_id, etc.)
    stream_type     VARCHAR(100) NOT NULL,              -- 'Asset','WorkOrder','Inspection','Sensor','ServiceRequest'
    event_type      VARCHAR(200) NOT NULL,              -- e.g. 'AssetRegistered','ConditionAssessed','WorkOrderCompleted'
    event_version   INTEGER NOT NULL,                   -- Sequence within the stream (for optimistic concurrency)
    payload         JSONB NOT NULL,                     -- Full event data
    metadata        JSONB NOT NULL DEFAULT '{}',        -- Actor, IP, correlation ID, causation ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
) PARTITION BY RANGE (created_at);

-- Indexes for event replay and querying
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_time ON event_store(created_at);
CREATE INDEX idx_event_stream_type ON event_store(stream_type);

-- GIN index on payload for ad-hoc JSONB queries
CREATE INDEX idx_event_payload ON event_store USING GIN(payload jsonb_path_ops);

-- Monthly partitions
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Snapshot table for performance (avoid replaying full history)
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,                  -- Event version at time of snapshot
    state           JSONB NOT NULL,                     -- Serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Type Catalogue

```sql
-- Reference table of all known event types and their schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                     -- JSON Schema for payload validation
    version         INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Event Types and Payloads

```sql
-- Example: AssetRegistered event
-- {
--   "event_type": "AssetRegistered",
--   "stream_type": "Asset",
--   "payload": {
--     "asset_code": "TX-4401",
--     "asset_type": "transformer",
--     "utility_type": "electric",
--     "name": "Main Street Pad-Mount Transformer",
--     "manufacturer": "ABB",
--     "model_number": "PAD-750-12470",
--     "serial_number": "ABB-2019-44012",
--     "install_date": "2019-06-15",
--     "commission_date": "2019-07-01",
--     "rated_power_kva": 750,
--     "primary_voltage_kv": 12.47,
--     "secondary_voltage_kv": 0.208,
--     "location": {
--       "type": "Point",
--       "coordinates": [-84.3880, 33.7490]
--     },
--     "organisation_id": "org-uuid",
--     "criticality": "high",
--     "replacement_cost": 45000.00
--   },
--   "metadata": {
--     "actor_id": "user-uuid",
--     "actor_email": "jdoe@utility.com",
--     "ip_address": "10.0.1.42",
--     "correlation_id": "req-uuid",
--     "source": "web_app"
--   }
-- }

-- Example: ConditionAssessed event
-- {
--   "event_type": "ConditionAssessed",
--   "stream_type": "Asset",
--   "payload": {
--     "inspection_id": "insp-uuid",
--     "inspector_id": "user-uuid",
--     "inspection_date": "2026-03-15T10:30:00Z",
--     "template_id": "tmpl-uuid",
--     "overall_score": 72.5,
--     "overall_grade": "C",
--     "responses": [
--       {"item": "oil_level", "value": "normal", "score": 90},
--       {"item": "corrosion", "value": "moderate", "score": 55, "is_defect": true, "severity": "minor"},
--       {"item": "bushing_condition", "value": "good", "score": 85}
--     ],
--     "location": {"type": "Point", "coordinates": [-84.3881, 33.7491]},
--     "photos": ["s3://bucket/insp-001.jpg", "s3://bucket/insp-002.jpg"]
--   }
-- }

-- Example: WorkOrderCreated event
-- {
--   "event_type": "WorkOrderCreated",
--   "stream_type": "WorkOrder",
--   "payload": {
--     "work_order_number": "WO-2026-00142",
--     "work_order_type": "corrective",
--     "asset_id": "asset-uuid",
--     "title": "Replace corroded bushing on TX-4401",
--     "description": "...",
--     "priority": 2,
--     "reported_by": "user-uuid",
--     "scheduled_start": "2026-04-01T08:00:00Z",
--     "estimated_hours": 6.0,
--     "estimated_cost": 3500.00,
--     "materials": [
--       {"part_number": "BUSH-12470-A", "quantity": 1}
--     ]
--   }
-- }

-- Example: SensorReadingReceived event
-- {
--   "event_type": "SensorReadingReceived",
--   "stream_type": "Sensor",
--   "payload": {
--     "sensor_id": "sensor-uuid",
--     "asset_id": "asset-uuid",
--     "observable_property": "oil_temperature",
--     "value": 78.3,
--     "unit": "celsius",
--     "quality": "good",
--     "observed_at": "2026-03-15T14:30:00Z"
--   }
-- }

-- Example: ThresholdExceeded event
-- {
--   "event_type": "ThresholdExceeded",
--   "stream_type": "Sensor",
--   "payload": {
--     "sensor_id": "sensor-uuid",
--     "asset_id": "asset-uuid",
--     "observable_property": "oil_temperature",
--     "value": 102.7,
--     "threshold": 95.0,
--     "threshold_type": "high",
--     "severity": "warning"
--   }
-- }
```

---

## Read Model Projections

These materialised views are rebuilt from the event stream. They are **not** the source of truth -- they are disposable and rebuildable.

### Asset Projection (Current State)

```sql
CREATE TABLE proj_asset (
    id                  UUID PRIMARY KEY,
    asset_code          VARCHAR(100) NOT NULL UNIQUE,
    asset_type          VARCHAR(100) NOT NULL,
    utility_type        VARCHAR(50) NOT NULL,
    parent_id           UUID REFERENCES proj_asset(id),
    organisation_id     UUID NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    serial_number       VARCHAR(100),
    manufacturer        VARCHAR(255),
    model_number        VARCHAR(100),
    install_date        DATE,
    commission_date     DATE,
    status              VARCHAR(50) NOT NULL DEFAULT 'active',
    criticality         VARCHAR(20),
    replacement_cost    NUMERIC(14,2),
    condition_score     NUMERIC(5,2),
    risk_score          NUMERIC(5,2),
    last_inspection_date TIMESTAMPTZ,
    last_inspection_score NUMERIC(5,2),
    open_work_orders    INTEGER NOT NULL DEFAULT 0,
    geom                GEOMETRY(Geometry, 4326),
    -- Equipment-specific fields stored as JSONB (avoids needing separate projection tables per type)
    equipment_detail    JSONB,
    -- Example equipment_detail for transformer:
    -- {
    --   "rated_power_kva": 750,
    --   "primary_voltage_kv": 12.47,
    --   "secondary_voltage_kv": 0.208,
    --   "cooling_type": "ONAN",
    --   "oil_volume_litres": 450
    -- }
    linear_detail       JSONB,                          -- For linear assets: material, diameter, sections
    last_event_version  INTEGER NOT NULL,               -- Track which event version this projection reflects
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_asset_geom ON proj_asset USING GIST(geom);
CREATE INDEX idx_proj_asset_type ON proj_asset(asset_type);
CREATE INDEX idx_proj_asset_status ON proj_asset(status);
CREATE INDEX idx_proj_asset_org ON proj_asset(organisation_id);
CREATE INDEX idx_proj_asset_condition ON proj_asset(condition_score);
CREATE INDEX idx_proj_asset_equipment ON proj_asset USING GIN(equipment_detail jsonb_path_ops);
```

### Work Order Projection

```sql
CREATE TABLE proj_work_order (
    id                  UUID PRIMARY KEY,
    work_order_number   VARCHAR(50) NOT NULL UNIQUE,
    parent_id           UUID REFERENCES proj_work_order(id),
    work_order_type     VARCHAR(50) NOT NULL,
    organisation_id     UUID NOT NULL,
    asset_id            UUID REFERENCES proj_asset(id),
    asset_code          VARCHAR(100),                   -- Denormalised for display
    asset_name          VARCHAR(255),                   -- Denormalised for display
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    priority            INTEGER NOT NULL,
    status              VARCHAR(50) NOT NULL,
    assigned_to_id      UUID,
    assigned_to_name    VARCHAR(255),                   -- Denormalised
    assigned_crew_id    UUID,
    assigned_crew_name  VARCHAR(255),                   -- Denormalised
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    estimated_hours     NUMERIC(8,2),
    actual_hours        NUMERIC(8,2),
    estimated_cost      NUMERIC(14,2),
    actual_cost         NUMERIC(14,2),
    failure_class       VARCHAR(100),
    failure_cause       VARCHAR(100),
    failure_remedy      VARCHAR(100),
    tasks               JSONB,                          -- Array of task objects
    materials           JSONB,                          -- Array of material usage objects
    labour              JSONB,                          -- Array of labour entries
    location_geom       GEOMETRY(Point, 4326),
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_wo_asset ON proj_work_order(asset_id);
CREATE INDEX idx_proj_wo_status ON proj_work_order(status);
CREATE INDEX idx_proj_wo_assigned ON proj_work_order(assigned_to_id);
CREATE INDEX idx_proj_wo_scheduled ON proj_work_order(scheduled_start);
CREATE INDEX idx_proj_wo_geom ON proj_work_order USING GIST(location_geom);
```

### Inspection Projection

```sql
CREATE TABLE proj_inspection (
    id                  UUID PRIMARY KEY,
    asset_id            UUID NOT NULL,
    asset_code          VARCHAR(100),
    asset_name          VARCHAR(255),
    template_name       VARCHAR(255),
    inspector_id        UUID,
    inspector_name      VARCHAR(255),
    inspection_date     TIMESTAMPTZ NOT NULL,
    overall_score       NUMERIC(5,2),
    overall_grade       VARCHAR(10),
    status              VARCHAR(50) NOT NULL,
    defect_count        INTEGER NOT NULL DEFAULT 0,
    critical_defects    INTEGER NOT NULL DEFAULT 0,
    responses           JSONB,                          -- Full response array
    photos              JSONB,                          -- Photo URLs and AI analysis results
    geom                GEOMETRY(Point, 4326),
    last_event_version  INTEGER NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_insp_asset ON proj_inspection(asset_id);
CREATE INDEX idx_proj_insp_date ON proj_inspection(inspection_date);
CREATE INDEX idx_proj_insp_score ON proj_inspection(overall_score);
```

### Sensor Telemetry Projection (Time-Series Optimised)

```sql
-- Latest readings per sensor (for dashboard display)
CREATE TABLE proj_sensor_latest (
    sensor_id           UUID PRIMARY KEY,
    asset_id            UUID NOT NULL,
    asset_code          VARCHAR(100),
    sensor_type         VARCHAR(100),
    observable_property VARCHAR(100),
    unit_of_measure     VARCHAR(50),
    latest_value        NUMERIC(16,6),
    latest_quality      VARCHAR(20),
    latest_observed_at  TIMESTAMPTZ,
    min_threshold       NUMERIC(12,4),
    max_threshold       NUMERIC(12,4),
    is_alarming         BOOLEAN NOT NULL DEFAULT false,
    geom                GEOMETRY(Point, 4326),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_sensor_asset ON proj_sensor_latest(asset_id);
CREATE INDEX idx_proj_sensor_alarm ON proj_sensor_latest(is_alarming) WHERE is_alarming = true;

-- Time-series readings (for charts and analysis) — partitioned
CREATE TABLE proj_sensor_timeseries (
    sensor_id       UUID NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL,
    value           NUMERIC(16,6),
    quality         VARCHAR(20),
    is_alert        BOOLEAN NOT NULL DEFAULT false,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (observed_at);
CREATE INDEX idx_proj_ts_sensor_time ON proj_sensor_timeseries(sensor_id, observed_at);
```

### Network Topology Projection

```sql
CREATE TABLE proj_network_node (
    id              UUID PRIMARY KEY,
    asset_id        UUID,
    node_type       VARCHAR(50) NOT NULL,               -- 'connectivity_node','junction','source','sink'
    utility_type    VARCHAR(50) NOT NULL,
    geom            GEOMETRY(Point, 4326),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_node_geom ON proj_network_node USING GIST(geom);

CREATE TABLE proj_network_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_node_id    UUID NOT NULL REFERENCES proj_network_node(id),
    to_node_id      UUID NOT NULL REFERENCES proj_network_node(id),
    asset_id        UUID,
    edge_type       VARCHAR(50),                        -- 'conductor','pipe','cable'
    utility_type    VARCHAR(50) NOT NULL,
    is_open         BOOLEAN NOT NULL DEFAULT false,     -- For switches/valves
    geom            GEOMETRY(LineString, 4326),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_edge_from ON proj_network_edge(from_node_id);
CREATE INDEX idx_proj_edge_to ON proj_network_edge(to_node_id);
CREATE INDEX idx_proj_edge_geom ON proj_network_edge USING GIST(geom);
```

### Compliance Dashboard Projection

```sql
CREATE TABLE proj_compliance_status (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework           VARCHAR(100) NOT NULL,
    standard_ref        VARCHAR(100) NOT NULL,
    organisation_id     UUID NOT NULL,
    obligation_text     TEXT,
    status              VARCHAR(50) NOT NULL,
    last_completed      TIMESTAMPTZ,
    next_due            TIMESTAMPTZ,
    days_until_due      INTEGER,
    evidence_event_ids  UUID[],                         -- Event IDs that constitute evidence
    responsible_user    VARCHAR(255),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_compliance_due ON proj_compliance_status(next_due);
CREATE INDEX idx_proj_compliance_framework ON proj_compliance_status(framework);
```

### AI Feature Store Projection

```sql
-- Pre-computed feature vectors for ML models
CREATE TABLE proj_ai_feature_store (
    asset_id            UUID PRIMARY KEY,
    asset_type          VARCHAR(100),
    age_years           NUMERIC(6,2),
    condition_score     NUMERIC(5,2),
    inspection_count    INTEGER,
    defect_count        INTEGER,
    work_order_count    INTEGER,
    failure_count       INTEGER,
    avg_inspection_interval_days INTEGER,
    last_inspection_days_ago INTEGER,
    sensor_alert_count_1yr INTEGER,
    replacement_cost    NUMERIC(14,2),
    criticality_numeric INTEGER,                       -- critical=4, high=3, medium=2, low=1
    climate_risk_score  NUMERIC(5,2),
    predicted_rul_days  INTEGER,
    failure_prob_1yr    NUMERIC(5,4),
    feature_vector      FLOAT8[],                      -- Embedding for similarity search
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_ai_type ON proj_ai_feature_store(asset_type);
CREATE INDEX idx_proj_ai_rul ON proj_ai_feature_store(predicted_rul_days);
```

---

## Supporting Tables (Non-Event-Sourced Reference Data)

```sql
-- Organisation (relatively static reference data, not event-sourced)
CREATE TABLE ref_organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES ref_organisation(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    jurisdiction_country CHAR(2),
    jurisdiction_subdivision VARCHAR(6),
    timezone        VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE ref_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES ref_organisation(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Roles and permissions
CREATE TABLE ref_role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    permissions     JSONB NOT NULL DEFAULT '[]'
);

CREATE TABLE ref_user_role (
    user_id         UUID NOT NULL REFERENCES ref_user(id),
    role_id         UUID NOT NULL REFERENCES ref_role(id),
    organisation_id UUID NOT NULL REFERENCES ref_organisation(id),
    PRIMARY KEY (user_id, role_id, organisation_id)
);

-- Asset type catalogue
CREATE TABLE ref_asset_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    cim_class       VARCHAR(100),
    expected_life_years INTEGER,
    is_linear       BOOLEAN NOT NULL DEFAULT false
);

-- Inspection templates
CREATE TABLE ref_inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    asset_type_id   UUID REFERENCES ref_asset_type(id),
    version         INTEGER NOT NULL DEFAULT 1,
    items           JSONB NOT NULL,                    -- Array of template items with weights, thresholds
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Material catalogue
CREATE TABLE ref_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    unit_of_issue   VARCHAR(50),
    unit_cost       NUMERIC(12,2)
);

-- Climate risk zones (spatial reference data)
CREATE TABLE ref_climate_risk_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    hazard_type     VARCHAR(50) NOT NULL,
    risk_level      VARCHAR(20) NOT NULL,
    geom            GEOMETRY(MultiPolygon, 4326) NOT NULL,
    data_source     VARCHAR(255),
    effective_date  DATE
);
CREATE INDEX idx_ref_climate_geom ON ref_climate_risk_zone USING GIST(geom);
```

---

## Projection Rebuild Infrastructure

```sql
-- Track projection build progress for each projection type
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_time TIMESTAMPTZ,
    last_rebuild_at TIMESTAMPTZ,
    rebuild_duration_ms BIGINT,
    event_count     BIGINT,
    status          VARCHAR(50) DEFAULT 'active',      -- 'active','rebuilding','failed'
    error_message   TEXT
);

-- Dead letter queue for events that fail processing
CREATE TABLE event_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    stream_id       UUID NOT NULL,
    event_type      VARCHAR(200) NOT NULL,
    projection_name VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_dead_letter_projection ON event_dead_letter(projection_name);
```

---

## Example Queries

### Temporal query: asset state at a specific point in time
```sql
-- Reconstruct asset state as of a specific date
SELECT payload
FROM event_store
WHERE stream_id = '<asset-uuid>'
  AND stream_type = 'Asset'
  AND created_at <= '2025-06-15T23:59:59Z'
ORDER BY event_version ASC;
-- Application code replays these events to reconstruct state at that date
```

### Query all events for audit evidence
```sql
-- Generate NERC CIP audit evidence for a BES cyber asset
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE stream_id = '<asset-uuid>'
  AND stream_type = 'Asset'
  AND event_type IN (
    'AssetRegistered','AssetModified','BESCyberAssetCategorised',
    'ConditionAssessed','WorkOrderCompleted','AssetDecommissioned'
  )
ORDER BY event_version ASC;
```

### Find assets with deteriorating condition trend
```sql
-- Using the projected feature store
SELECT asset_id, asset_type, condition_score,
       inspection_count, defect_count, predicted_rul_days
FROM proj_ai_feature_store
WHERE failure_prob_1yr > 0.3
  AND condition_score < 50
ORDER BY failure_prob_1yr DESC
LIMIT 50;
```

### Network trace using projection
```sql
WITH RECURSIVE downstream AS (
    SELECT to_node_id AS node_id, e.asset_id, 1 AS depth
    FROM proj_network_edge e
    WHERE from_node_id = '<start-node-uuid>' AND is_open = false
    UNION ALL
    SELECT e.to_node_id, e.asset_id, d.depth + 1
    FROM proj_network_edge e
    JOIN downstream d ON e.from_node_id = d.node_id
    WHERE e.is_open = false AND d.depth < 100
)
SELECT DISTINCT d.asset_id, a.asset_code, a.name, a.asset_type
FROM downstream d
JOIN proj_asset a ON a.id = d.asset_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 4 | Event store (partitioned), snapshots, type registry, dead letter |
| Projection Infrastructure | 1 | Checkpoint tracking for projection rebuilds |
| Asset Projections | 1 | Current asset state with JSONB equipment details |
| Work Order Projections | 1 | Denormalised work orders with embedded tasks/materials |
| Inspection Projections | 1 | Denormalised inspections with embedded responses |
| Sensor Projections | 2 | Latest readings + time-series (partitioned) |
| Network Projections | 2 | Nodes and edges for graph traversal |
| Compliance Projections | 1 | Dashboard-ready compliance status |
| AI Projections | 1 | Feature store for ML models |
| Reference Data | 8 | Organisations, users, roles, asset types, templates, materials, climate zones |
| **Total** | **22** | Plus partitions; projections are disposable and rebuildable |

---

## Key Design Decisions

1. **Single event store table for all aggregate types** -- Rather than separate event tables per entity, a single partitioned event_store table stores all events. The stream_type discriminator enables filtering. This simplifies cross-entity event queries (e.g., "all events related to asset X including work orders and inspections") and keeps the write path simple.

2. **JSONB payloads with JSON Schema validation** -- Event payloads are stored as JSONB, providing flexibility for schema evolution. The event_type_registry table holds JSON Schema definitions for payload validation at write time, ensuring data quality without rigid column structures. New event types can be added without DDL changes.

3. **Snapshots for performance** -- For long-lived aggregates (assets with thousands of events), periodic snapshots store the current state, allowing replay to start from the snapshot rather than from event 0. Snapshot frequency is configurable per stream type.

4. **Projection tables are denormalised and disposable** -- Read models are heavily denormalised (asset names embedded in work orders, inspector names in inspections) to eliminate joins at query time. If a projection becomes corrupted or its schema changes, it can be dropped and rebuilt from the event store.

5. **Equipment details in JSONB, not separate tables** -- Unlike the normalised model (Suggestion 1), equipment-specific fields (transformer ratings, pipe materials) are stored as JSONB in the proj_asset.equipment_detail column. This avoids a proliferation of projection tables and makes the read model simpler, while the event store retains full-fidelity typed data.

6. **Network topology as a projection** -- The network graph (nodes and edges) is a projection rebuilt from asset registration and topology events. This means network tracing uses standard recursive SQL on the projection tables, while the authoritative data lives in the event store.

7. **AI feature store as a first-class projection** -- Pre-computed feature vectors for ML models are maintained as a projection, updated as new events arrive. This eliminates the need for a separate ETL pipeline to extract training features from the operational database.

8. **Reference data is not event-sourced** -- Relatively static data (organisations, users, asset type catalogues, inspection templates, materials) lives in conventional relational tables prefixed with `ref_`. These change infrequently and don't benefit from the overhead of event sourcing.
