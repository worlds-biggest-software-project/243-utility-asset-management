# Data Model Suggestion 4: Graph-Relational (Property Graph + Relational Operations)

> Project: Utility Asset Management · Created: 2026-05-22

## Philosophy

Utility networks are fundamentally graphs. An electric distribution system is a tree of feeders, switches, transformers, and conductors connected at nodes. A water system is a directed graph of pipes, valves, pumps, and reservoirs. The most demanding operations in utility asset management -- network tracing, isolation analysis, impact assessment, and connectivity validation -- are graph traversal problems that relational databases handle awkwardly through recursive CTEs on junction tables.

This model uses a property graph layer (implemented as `graph_node` and `graph_edge` tables in PostgreSQL with PostGIS) as the primary representation of the network topology, while keeping operational data (work orders, inspections, inventory, compliance) in conventional relational tables. The graph layer stores network connectivity, flow direction, switch state, and spatial geometry. The relational layer stores business process data that references nodes and edges in the graph.

This is not a full Neo4j deployment. The graph is implemented in PostgreSQL using an adjacency list pattern with typed edges and JSONB properties on both nodes and edges. This keeps the entire system in one database, avoids the operational complexity of a separate graph database, and still provides dramatically better network traversal performance than the normalised model's Terminal/ConnectivityNode pattern. For organisations that later need the performance of a dedicated graph engine, the node/edge tables can be replicated to Neo4j or Apache AGE (PostgreSQL graph extension) without changing the relational side.

The key architectural insight is that utility networks have two distinct data access patterns: (1) graph traversal for network operations (tracing, isolation, impact analysis) and (2) tabular CRUD for business processes (work orders, inspections, compliance). This model optimises for both by giving each pattern its own data structure.

**Best for:** Utilities where network tracing, isolation analysis, and connectivity validation are core daily operations; electric distribution utilities with complex feeder management; multi-utility operators who need cross-utility impact analysis; organisations planning future migration to a dedicated graph database.

**Trade-offs:**
- (+) Network tracing, isolation, and impact analysis are natural graph operations, not awkward recursive CTEs
- (+) Graph structure naturally represents the physical connectivity of utility infrastructure
- (+) Cross-utility dependency analysis (electric pole supports telecom cable) is a simple edge relationship
- (+) Path-based queries (shortest route for field crew, distance to source) are efficient
- (+) Future migration to Neo4j or Apache AGE is straightforward
- (-) Developers must understand both relational and graph query patterns
- (-) Graph consistency must be maintained alongside relational consistency (dual write concern)
- (-) PostgreSQL graph traversal is still slower than native graph databases for very large networks (10M+ nodes)
- (-) Graph visualisation tools may require additional frontend investment
- (-) Less conventional architecture -- harder to hire developers with experience

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IEC 61970-301 (CIM Core) | Graph node types map to CIM equipment classes; edge types map to CIM connectivity model (Terminal/ConnectivityNode) |
| IEC 61968-11 (CIM Distribution) | Operational tables (work orders, assets, inspections) follow CIM distribution management class structure |
| ISO 55000:2024 | Asset lifecycle fields on graph nodes and relational tables; graph enables lifecycle dependency analysis |
| ISO 3166-1/2 | Jurisdiction codes on organisation and location tables |
| OGC / RFC 7946 | PostGIS geometry on graph nodes and edges; GeoJSON export for map rendering |
| NERC CIP | Graph enables rapid identification of all assets within a BES cyber system boundary via connected-component analysis |
| W3C SOSA/SSN | Sensors modeled as graph nodes connected to equipment nodes; observation data in relational time-series |
| OGC GeoSPARQL 1.1 | Graph structure is compatible with future RDF/SPARQL export for linked-data interoperability |
| OpenAPI 3.1 | Relational tables designed for direct OpenAPI schema mapping; graph traversal exposed as API operations |

---

## Graph Layer

```sql
-- ============================================================
-- GRAPH NODE: Every physical component in the utility network
-- ============================================================
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,
    -- Node types for electric:
    --   'substation','feeder_head','transformer','switch','fuse','recloser',
    --   'pole','conductor_junction','meter_point','capacitor_bank','regulator'
    -- Node types for water:
    --   'treatment_plant','pump_station','reservoir','tank','valve',
    --   'hydrant','pipe_junction','meter_point','pressure_reducer'
    -- Node types for gas:
    --   'gate_station','regulator_station','valve','pipe_junction','meter_point'
    -- Shared:
    --   'connectivity_point','boundary_point'

    utility_type    VARCHAR(50) NOT NULL CHECK (utility_type IN (
        'electric','gas','water','wastewater','stormwater','telecom','shared'
    )),
    subnetwork_id   UUID,                               -- Feeder, pressure zone, basin
    asset_id        UUID,                               -- Link to relational asset record (nullable for pure connectivity points)
    organisation_id UUID NOT NULL,
    name            VARCHAR(255),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',

    -- Spatial
    geom            GEOMETRY(Point, 4326) NOT NULL,
    elevation_m     NUMERIC(10,2),

    -- Graph-specific properties
    is_source       BOOLEAN NOT NULL DEFAULT false,     -- Feeder head, pump station, gate station
    is_boundary     BOOLEAN NOT NULL DEFAULT false,     -- Subnetwork boundary
    is_switchable   BOOLEAN NOT NULL DEFAULT false,     -- Can open/close (switch, valve)
    is_open         BOOLEAN NOT NULL DEFAULT false,     -- Current state: open = disconnected
    scada_point_id  VARCHAR(100),                       -- SCADA integration point

    -- Equipment properties (JSONB for type-specific attributes)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for transformer node:
    -- {
    --   "rated_power_kva": 750,
    --   "primary_voltage_kv": 12.47,
    --   "secondary_voltage_kv": 0.208,
    --   "phase_count": 3,
    --   "cooling_type": "ONAN"
    -- }
    -- Example for valve node:
    -- {
    --   "valve_type": "gate",
    --   "nominal_diameter_mm": 300,
    --   "turns_to_close": 24,
    --   "last_exercise_date": "2026-01-15"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_geom ON graph_node USING GIST(geom);
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_utility ON graph_node(utility_type);
CREATE INDEX idx_gn_subnetwork ON graph_node(subnetwork_id);
CREATE INDEX idx_gn_asset ON graph_node(asset_id);
CREATE INDEX idx_gn_org ON graph_node(organisation_id);
CREATE INDEX idx_gn_status ON graph_node(status);
CREATE INDEX idx_gn_switchable ON graph_node(is_switchable) WHERE is_switchable = true;
CREATE INDEX idx_gn_source ON graph_node(is_source) WHERE is_source = true;
CREATE INDEX idx_gn_properties ON graph_node USING GIN(properties jsonb_path_ops);

-- ============================================================
-- GRAPH EDGE: Physical connection between two nodes
-- ============================================================
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_node_id    UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    to_node_id      UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types for electric:
    --   'overhead_conductor','underground_cable','busbar','jumper','service_drop'
    -- Edge types for water:
    --   'main','service_line','force_main','gravity_main'
    -- Edge types for gas:
    --   'transmission_main','distribution_main','service_line'
    -- Structural:
    --   'contains','supports','feeds','monitors'

    utility_type    VARCHAR(50) NOT NULL,
    asset_id        UUID,                               -- Link to relational asset record
    is_directed     BOOLEAN NOT NULL DEFAULT true,      -- Flow direction matters
    is_active       BOOLEAN NOT NULL DEFAULT true,      -- Can be deactivated for planned removals
    weight          NUMERIC(12,4) DEFAULT 1.0,          -- For shortest path: length, impedance, or cost

    -- Spatial (for linear assets)
    geom            GEOMETRY(LineString, 4326),
    length_m        NUMERIC(12,2),

    -- Edge-specific properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for conductor edge:
    -- {
    --   "material": "ACSR",
    --   "cross_section_mm2": 120,
    --   "voltage_class_kv": 12.47,
    --   "ampacity_a": 350,
    --   "is_underground": false,
    --   "span_length_m": 75
    -- }
    -- Example for pipe edge:
    -- {
    --   "material": "ductile_iron",
    --   "nominal_diameter_mm": 300,
    --   "pressure_class": "C30",
    --   "lining_type": "cement_mortar",
    --   "install_date": "1985-06-01"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Prevent self-loops
    CHECK (from_node_id != to_node_id)
);

CREATE INDEX idx_ge_from ON graph_edge(from_node_id);
CREATE INDEX idx_ge_to ON graph_edge(to_node_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_utility ON graph_edge(utility_type);
CREATE INDEX idx_ge_asset ON graph_edge(asset_id);
CREATE INDEX idx_ge_geom ON graph_edge USING GIST(geom);
CREATE INDEX idx_ge_active ON graph_edge(is_active) WHERE is_active = true;
CREATE INDEX idx_ge_properties ON graph_edge USING GIN(properties jsonb_path_ops);

-- Composite index for graph traversal (most common query pattern)
CREATE INDEX idx_ge_from_active ON graph_edge(from_node_id) WHERE is_active = true;
CREATE INDEX idx_ge_to_active ON graph_edge(to_node_id) WHERE is_active = true;

-- ============================================================
-- SUBNETWORK: Named partition of the graph (feeder, zone, basin)
-- ============================================================
CREATE TABLE subnetwork (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL,
    subnetwork_type VARCHAR(50),                         -- 'feeder','pressure_zone','collection_basin','circuit'
    source_node_id  UUID REFERENCES graph_node(id),      -- Head of the subnetwork
    organisation_id UUID NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "nominal_voltage_kv": 12.47,
    --   "total_customers": 2400,
    --   "total_length_km": 45.2,
    --   "peak_load_mw": 15.6
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CROSS-UTILITY DEPENDENCIES
-- ============================================================
-- Model dependencies between different utility networks
-- e.g., "electric pole supports telecom cable" or "pump station depends on electric feeder"
CREATE TABLE cross_utility_dependency (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_node_id UUID NOT NULL REFERENCES graph_node(id),
    dependent_node_id UUID NOT NULL REFERENCES graph_node(id),
    dependency_type VARCHAR(50) NOT NULL,                -- 'powers','supports','feeds','protects'
    criticality     VARCHAR(20),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (provider_node_id != dependent_node_id)
);
CREATE INDEX idx_cud_provider ON cross_utility_dependency(provider_node_id);
CREATE INDEX idx_cud_dependent ON cross_utility_dependency(dependent_node_id);
```

## Relational Operations Layer

### Asset Register (linked to graph nodes)

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    country_code    CHAR(2),
    subdivision_code VARCHAR(6),
    timezone        VARCHAR(50),
    settings        JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_user_org ON app_user(organisation_id);

-- Asset record (operational data that supplements the graph node)
CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id   UUID REFERENCES graph_node(id),      -- For point assets
    graph_edge_id   UUID REFERENCES graph_edge(id),      -- For linear assets (pipes, conductors)
    asset_code      VARCHAR(100) NOT NULL,
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    asset_category  VARCHAR(100) NOT NULL,               -- 'transformer','pipe','valve','conductor','hydrant'
    utility_type    VARCHAR(50) NOT NULL,
    cim_class       VARCHAR(100),

    -- Lifecycle
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    install_date    DATE,
    commission_date DATE,
    expected_disposal DATE,
    serial_number   VARCHAR(100),
    manufacturer    VARCHAR(255),
    model_number    VARCHAR(100),

    -- Financial
    replacement_cost NUMERIC(14,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    criticality     VARCHAR(20),

    -- Condition
    condition_score NUMERIC(5,2),
    risk_score      NUMERIC(5,2),
    last_inspection_date TIMESTAMPTZ,

    -- Extended properties
    properties      JSONB NOT NULL DEFAULT '{}',
    compliance      JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, asset_code)
);
CREATE INDEX idx_asset_node ON asset(graph_node_id);
CREATE INDEX idx_asset_edge ON asset(graph_edge_id);
CREATE INDEX idx_asset_org ON asset(organisation_id);
CREATE INDEX idx_asset_category ON asset(asset_category);
CREATE INDEX idx_asset_status ON asset(status);
CREATE INDEX idx_asset_condition ON asset(condition_score);
CREATE INDEX idx_asset_properties ON asset USING GIN(properties jsonb_path_ops);
```

### Work Order Management

```sql
CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_number VARCHAR(50) NOT NULL,
    parent_id       UUID REFERENCES work_order(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    asset_id        UUID REFERENCES asset(id),
    graph_node_id   UUID REFERENCES graph_node(id),      -- Direct graph reference for map operations
    work_order_type VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    priority        INTEGER NOT NULL DEFAULT 3,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    reported_by     UUID REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    estimated_cost  NUMERIC(14,2),
    actual_cost     NUMERIC(14,2),
    tasks           JSONB NOT NULL DEFAULT '[]',
    labour          JSONB NOT NULL DEFAULT '[]',
    materials       JSONB NOT NULL DEFAULT '[]',
    failure_analysis JSONB,
    -- Graph context: affected nodes/edges for impact assessment
    affected_nodes  UUID[],                              -- Array of graph_node IDs affected
    affected_edges  UUID[],                              -- Array of graph_edge IDs affected
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, work_order_number)
);
CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_node ON work_order(graph_node_id);
CREATE INDEX idx_wo_org ON work_order(organisation_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_scheduled ON work_order(scheduled_start);
CREATE INDEX idx_wo_affected_nodes ON work_order USING GIN(affected_nodes);

-- Preventive maintenance schedule
CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    asset_category  VARCHAR(100),
    asset_id        UUID REFERENCES asset(id),
    trigger_config  JSONB NOT NULL,
    work_order_template JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pm_org ON pm_schedule(organisation_id);
CREATE INDEX idx_pm_next ON pm_schedule(next_due);
```

### Inspection & Condition Assessment

```sql
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    work_order_id   UUID REFERENCES work_order(id),
    inspector_id    UUID NOT NULL REFERENCES app_user(id),
    inspection_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    template_name   VARCHAR(255),
    overall_score   NUMERIC(5,2),
    overall_grade   VARCHAR(10),
    defect_count    INTEGER NOT NULL DEFAULT 0,
    critical_defects INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    geom            GEOMETRY(Point, 4326),
    responses       JSONB NOT NULL DEFAULT '[]',
    attachments     JSONB NOT NULL DEFAULT '[]',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insp_asset ON inspection(asset_id);
CREATE INDEX idx_insp_node ON inspection(graph_node_id);
CREATE INDEX idx_insp_date ON inspection(inspection_date);
CREATE INDEX idx_insp_score ON inspection(overall_score);
CREATE INDEX idx_insp_geom ON inspection USING GIST(geom);
```

### Sensors & Telemetry

```sql
CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    graph_node_id   UUID REFERENCES graph_node(id),      -- Sensor location in the network graph
    sensor_type     VARCHAR(100) NOT NULL,
    observable_property VARCHAR(100) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_reading_at TIMESTAMPTZ,
    last_value      NUMERIC(16,6),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_asset ON sensor(asset_id);
CREATE INDEX idx_sensor_node ON sensor(graph_node_id);

-- Time-series observations (partitioned)
CREATE TABLE sensor_observation (
    sensor_id       UUID NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL,
    value           NUMERIC(16,6),
    quality         VARCHAR(20) DEFAULT 'good'
) PARTITION BY RANGE (observed_at);
CREATE INDEX idx_obs_sensor_time ON sensor_observation(sensor_id, observed_at);

-- Sensor alerts
CREATE TABLE sensor_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    threshold_value NUMERIC(12,4),
    actual_value    NUMERIC(12,4),
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_node ON sensor_alert(graph_node_id);
CREATE INDEX idx_alert_unack ON sensor_alert(is_acknowledged) WHERE is_acknowledged = false;
```

### Inventory, Compliance, Documents

```sql
CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    graph_node_id   UUID REFERENCES graph_node(id),      -- Location in network
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    unit_of_issue   VARCHAR(50) NOT NULL,
    unit_cost       NUMERIC(12,2),
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inventory_balance (
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    material_id     UUID NOT NULL REFERENCES material(id),
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 0,
    reorder_point   NUMERIC(10,2),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (storeroom_id, material_id)
);

CREATE TABLE compliance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    framework       VARCHAR(100) NOT NULL,
    standard_ref    VARCHAR(100) NOT NULL,
    obligation_text TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    responsible_user UUID REFERENCES app_user(id),
    last_completed  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    status          VARCHAR(50) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compliance_due ON compliance_obligation(next_due);

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    title           VARCHAR(500),
    doc_type        VARCHAR(50) NOT NULL,
    file_name       VARCHAR(255) NOT NULL,
    storage_path    TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_doc_entity ON document(entity_type, entity_id);

-- Audit trail
CREATE TABLE audit_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    metadata        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);

-- Climate risk zones
CREATE TABLE climate_risk_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    hazard_type     VARCHAR(50) NOT NULL,
    risk_level      VARCHAR(20) NOT NULL,
    geom            GEOMETRY(MultiPolygon, 4326) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_crz_geom ON climate_risk_zone USING GIST(geom);

-- Service requests
CREATE TABLE service_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    request_number  VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    reporter        JSONB,
    geom            GEOMETRY(Point, 4326),
    nearest_node_id UUID REFERENCES graph_node(id),      -- Closest network node to reported location
    priority        INTEGER DEFAULT 3,
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES app_user(id),
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, request_number)
);
CREATE INDEX idx_sr_status ON service_request(status);
CREATE INDEX idx_sr_geom ON service_request USING GIST(geom);
CREATE INDEX idx_sr_node ON service_request(nearest_node_id);
```

### AI & Capital Planning

```sql
CREATE TABLE condition_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    prediction_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    model_info      JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pred_asset ON condition_prediction(asset_id);
CREATE INDEX idx_pred_node ON condition_prediction(graph_node_id);

CREATE TABLE capital_programme (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    total_budget    NUMERIC(16,2),
    status          VARCHAR(50) DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE capital_candidate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id    UUID NOT NULL REFERENCES capital_programme(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    priority_rank   INTEGER,
    estimated_cost  NUMERIC(14,2),
    risk_score      NUMERIC(5,2),
    planned_year    INTEGER,
    status          VARCHAR(50) DEFAULT 'candidate',
    justification   JSONB,
    -- Graph-derived impact analysis stored here
    downstream_customer_count INTEGER,                   -- Computed from graph traversal
    downstream_critical_assets INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cap_prog ON capital_candidate(programme_id);
CREATE INDEX idx_cap_asset ON capital_candidate(asset_id);
```

---

## Example Graph Queries

### Downstream trace from a switch (find all affected customers)
```sql
-- Trace all nodes downstream of a switch, stopping at open switches/valves
WITH RECURSIVE downstream AS (
    -- Start from the edges leaving the switch node
    SELECT e.to_node_id AS node_id, e.id AS edge_id,
           ARRAY[e.from_node_id, e.to_node_id] AS path,
           1 AS depth
    FROM graph_edge e
    WHERE e.from_node_id = '<switch-node-uuid>'
      AND e.is_active = true

    UNION ALL

    -- Follow edges from each reached node
    SELECT e.to_node_id, e.id,
           d.path || e.to_node_id,
           d.depth + 1
    FROM graph_edge e
    JOIN downstream d ON e.from_node_id = d.node_id
    JOIN graph_node n ON n.id = d.node_id
    WHERE e.is_active = true
      AND e.to_node_id != ALL(d.path)          -- Prevent cycles
      AND (NOT n.is_switchable OR NOT n.is_open) -- Stop at open switches
      AND d.depth < 200                          -- Safety limit
)
SELECT n.id, n.node_type, n.name, n.properties,
       a.asset_code, a.condition_score,
       ST_AsGeoJSON(n.geom) AS geojson
FROM downstream d
JOIN graph_node n ON n.id = d.node_id
LEFT JOIN asset a ON a.graph_node_id = n.id
ORDER BY d.depth;
```

### Isolation analysis: find minimum switches to isolate a fault
```sql
-- Find the nearest upstream and downstream switches/valves that can isolate a faulted node
WITH RECURSIVE upstream_switches AS (
    SELECT e.from_node_id AS node_id, 1 AS depth
    FROM graph_edge e
    WHERE e.to_node_id = '<faulted-node-uuid>' AND e.is_active = true

    UNION ALL

    SELECT e.from_node_id, us.depth + 1
    FROM graph_edge e
    JOIN upstream_switches us ON e.to_node_id = us.node_id
    JOIN graph_node n ON n.id = e.from_node_id
    WHERE e.is_active = true
      AND NOT n.is_switchable  -- Stop when we reach a switchable device
      AND us.depth < 50
),
downstream_switches AS (
    SELECT e.to_node_id AS node_id, 1 AS depth
    FROM graph_edge e
    WHERE e.from_node_id = '<faulted-node-uuid>' AND e.is_active = true

    UNION ALL

    SELECT e.to_node_id, ds.depth + 1
    FROM graph_edge e
    JOIN downstream_switches ds ON e.from_node_id = ds.node_id
    JOIN graph_node n ON n.id = e.to_node_id
    WHERE e.is_active = true
      AND NOT n.is_switchable
      AND ds.depth < 50
)
-- Upstream isolation points
SELECT n.id, n.node_type, n.name, 'upstream' AS direction,
       us.depth, ST_AsGeoJSON(n.geom) AS geojson
FROM upstream_switches us
JOIN graph_node n ON n.id = us.node_id
WHERE n.is_switchable = true

UNION ALL

-- Downstream isolation points
SELECT n.id, n.node_type, n.name, 'downstream' AS direction,
       ds.depth, ST_AsGeoJSON(n.geom) AS geojson
FROM downstream_switches ds
JOIN graph_node n ON n.id = ds.node_id
WHERE n.is_switchable = true;
```

### Cross-utility impact: electric outage affecting water pumps
```sql
-- Find water pump stations that depend on a specific electric feeder
WITH RECURSIVE feeder_nodes AS (
    SELECT id AS node_id FROM graph_node WHERE id = '<feeder-head-uuid>'
    UNION ALL
    SELECT e.to_node_id
    FROM graph_edge e
    JOIN feeder_nodes fn ON e.from_node_id = fn.node_id
    JOIN graph_node n ON n.id = fn.node_id
    WHERE e.is_active = true AND e.utility_type = 'electric'
      AND (NOT n.is_switchable OR NOT n.is_open)
)
SELECT dep.dependent_node_id, wn.name, wn.node_type,
       wn.properties, ST_AsGeoJSON(wn.geom) AS geojson
FROM feeder_nodes fn
JOIN cross_utility_dependency dep ON dep.provider_node_id = fn.node_id
JOIN graph_node wn ON wn.id = dep.dependent_node_id
WHERE wn.utility_type = 'water'
  AND wn.node_type IN ('pump_station','treatment_plant');
```

### Shortest path for field crew routing
```sql
-- Find shortest path between two nodes using Dijkstra-like BFS with edge weights
WITH RECURSIVE path_search AS (
    SELECT e.to_node_id AS node_id,
           e.weight AS total_cost,
           ARRAY[e.from_node_id, e.to_node_id] AS path,
           1 AS depth
    FROM graph_edge e
    WHERE e.from_node_id = '<start-node-uuid>' AND e.is_active = true

    UNION ALL

    SELECT e.to_node_id,
           ps.total_cost + e.weight,
           ps.path || e.to_node_id,
           ps.depth + 1
    FROM graph_edge e
    JOIN path_search ps ON e.from_node_id = ps.node_id
    WHERE e.is_active = true
      AND e.to_node_id != ALL(ps.path)
      AND ps.depth < 100
)
SELECT path, total_cost
FROM path_search
WHERE node_id = '<end-node-uuid>'
ORDER BY total_cost ASC
LIMIT 1;
```

### Connected component analysis for subnetwork identification
```sql
-- Find all nodes in the same connected component as a given node
WITH RECURSIVE component AS (
    SELECT '<start-node-uuid>'::UUID AS node_id

    UNION

    SELECT CASE
        WHEN e.from_node_id = c.node_id THEN e.to_node_id
        ELSE e.from_node_id
    END AS node_id
    FROM graph_edge e
    JOIN component c ON (e.from_node_id = c.node_id OR e.to_node_id = c.node_id)
    WHERE e.is_active = true
      AND e.utility_type = '<utility-type>'
)
SELECT n.*, a.asset_code, a.condition_score
FROM component c
JOIN graph_node n ON n.id = c.node_id
LEFT JOIN asset a ON a.graph_node_id = n.id;
```

### Find assets in climate risk zones with graph context
```sql
-- Find high-risk assets and count their downstream dependents
WITH at_risk AS (
    SELECT n.id AS node_id, n.name, n.node_type,
           crz.hazard_type, crz.risk_level,
           a.asset_code, a.condition_score, a.replacement_cost
    FROM graph_node n
    JOIN asset a ON a.graph_node_id = n.id
    JOIN climate_risk_zone crz ON ST_Intersects(n.geom, crz.geom)
    WHERE crz.risk_level IN ('extreme','high')
      AND n.status = 'active'
),
downstream_counts AS (
    SELECT ar.node_id,
           COUNT(DISTINCT e.to_node_id) AS direct_downstream
    FROM at_risk ar
    JOIN graph_edge e ON e.from_node_id = ar.node_id AND e.is_active = true
    GROUP BY ar.node_id
)
SELECT ar.*, COALESCE(dc.direct_downstream, 0) AS downstream_count
FROM at_risk ar
LEFT JOIN downstream_counts dc ON dc.node_id = ar.node_id
ORDER BY ar.risk_level, ar.condition_score ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 4 | Nodes, edges, subnetworks, cross-utility dependencies |
| Core Identity | 2 | Organisations, users |
| Asset Register | 1 | Assets linked to graph nodes/edges |
| Work Orders | 2 | Work orders with affected node arrays; PM schedules |
| Inspections | 1 | Inspections linked to graph nodes |
| Sensors & Telemetry | 3 | Sensors, observations (partitioned), alerts -- all graph-linked |
| Inventory | 3 | Storerooms (graph-located), materials, balances |
| Compliance | 1 | Obligations with JSONB config |
| Documents & Audit | 2 | Documents, audit log (partitioned) |
| Risk & Climate | 1 | Climate risk zones with spatial overlay |
| Service Requests | 1 | Requests with nearest-node linking |
| AI & Capital | 3 | Predictions, programmes, candidates with graph-derived metrics |
| **Total** | **24** | Compact; graph layer absorbs topology that would be 4-6 tables in normalised model |

---

## Key Design Decisions

1. **Graph nodes and edges in PostgreSQL, not a separate database** -- The graph is implemented as two tables (graph_node, graph_edge) in PostgreSQL with PostGIS geometry columns. This keeps the entire system in one database, one transaction scope, and one backup strategy. The trade-off is that very large networks (10M+ nodes) may need migration to Apache AGE (PostgreSQL graph extension) or Neo4j, but the node/edge schema is directly compatible with both.

2. **Dual linking: asset references graph_node_id and graph_edge_id** -- Point assets (transformers, valves, hydrants) link to graph nodes. Linear assets (pipes, conductors) link to graph edges. This eliminates the need to decide whether an asset "is" a node or an edge -- it can be either, depending on its physical nature.

3. **is_switchable and is_open flags on graph nodes** -- Switch state is modeled directly on graph nodes rather than requiring a join to a switch_info table. The `is_switchable` flag marks nodes that can change state (switches, breakers, valves), and `is_open` tracks the current state. This makes graph traversal queries simpler -- the traversal CTE can check `is_open` directly without joining to equipment detail tables.

4. **Cross-utility dependency table** -- A dedicated table models dependencies between different utility networks (electric pole supports telecom cable, pump station powered by electric feeder). This enables cross-utility impact analysis that is impossible in siloed single-utility data models.

5. **Affected nodes/edges arrays on work orders** -- Work orders carry `affected_nodes` and `affected_edges` UUID arrays that record which parts of the network are impacted. This enables spatial and topological querying of active maintenance work and planned outages without running graph traversals at query time.

6. **Nearest-node linking for service requests** -- Service requests are spatially snapped to the nearest graph node, enabling immediate network context ("this leak report is near valve V-2201 on Main Street water main").

7. **Graph-derived metrics on capital candidates** -- The capital_candidate table stores `downstream_customer_count` and `downstream_critical_assets`, computed from graph traversal at planning time. This pre-computes the expensive graph operation so that capital programme ranking can use simple column sorts.

8. **Edge weight for shortest path and routing** -- Graph edges carry a `weight` column (defaulting to 1.0) that can represent physical length, impedance, flow resistance, or cost. This supports Dijkstra-like shortest path queries for field crew routing, network analysis, and capacity planning.
