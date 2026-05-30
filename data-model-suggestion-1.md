# Data Model Suggestion 1: Entity-Centric Normalized Relational (IEC CIM-Aligned)

> Project: Utility Asset Management · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational design where every domain concept gets its own table with strict foreign key relationships. The schema is explicitly aligned with the IEC Common Information Model (IEC 61970/61968), mapping CIM packages (Core, Assets, Work, Wires, Topology, Metering) to PostgreSQL table groups. Every equipment class in the CIM hierarchy (PowerSystemResource > Equipment > ConductingEquipment > ACLineSegment, etc.) has a corresponding table, enabling standards-compliant data exchange with SCADA, EMS, and other CIM-speaking systems.

The design prioritises data integrity and interoperability over development velocity. Reference tables enforce ISO standards for jurisdictions (ISO 3166), asset identifiers, and measurement units. Spatial data is stored natively in PostGIS geometry columns with SRID 4326 (WGS 84) as the default coordinate reference system, with support for projected CRS per jurisdiction. The schema supports multi-utility deployments (electric, gas, water) through a shared core with utility-type-specific extension tables.

This is the approach most similar to IBM Maximo, Esri UPDM, and enterprise EAM platforms. It trades simplicity for completeness and standards alignment.

**Best for:** Regulated utilities that must exchange data with SCADA/EMS systems via IEC CIM, undergo ISO 55000 certification audits, and need rigorous referential integrity across millions of asset records.

**Trade-offs:**
- (+) Maximum data integrity through foreign keys and constraints
- (+) Direct alignment with IEC CIM enables standards-compliant data exchange
- (+) Complex cross-entity queries perform well with proper indexing
- (+) Familiar to enterprise database teams
- (-) High table count (120+) increases schema complexity
- (-) Adding new asset types requires DDL changes (new tables/columns)
- (-) Rigid schema makes multi-jurisdiction variation harder to accommodate
- (-) Longer development time to implement all entity relationships

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IEC 61970-301 (CIM Core) | Table hierarchy mirrors CIM class inheritance: `power_system_resource` > `equipment` > `conducting_equipment` > specific types |
| IEC 61968-11 (CIM Distribution) | Asset, Work, Location, and Customer packages map to table groups |
| IEC 61968-4 (Records & Asset Mgmt) | Asset lifecycle states and condition records follow CIM message patterns |
| IEC 61968-6 (Maintenance & Construction) | Work order, crew, and schedule tables follow CIM work management classes |
| ISO 55000:2024 | Asset lifecycle fields (acquisition_date, commissioning_date, expected_disposal_date) support ISO 55001 evidence |
| ISO 3166-1/2 | `jurisdiction` reference table uses ISO 3166 codes for countries and subdivisions |
| OGC / RFC 7946 | PostGIS geometry columns; GeoJSON export via ST_AsGeoJSON(); OGC API Features compatibility |
| NERC CIP-002 | `bes_cyber_asset` table with impact_rating, categorization_date, and evidence_document linkage |
| W3C SOSA/SSN | Sensor and observation tables follow SOSA ontology (Sensor, Observation, ObservableProperty, FeatureOfInterest) |
| OpenAPI 3.1 | All table structures designed for direct mapping to OpenAPI schema components |

---

## Core Identity & Organisation

```sql
-- Organisation hierarchy (utility, department, division, crew)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL CHECK (org_type IN ('utility','division','department','crew','contractor')),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisation_parent ON organisation(parent_id);
CREATE INDEX idx_organisation_type ON organisation(org_type);

-- Jurisdiction reference (ISO 3166)
CREATE TABLE jurisdiction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,        -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),              -- ISO 3166-2 subdivision
    name            VARCHAR(255) NOT NULL,
    regulatory_body VARCHAR(255),
    timezone        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(country_code, subdivision_code)
);

-- Users and authentication
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_app_user_org ON app_user(organisation_id);

-- Role-based access control
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, organisation_id)
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,    -- e.g. 'asset', 'work_order', 'inspection'
    action          VARCHAR(50) NOT NULL,     -- e.g. 'read', 'write', 'delete', 'approve'
    UNIQUE(resource, action)
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id),
    permission_id   UUID NOT NULL REFERENCES permission(id),
    PRIMARY KEY (role_id, permission_id)
);
```

## Location & Spatial Model

```sql
-- Physical location (CIM Location class)
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255),
    location_type   VARCHAR(50) NOT NULL CHECK (location_type IN (
        'substation','pole','manhole','vault','cabinet','pump_station',
        'treatment_plant','reservoir','right_of_way','address','gps_point'
    )),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    jurisdiction_id UUID REFERENCES jurisdiction(id),
    geom            GEOMETRY(Geometry, 4326),  -- Point, LineString, or Polygon
    elevation_m     NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_location_geom ON location USING GIST(geom);
CREATE INDEX idx_location_type ON location(location_type);
CREATE INDEX idx_location_jurisdiction ON location(jurisdiction_id);

-- Linear referencing for pipelines, cables, roads
CREATE TABLE linear_reference_system (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    route_geom      GEOMETRY(LineString, 4326) NOT NULL,
    total_length_m  NUMERIC(12,2) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL CHECK (utility_type IN ('electric','gas','water','wastewater','telecom','stormwater')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_lrs_geom ON linear_reference_system USING GIST(route_geom);
```

## Asset Register (CIM Assets Package)

```sql
-- Asset classification / type catalogue
CREATE TABLE asset_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL CHECK (utility_type IN ('electric','gas','water','wastewater','telecom','stormwater','common')),
    category        VARCHAR(100) NOT NULL,    -- e.g. 'conductor', 'transformer', 'valve', 'pipe', 'meter'
    cim_class       VARCHAR(100),             -- IEC CIM class name, e.g. 'ACLineSegment', 'PowerTransformer'
    expected_life_years INTEGER,
    is_linear       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_asset_type_utility ON asset_type(utility_type);
CREATE INDEX idx_asset_type_category ON asset_type(category);

-- Core asset record (CIM Asset class)
CREATE TABLE asset (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_code          VARCHAR(100) NOT NULL UNIQUE,  -- Human-readable identifier
    asset_type_id       UUID NOT NULL REFERENCES asset_type(id),
    parent_id           UUID REFERENCES asset(id),     -- Asset hierarchy
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    location_id         UUID REFERENCES location(id),
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    serial_number       VARCHAR(100),
    manufacturer        VARCHAR(255),
    model_number        VARCHAR(100),
    install_date        DATE,
    commission_date     DATE,
    expected_disposal   DATE,
    actual_disposal     DATE,
    status              VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN (
        'planned','procurement','installed','active','degraded',
        'out_of_service','decommissioned','disposed'
    )),
    criticality         VARCHAR(20) CHECK (criticality IN ('critical','high','medium','low')),
    replacement_cost    NUMERIC(14,2),
    currency_code       CHAR(3) DEFAULT 'USD',         -- ISO 4217
    condition_score     NUMERIC(5,2),                  -- 0-100 scale
    risk_score          NUMERIC(5,2),                  -- Computed: condition * criticality * consequence
    geom                GEOMETRY(Geometry, 4326),       -- Spatial location (may differ from location.geom for precision)
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_asset_type ON asset(asset_type_id);
CREATE INDEX idx_asset_parent ON asset(parent_id);
CREATE INDEX idx_asset_org ON asset(organisation_id);
CREATE INDEX idx_asset_location ON asset(location_id);
CREATE INDEX idx_asset_status ON asset(status);
CREATE INDEX idx_asset_geom ON asset USING GIST(geom);
CREATE INDEX idx_asset_code ON asset(asset_code);
CREATE INDEX idx_asset_install_date ON asset(install_date);

-- Asset attributes (typed key-value for asset-type-specific fields)
CREATE TABLE asset_attribute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
    attribute_name  VARCHAR(100) NOT NULL,
    attribute_value TEXT,
    unit_of_measure VARCHAR(50),
    UNIQUE(asset_id, attribute_name)
);
CREATE INDEX idx_asset_attr_asset ON asset_attribute(asset_id);

-- Linear asset sections (for pipelines, cables, roads)
CREATE TABLE linear_asset_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
    lrs_id          UUID NOT NULL REFERENCES linear_reference_system(id),
    from_measure_m  NUMERIC(12,2) NOT NULL,
    to_measure_m    NUMERIC(12,2) NOT NULL,
    section_geom    GEOMETRY(LineString, 4326),
    material        VARCHAR(100),
    diameter_mm     NUMERIC(8,2),
    wall_thickness_mm NUMERIC(6,2),
    coating_type    VARCHAR(100),
    burial_depth_m  NUMERIC(6,2),
    install_date    DATE,
    condition_score NUMERIC(5,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (to_measure_m > from_measure_m)
);
CREATE INDEX idx_linear_section_asset ON linear_asset_section(asset_id);
CREATE INDEX idx_linear_section_lrs ON linear_asset_section(lrs_id);
CREATE INDEX idx_linear_section_geom ON linear_asset_section USING GIST(section_geom);
```

## Utility-Specific Equipment Tables (CIM Wires / Equipment Packages)

```sql
-- Electric: Transformer details
CREATE TABLE transformer_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    rated_power_kva     NUMERIC(10,2),
    primary_voltage_kv  NUMERIC(8,2),
    secondary_voltage_kv NUMERIC(8,2),
    phase_count         INTEGER CHECK (phase_count IN (1, 3)),
    cooling_type        VARCHAR(50),
    tap_changer_type    VARCHAR(50),
    oil_volume_litres   NUMERIC(10,2),
    pcb_content_ppm     NUMERIC(8,2),
    last_oil_test_date  DATE,
    dissolved_gas_status VARCHAR(50)
);

-- Electric: Conductor / cable details
CREATE TABLE conductor_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    conductor_material  VARCHAR(50),           -- e.g. 'ACSR', 'AAC', 'copper', 'aluminium'
    cross_section_mm2   NUMERIC(8,2),
    voltage_class_kv    NUMERIC(8,2),
    ampacity_a          NUMERIC(8,2),
    insulation_type     VARCHAR(50),
    is_underground      BOOLEAN NOT NULL DEFAULT false,
    span_length_m       NUMERIC(10,2)
);

-- Electric: Switch / breaker details
CREATE TABLE switch_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    switch_type         VARCHAR(50) NOT NULL,   -- 'breaker','recloser','fuse','disconnect','sectionaliser'
    rated_current_a     NUMERIC(8,2),
    rated_voltage_kv    NUMERIC(8,2),
    interrupting_capacity_ka NUMERIC(8,2),
    is_normally_open    BOOLEAN NOT NULL DEFAULT false,
    is_remote_controlled BOOLEAN NOT NULL DEFAULT false,
    scada_point_id      VARCHAR(100)
);

-- Water: Pipe details
CREATE TABLE pipe_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    pipe_material       VARCHAR(50),            -- 'ductile_iron','pvc','hdpe','cast_iron','concrete','steel'
    nominal_diameter_mm NUMERIC(8,2),
    pressure_class      VARCHAR(50),
    lining_type         VARCHAR(50),
    joint_type          VARCHAR(50),
    cathodic_protection BOOLEAN DEFAULT false
);

-- Water: Valve details
CREATE TABLE valve_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    valve_type          VARCHAR(50) NOT NULL,    -- 'gate','butterfly','check','pressure_reducing','air_release'
    nominal_diameter_mm NUMERIC(8,2),
    turns_to_close      INTEGER,
    is_normally_open    BOOLEAN NOT NULL DEFAULT true,
    last_exercise_date  DATE,
    exercise_result     VARCHAR(50)
);

-- Water: Hydrant details
CREATE TABLE hydrant_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    hydrant_type        VARCHAR(50),             -- 'dry_barrel','wet_barrel'
    outlet_count        INTEGER,
    main_diameter_mm    NUMERIC(8,2),
    flow_rate_lpm       NUMERIC(10,2),
    static_pressure_kpa NUMERIC(8,2),
    last_flow_test_date DATE,
    nfpa_classification VARCHAR(20)              -- NFPA colour code
);

-- Gas: Regulator / meter station details
CREATE TABLE gas_regulator_info (
    asset_id            UUID PRIMARY KEY REFERENCES asset(id) ON DELETE CASCADE,
    regulator_type      VARCHAR(50),
    inlet_pressure_kpa  NUMERIC(10,2),
    outlet_pressure_kpa NUMERIC(10,2),
    capacity_m3_hr      NUMERIC(10,2),
    relief_valve_setting_kpa NUMERIC(10,2),
    odorizer_type       VARCHAR(50)
);
```

## Network Topology (CIM Topology Package)

```sql
-- Connectivity node (CIM ConnectivityNode)
CREATE TABLE connectivity_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255),
    utility_type    VARCHAR(50) NOT NULL,
    location_id     UUID REFERENCES location(id),
    geom            GEOMETRY(Point, 4326),
    is_boundary     BOOLEAN NOT NULL DEFAULT false,  -- Boundary between subnetworks
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conn_node_geom ON connectivity_node USING GIST(geom);
CREATE INDEX idx_conn_node_utility ON connectivity_node(utility_type);

-- Terminal: connection point on an asset (CIM Terminal)
CREATE TABLE terminal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
    connectivity_node_id UUID REFERENCES connectivity_node(id),
    sequence_number INTEGER NOT NULL DEFAULT 1,
    terminal_name   VARCHAR(100),
    is_connected    BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(asset_id, sequence_number)
);
CREATE INDEX idx_terminal_asset ON terminal(asset_id);
CREATE INDEX idx_terminal_conn_node ON terminal(connectivity_node_id);

-- Subnetwork definition
CREATE TABLE subnetwork (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL,
    subnetwork_type VARCHAR(50),               -- e.g. 'feeder','pressure_zone','collection_basin'
    source_asset_id UUID REFERENCES asset(id), -- Feeder head / pump station / regulator
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Subnetwork membership
CREATE TABLE subnetwork_member (
    subnetwork_id   UUID NOT NULL REFERENCES subnetwork(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    PRIMARY KEY (subnetwork_id, asset_id)
);
```

## Work Order Management (CIM Work Package)

```sql
-- Work order types and priority reference
CREATE TABLE work_order_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,  -- 'corrective','preventive','condition_based','emergency','capital'
    description     TEXT,
    default_priority INTEGER NOT NULL DEFAULT 3
);

-- Work order (CIM Work / WorkTask)
CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_number VARCHAR(50) NOT NULL UNIQUE,
    parent_id       UUID REFERENCES work_order(id),   -- Child work orders
    work_order_type_id UUID NOT NULL REFERENCES work_order_type(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    asset_id        UUID REFERENCES asset(id),
    location_id     UUID REFERENCES location(id),
    reported_by     UUID REFERENCES app_user(id),
    assigned_to     UUID REFERENCES app_user(id),
    assigned_crew   UUID REFERENCES organisation(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    priority        INTEGER NOT NULL DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft','pending_approval','approved','scheduled','in_progress',
        'on_hold','completed','closed','cancelled'
    )),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    estimated_hours NUMERIC(8,2),
    actual_hours    NUMERIC(8,2),
    estimated_cost  NUMERIC(14,2),
    actual_cost     NUMERIC(14,2),
    failure_class   VARCHAR(100),
    failure_cause   VARCHAR(100),
    failure_remedy  VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_location ON work_order(location_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_crew ON work_order(assigned_crew);
CREATE INDEX idx_wo_type ON work_order(work_order_type_id);
CREATE INDEX idx_wo_scheduled ON work_order(scheduled_start);
CREATE INDEX idx_wo_parent ON work_order(parent_id);

-- Work order tasks
CREATE TABLE work_order_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    sequence_number INTEGER NOT NULL,
    description     TEXT NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    estimated_hours NUMERIC(6,2),
    actual_hours    NUMERIC(6,2),
    completed_at    TIMESTAMPTZ,
    completed_by    UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_task_wo ON work_order_task(work_order_id);

-- Labour time tracking
CREATE TABLE work_order_labour (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    hours           NUMERIC(6,2),
    labour_type     VARCHAR(50),               -- 'regular','overtime','travel'
    hourly_rate     NUMERIC(8,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_labour_wo ON work_order_labour(work_order_id);
CREATE INDEX idx_wo_labour_user ON work_order_labour(user_id);

-- Preventive maintenance schedule
CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    asset_type_id   UUID REFERENCES asset_type(id),
    asset_id        UUID REFERENCES asset(id),         -- Specific asset or null for type-based
    work_order_type_id UUID NOT NULL REFERENCES work_order_type(id),
    trigger_type    VARCHAR(50) NOT NULL CHECK (trigger_type IN ('time','meter','condition')),
    interval_days   INTEGER,                           -- For time-based triggers
    meter_threshold NUMERIC(12,2),                     -- For meter-based triggers
    condition_threshold NUMERIC(5,2),                  -- For condition-based triggers
    lead_time_days  INTEGER DEFAULT 7,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pm_schedule_asset ON pm_schedule(asset_id);
CREATE INDEX idx_pm_schedule_type ON pm_schedule(asset_type_id);
CREATE INDEX idx_pm_schedule_next ON pm_schedule(next_due);
```

## Inspection & Condition Assessment

```sql
-- Inspection template
CREATE TABLE inspection_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    asset_type_id   UUID REFERENCES asset_type(id),
    version         INTEGER NOT NULL DEFAULT 1,
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inspection template questions/items
CREATE TABLE inspection_template_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES inspection_template(id) ON DELETE CASCADE,
    sequence_number INTEGER NOT NULL,
    category        VARCHAR(100),
    question_text   TEXT NOT NULL,
    response_type   VARCHAR(50) NOT NULL CHECK (response_type IN (
        'boolean','numeric','text','choice','photo','measurement'
    )),
    choices         TEXT[],                     -- For 'choice' response type
    unit_of_measure VARCHAR(50),               -- For 'numeric'/'measurement' response type
    weight          NUMERIC(5,2) DEFAULT 1.0,  -- Weight in condition score calculation
    is_required     BOOLEAN NOT NULL DEFAULT true,
    defect_threshold NUMERIC(8,2),             -- Value that triggers defect flag
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insp_item_template ON inspection_template_item(template_id);

-- Inspection record
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    template_id     UUID NOT NULL REFERENCES inspection_template(id),
    work_order_id   UUID REFERENCES work_order(id),
    inspector_id    UUID NOT NULL REFERENCES app_user(id),
    inspection_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_score   NUMERIC(5,2),              -- Computed from item responses
    overall_grade   VARCHAR(10),               -- e.g. 'A','B','C','D','F' or '1'-'5'
    status          VARCHAR(50) NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft','in_progress','completed','reviewed','approved'
    )),
    notes           TEXT,
    geom            GEOMETRY(Point, 4326),     -- GPS location at time of inspection
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inspection_asset ON inspection(asset_id);
CREATE INDEX idx_inspection_date ON inspection(inspection_date);
CREATE INDEX idx_inspection_status ON inspection(status);

-- Inspection responses
CREATE TABLE inspection_response (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id       UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    template_item_id    UUID NOT NULL REFERENCES inspection_template_item(id),
    response_boolean    BOOLEAN,
    response_numeric    NUMERIC(12,4),
    response_text       TEXT,
    response_choice     VARCHAR(255),
    is_defect           BOOLEAN NOT NULL DEFAULT false,
    defect_severity     VARCHAR(20) CHECK (defect_severity IN ('critical','major','minor','observation')),
    score_contribution  NUMERIC(5,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insp_resp_inspection ON inspection_response(inspection_id);
CREATE INDEX idx_insp_resp_defect ON inspection_response(is_defect) WHERE is_defect = true;

-- Inspection photos / attachments
CREATE TABLE inspection_attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspection(id) ON DELETE CASCADE,
    response_id     UUID REFERENCES inspection_response(id),
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(50) NOT NULL,
    file_size_bytes BIGINT,
    storage_path    TEXT NOT NULL,
    geom            GEOMETRY(Point, 4326),
    captured_at     TIMESTAMPTZ,
    ai_analysis     TEXT,                      -- CV defect detection results
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insp_attach_inspection ON inspection_attachment(inspection_id);
```

## IoT Sensor & Telemetry (SOSA/SSN-aligned)

```sql
-- Sensor definition (SOSA Sensor)
CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    sensor_type     VARCHAR(100) NOT NULL,     -- e.g. 'vibration','temperature','pressure','flow','voltage'
    manufacturer    VARCHAR(255),
    model           VARCHAR(100),
    serial_number   VARCHAR(100),
    observable_property VARCHAR(100) NOT NULL,  -- SOSA ObservableProperty
    unit_of_measure VARCHAR(50) NOT NULL,
    min_threshold   NUMERIC(12,4),
    max_threshold   NUMERIC(12,4),
    sampling_interval_seconds INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    installed_date  DATE,
    last_reading_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_asset ON sensor(asset_id);
CREATE INDEX idx_sensor_type ON sensor(sensor_type);

-- Sensor observations (SOSA Observation) — partitioned by month
CREATE TABLE sensor_observation (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    value_numeric   NUMERIC(16,6),
    value_text      VARCHAR(255),
    quality_flag    VARCHAR(20) DEFAULT 'good',  -- 'good','suspect','bad','missing'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (observed_at);
CREATE INDEX idx_obs_sensor_time ON sensor_observation(sensor_id, observed_at);

-- Create monthly partitions (example)
-- CREATE TABLE sensor_observation_2026_01 PARTITION OF sensor_observation
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Sensor alerts (threshold exceedances)
CREATE TABLE sensor_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,      -- 'high_threshold','low_threshold','rate_of_change','anomaly'
    threshold_value NUMERIC(12,4),
    actual_value    NUMERIC(12,4),
    severity        VARCHAR(20) NOT NULL CHECK (severity IN ('critical','warning','info')),
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES app_user(id),
    work_order_id   UUID REFERENCES work_order(id),  -- Auto-generated WO
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_sensor ON sensor_alert(sensor_id);
CREATE INDEX idx_alert_severity ON sensor_alert(severity);
CREATE INDEX idx_alert_unack ON sensor_alert(is_acknowledged) WHERE is_acknowledged = false;
```

## Inventory & Materials

```sql
-- Storeroom / warehouse
CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    location_id     UUID REFERENCES location(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Spare part / material catalogue
CREATE TABLE material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_number     VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
    unit_of_issue   VARCHAR(50) NOT NULL,      -- 'each','metre','litre','kg'
    unit_cost       NUMERIC(12,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    reorder_point   NUMERIC(10,2),
    reorder_quantity NUMERIC(10,2),
    lead_time_days  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_material_category ON material(category);

-- Storeroom inventory balances
CREATE TABLE inventory_balance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    material_id     UUID NOT NULL REFERENCES material(id),
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 0,
    last_count_date DATE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(storeroom_id, material_id)
);

-- Material usage on work orders
CREATE TABLE work_order_material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    material_id     UUID NOT NULL REFERENCES material(id),
    storeroom_id    UUID REFERENCES storeroom(id),
    quantity_planned NUMERIC(10,2),
    quantity_used   NUMERIC(10,2),
    unit_cost       NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wo_material_wo ON work_order_material(work_order_id);

-- Compatible materials for asset types
CREATE TABLE asset_type_material (
    asset_type_id   UUID NOT NULL REFERENCES asset_type(id),
    material_id     UUID NOT NULL REFERENCES material(id),
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (asset_type_id, material_id)
);
```

## Compliance & Audit

```sql
-- NERC CIP BES Cyber Asset inventory
CREATE TABLE bes_cyber_asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    bes_impact_rating VARCHAR(20) NOT NULL CHECK (bes_impact_rating IN ('high','medium','low')),
    cyber_system_name VARCHAR(255),
    categorization_date DATE NOT NULL,
    review_due_date DATE NOT NULL,             -- 15-month review cycle (NERC CIP-002)
    responsible_entity UUID REFERENCES organisation(id),
    evidence_notes  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bes_asset ON bes_cyber_asset(asset_id);
CREATE INDEX idx_bes_review ON bes_cyber_asset(review_due_date);

-- Compliance obligation tracking
CREATE TABLE compliance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework       VARCHAR(100) NOT NULL,     -- 'NERC_CIP','ISO_55000','AWWA','OSHA','EPA'
    standard_ref    VARCHAR(100) NOT NULL,     -- e.g. 'CIP-002-5.1a', 'ISO 55001:2024 Clause 8.1'
    obligation_text TEXT NOT NULL,
    frequency       VARCHAR(50),               -- 'annual','quarterly','15_months','continuous'
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    responsible_user UUID REFERENCES app_user(id),
    last_completed  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    status          VARCHAR(50) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compliance_framework ON compliance_obligation(framework);
CREATE INDEX idx_compliance_due ON compliance_obligation(next_due);

-- Audit trail (immutable append-only log)
CREATE TABLE audit_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(50) NOT NULL,      -- 'create','update','delete','approve','login'
    entity_type     VARCHAR(100) NOT NULL,     -- e.g. 'asset','work_order','inspection'
    entity_id       UUID NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

## AI & Analytics

```sql
-- AI condition assessment predictions
CREATE TABLE condition_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    model_name      VARCHAR(100) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    prediction_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    remaining_useful_life_days INTEGER,
    failure_probability_1yr NUMERIC(5,4),      -- 0.0000 to 1.0000
    failure_probability_5yr NUMERIC(5,4),
    confidence_score NUMERIC(5,4),
    risk_rank       INTEGER,                   -- Relative rank within asset class
    input_features  JSONB,                     -- Features used for prediction
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cond_pred_asset ON condition_prediction(asset_id);
CREATE INDEX idx_cond_pred_date ON condition_prediction(prediction_date);

-- Capital replacement programme
CREATE TABLE capital_programme (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    total_budget    NUMERIC(16,2),
    currency_code   CHAR(3) DEFAULT 'USD',
    status          VARCHAR(50) DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Capital replacement candidates
CREATE TABLE capital_replacement_candidate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id    UUID NOT NULL REFERENCES capital_programme(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    priority_rank   INTEGER,
    estimated_cost  NUMERIC(14,2),
    risk_score      NUMERIC(5,2),
    consequence_of_failure TEXT,
    planned_year    INTEGER,
    status          VARCHAR(50) DEFAULT 'candidate',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cap_candidate_prog ON capital_replacement_candidate(programme_id);
CREATE INDEX idx_cap_candidate_asset ON capital_replacement_candidate(asset_id);

-- Climate risk overlay
CREATE TABLE climate_risk_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    hazard_type     VARCHAR(50) NOT NULL CHECK (hazard_type IN (
        'flood','wildfire','hurricane','earthquake','extreme_heat','ice_storm','landslide'
    )),
    risk_level      VARCHAR(20) NOT NULL CHECK (risk_level IN ('extreme','high','moderate','low')),
    geom            GEOMETRY(MultiPolygon, 4326) NOT NULL,
    data_source     VARCHAR(255),
    effective_date  DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_climate_risk_geom ON climate_risk_zone USING GIST(geom);
CREATE INDEX idx_climate_risk_hazard ON climate_risk_zone(hazard_type);
```

## Document & Attachment Management

```sql
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title           VARCHAR(500) NOT NULL,
    doc_type        VARCHAR(50) NOT NULL,       -- 'manual','drawing','certificate','report','photo','video'
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(50),
    file_size_bytes BIGINT,
    storage_path    TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Polymorphic document linkage
CREATE TABLE document_link (
    document_id     UUID NOT NULL REFERENCES document(id),
    entity_type     VARCHAR(100) NOT NULL,     -- 'asset','work_order','inspection','location'
    entity_id       UUID NOT NULL,
    PRIMARY KEY (document_id, entity_type, entity_id)
);
CREATE INDEX idx_doc_link_entity ON document_link(entity_type, entity_id);
```

## Service Request

```sql
CREATE TABLE service_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_number  VARCHAR(50) NOT NULL UNIQUE,
    reporter_name   VARCHAR(255),
    reporter_email  VARCHAR(255),
    reporter_phone  VARCHAR(50),
    category        VARCHAR(100) NOT NULL,     -- 'leak','outage','damage','odour','other'
    description     TEXT NOT NULL,
    location_description TEXT,
    geom            GEOMETRY(Point, 4326),
    priority        INTEGER DEFAULT 3,
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES app_user(id),
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sr_status ON service_request(status);
CREATE INDEX idx_sr_geom ON service_request USING GIST(geom);
```

---

## Example Queries

### Recursive asset hierarchy traversal
```sql
WITH RECURSIVE asset_tree AS (
    SELECT id, asset_code, name, parent_id, 0 AS depth
    FROM asset WHERE id = '<root-asset-id>'
    UNION ALL
    SELECT a.id, a.asset_code, a.name, a.parent_id, t.depth + 1
    FROM asset a JOIN asset_tree t ON a.parent_id = t.id
)
SELECT * FROM asset_tree ORDER BY depth;
```

### Network trace: find all assets downstream of a switch
```sql
WITH RECURSIVE downstream AS (
    SELECT t2.asset_id
    FROM terminal t1
    JOIN terminal t2 ON t1.connectivity_node_id = t2.connectivity_node_id
    WHERE t1.asset_id = '<switch-asset-id>' AND t2.asset_id != t1.asset_id
    UNION
    SELECT t4.asset_id
    FROM downstream d
    JOIN terminal t3 ON t3.asset_id = d.asset_id
    JOIN terminal t4 ON t3.connectivity_node_id = t4.connectivity_node_id
        AND t4.asset_id != t3.asset_id
    JOIN asset a ON a.id = t4.asset_id
    LEFT JOIN switch_info si ON si.asset_id = a.id
    WHERE si.asset_id IS NULL OR si.is_normally_open = false
)
SELECT DISTINCT a.* FROM downstream d JOIN asset a ON a.id = d.asset_id;
```

### Assets in climate risk zones
```sql
SELECT a.asset_code, a.name, at.name AS asset_type,
       crz.hazard_type, crz.risk_level
FROM asset a
JOIN asset_type at ON a.asset_type_id = at.id
JOIN climate_risk_zone crz ON ST_Intersects(a.geom, crz.geom)
WHERE crz.risk_level IN ('extreme','high')
ORDER BY crz.risk_level, a.condition_score;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Organisation | 6 | Users, roles, permissions, organisations, jurisdictions |
| Location & Spatial | 2 | Locations, linear reference systems |
| Asset Register | 4 | Assets, attributes, types, linear sections |
| Equipment Detail (electric) | 3 | Transformers, conductors, switches |
| Equipment Detail (water) | 3 | Pipes, valves, hydrants |
| Equipment Detail (gas) | 1 | Regulators |
| Network Topology | 4 | Connectivity nodes, terminals, subnetworks |
| Work Order Management | 6 | Work orders, tasks, labour, PM schedules, materials |
| Inspection & Condition | 5 | Templates, items, inspections, responses, attachments |
| IoT & Telemetry | 3 | Sensors, observations (partitioned), alerts |
| Inventory & Materials | 5 | Storerooms, materials, balances, usage, compatibility |
| Compliance & Audit | 4 | BES cyber assets, obligations, audit log (partitioned) |
| AI & Analytics | 4 | Predictions, capital programmes, candidates, climate risk |
| Documents | 2 | Documents, polymorphic links |
| Service Requests | 1 | Public-facing intake |
| **Total** | **53** | Expandable to 120+ with additional utility types |

---

## Key Design Decisions

1. **IEC CIM class hierarchy mapped to relational inheritance** — Asset types mirror the CIM PowerSystemResource > Equipment > ConductingEquipment inheritance chain, with utility-specific detail tables (transformer_info, pipe_info) using shared primary keys with the parent asset table. This enables CIM-compliant data export without runtime transformation.

2. **PostGIS native geometry on both location and asset** — Assets carry their own geometry column separate from their parent location, because precision field GPS coordinates for an individual asset often differ from the location's reference point. GIST indexes on all geometry columns enable spatial queries and OGC API Features serving.

3. **Linear referencing as a first-class concept** — Pipelines, cables, and roads are modeled with a dedicated linear_reference_system table and linear_asset_section table using from/to measures. This mirrors the Esri UPDM approach and supports section-level attribute recording and defect location.

4. **Connectivity-node topology for network tracing** — The CIM Terminal/ConnectivityNode pattern enables recursive SQL-based network tracing (upstream/downstream, isolation) without requiring a graph database. Performance is adequate for networks up to ~1M nodes with proper indexing.

5. **Sensor observations partitioned by time** — The sensor_observation table uses PostgreSQL declarative partitioning by month, keeping per-partition sizes manageable and enabling efficient time-range queries for telemetry dashboards.

6. **Audit log as append-only partitioned table** — The audit_log captures old/new JSONB values for every state change, supporting NERC CIP evidence generation and ISO 55000 certification audits. Partitioning by time prevents the audit table from becoming a performance bottleneck.

7. **Inspection scoring engine in the schema** — Inspection templates with weighted items and defect thresholds enable automated condition score calculation. The score propagates to the parent asset's condition_score field, feeding into risk ranking and AI prediction models.

8. **Separate equipment detail tables per asset class** — Rather than a single wide table with nullable columns for all equipment types, each equipment class gets its own table (transformer_info, pipe_info, valve_info). This maintains data integrity and avoids sparse-column antipatterns, at the cost of requiring joins for cross-type queries.
