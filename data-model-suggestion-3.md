# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Utility Asset Management · Created: 2026-05-22

## Philosophy

This model keeps core structural relationships in normalized relational tables (assets, work orders, inspections, network topology) while pushing variable, domain-specific, and jurisdiction-dependent data into JSONB columns. The insight is that a utility asset management platform must serve electric, gas, water, wastewater, stormwater, and telecom utilities across dozens of jurisdictions -- each with different asset attributes, inspection requirements, regulatory fields, and reporting formats. A fully normalized schema (Suggestion 1) handles this by adding new tables and columns for every variation; this hybrid approach handles it by allowing the JSONB columns to absorb the variation without DDL changes.

The relational core provides foreign key integrity, efficient joins, and standard indexing for the fields that are universal across all utility types: asset identity, hierarchy, location, status, work order lifecycle, and user management. The JSONB columns provide schema-flexible storage for fields that vary by asset type (transformer ratings vs. pipe materials vs. valve specifications), by jurisdiction (NERC CIP fields for US electric vs. Ofgem fields for UK gas), and by organisation (custom fields defined by individual utility operators).

This is the approach best suited for a rapid MVP that needs to support multiple utility types from day one without a massive upfront schema design effort. It also supports multi-tenant SaaS deployments where different tenants have different custom field requirements.

**Best for:** Multi-utility platforms serving electric, gas, and water from one codebase; SaaS deployments with per-tenant customisation; rapid MVP development where the full domain is not yet understood; organisations that need to add custom fields without developer involvement.

**Trade-offs:**
- (+) Fast to develop: core schema is compact; new asset types don't require DDL changes
- (+) Multi-utility support from day one without separate schemas per utility type
- (+) Per-tenant custom fields are trivial (just JSONB keys)
- (+) Schema evolution is easier -- add new JSONB fields without migrations
- (+) Smaller table count than fully normalised model
- (-) JSONB fields are harder to enforce at the database level (constraints require triggers or CHECK with jsonb_typeof)
- (-) Complex JSONB queries can be slower than indexed column queries (though GIN indexes help)
- (-) Reporting tools may struggle with JSONB columns; requires JSONB extraction in views
- (-) Risk of inconsistent JSONB structures across records without application-level validation
- (-) Developers must understand both relational and JSONB query patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IEC 61970/61968 (CIM) | Core relational tables align with CIM class hierarchy; CIM-specific attributes stored in JSONB with CIM field names |
| ISO 55000:2024 | Asset lifecycle fields in relational columns; ISO 55001 evidence artefacts in JSONB metadata |
| ISO 3166-1/2 | Jurisdiction codes as relational columns with ISO 3166 values |
| NERC CIP | Compliance fields stored in JSONB on assets and compliance tables; adaptable to CIP version changes without DDL |
| OGC / RFC 7946 | PostGIS geometry columns for spatial data; GeoJSON in JSONB for portable asset exchange |
| W3C SOSA/SSN | Sensor configuration and observation metadata in JSONB following SOSA property names |
| OpenAPI 3.1 | Core relational columns map to required schema properties; JSONB columns map to additionalProperties |

---

## Core Identity & Organisation

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(50) NOT NULL,
    country_code    CHAR(2),                            -- ISO 3166-1
    subdivision_code VARCHAR(6),                         -- ISO 3166-2
    timezone        VARCHAR(50),
    settings        JSONB NOT NULL DEFAULT '{}',         -- Tenant-specific configuration
    -- Example settings:
    -- {
    --   "utility_types": ["electric", "water"],
    --   "default_currency": "USD",
    --   "fiscal_year_start_month": 7,
    --   "custom_asset_fields": {
    --     "transformer": [
    --       {"name": "pcb_content_ppm", "type": "number", "label": "PCB Content (ppm)"}
    --     ]
    --   },
    --   "compliance_frameworks": ["NERC_CIP", "ISO_55000"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_parent ON organisation(parent_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '[]',         -- Per-user permission overrides
    preferences     JSONB NOT NULL DEFAULT '{}',         -- UI preferences, default map view, etc.
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_user_org ON app_user(organisation_id);
CREATE INDEX idx_user_role ON app_user(role);
```

## Location & Spatial

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255),
    location_type   VARCHAR(50) NOT NULL,
    geom            GEOMETRY(Geometry, 4326),
    address         JSONB,                              -- Flexible address format per jurisdiction
    -- Example address (US):
    -- {
    --   "line1": "123 Main Street",
    --   "city": "Atlanta",
    --   "state": "GA",
    --   "postal_code": "30301",
    --   "country": "US"
    -- }
    -- Example address (UK):
    -- {
    --   "line1": "10 Downing Street",
    --   "city": "London",
    --   "postcode": "SW1A 2AA",
    --   "country": "GB"
    -- }
    properties      JSONB NOT NULL DEFAULT '{}',         -- Additional location metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_location_geom ON location USING GIST(geom);
CREATE INDEX idx_location_org ON location(organisation_id);
CREATE INDEX idx_location_type ON location(location_type);
```

## Asset Register

```sql
-- Asset type catalogue with JSONB schema definition
CREATE TABLE asset_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),   -- NULL = global, non-NULL = tenant-specific
    name            VARCHAR(255) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    cim_class       VARCHAR(100),
    expected_life_years INTEGER,
    is_linear       BOOLEAN NOT NULL DEFAULT false,
    -- JSON Schema defining the expected structure of asset.properties for this type
    properties_schema JSONB,
    -- Example properties_schema for transformer:
    -- {
    --   "type": "object",
    --   "properties": {
    --     "rated_power_kva": {"type": "number"},
    --     "primary_voltage_kv": {"type": "number"},
    --     "secondary_voltage_kv": {"type": "number"},
    --     "phase_count": {"type": "integer", "enum": [1, 3]},
    --     "cooling_type": {"type": "string"},
    --     "oil_volume_litres": {"type": "number"},
    --     "tap_changer_type": {"type": "string"}
    --   },
    --   "required": ["rated_power_kva", "primary_voltage_kv"]
    -- }
    inspection_template JSONB,                          -- Default inspection items for this type
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_asset_type_utility ON asset_type(utility_type);
CREATE INDEX idx_asset_type_org ON asset_type(organisation_id);

-- Core asset record
CREATE TABLE asset (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_code          VARCHAR(100) NOT NULL,
    asset_type_id       UUID NOT NULL REFERENCES asset_type(id),
    parent_id           UUID REFERENCES asset(id),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    location_id         UUID REFERENCES location(id),
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(50) NOT NULL DEFAULT 'active' CHECK (status IN (
        'planned','procurement','installed','active','degraded',
        'out_of_service','decommissioned','disposed'
    )),
    -- Universal relational fields (present on all assets)
    install_date        DATE,
    commission_date     DATE,
    serial_number       VARCHAR(100),
    manufacturer        VARCHAR(255),
    model_number        VARCHAR(100),
    criticality         VARCHAR(20),
    replacement_cost    NUMERIC(14,2),
    currency_code       CHAR(3) DEFAULT 'USD',
    condition_score     NUMERIC(5,2),
    risk_score          NUMERIC(5,2),
    geom                GEOMETRY(Geometry, 4326),

    -- JSONB for asset-type-specific and custom fields
    properties          JSONB NOT NULL DEFAULT '{}',
    -- Example properties for a transformer:
    -- {
    --   "rated_power_kva": 750,
    --   "primary_voltage_kv": 12.47,
    --   "secondary_voltage_kv": 0.208,
    --   "phase_count": 3,
    --   "cooling_type": "ONAN",
    --   "oil_volume_litres": 450,
    --   "tap_changer_type": "OLTC",
    --   "last_oil_test_date": "2025-11-15",
    --   "dissolved_gas_status": "normal"
    -- }
    --
    -- Example properties for a water pipe:
    -- {
    --   "pipe_material": "ductile_iron",
    --   "nominal_diameter_mm": 300,
    --   "pressure_class": "C30",
    --   "lining_type": "cement_mortar",
    --   "joint_type": "push_on",
    --   "cathodic_protection": true,
    --   "burial_depth_m": 1.2
    -- }
    --
    -- Example properties for a hydrant:
    -- {
    --   "hydrant_type": "dry_barrel",
    --   "outlet_count": 3,
    --   "main_diameter_mm": 150,
    --   "flow_rate_lpm": 2800,
    --   "static_pressure_kpa": 350,
    --   "nfpa_classification": "Class AA",
    --   "last_flow_test_date": "2026-02-20"
    -- }

    -- JSONB for linear asset details (populated only for linear assets)
    linear_detail       JSONB,
    -- Example linear_detail:
    -- {
    --   "total_length_m": 1250.5,
    --   "sections": [
    --     {
    --       "from_m": 0, "to_m": 500,
    --       "material": "ductile_iron", "diameter_mm": 300,
    --       "install_date": "1985-06-01", "condition_score": 45
    --     },
    --     {
    --       "from_m": 500, "to_m": 1250.5,
    --       "material": "pvc", "diameter_mm": 300,
    --       "install_date": "2010-03-15", "condition_score": 88
    --     }
    --   ],
    --   "route_geojson": { "type": "LineString", "coordinates": [...] }
    -- }

    -- JSONB for compliance / regulatory metadata
    compliance          JSONB NOT NULL DEFAULT '{}',
    -- Example compliance for a BES cyber asset:
    -- {
    --   "nerc_cip": {
    --     "bes_impact_rating": "medium",
    --     "cyber_system_name": "Substation Alpha SCADA",
    --     "categorization_date": "2025-09-01",
    --     "review_due_date": "2026-12-01"
    --   },
    --   "iso_55000": {
    --     "strategic_plan_ref": "SAMP-2025-001",
    --     "lifecycle_stage": "operate"
    --   }
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, asset_code)
);
CREATE INDEX idx_asset_type ON asset(asset_type_id);
CREATE INDEX idx_asset_parent ON asset(parent_id);
CREATE INDEX idx_asset_org ON asset(organisation_id);
CREATE INDEX idx_asset_status ON asset(status);
CREATE INDEX idx_asset_geom ON asset USING GIST(geom);
CREATE INDEX idx_asset_install ON asset(install_date);
CREATE INDEX idx_asset_condition ON asset(condition_score);
-- GIN indexes for JSONB querying
CREATE INDEX idx_asset_properties ON asset USING GIN(properties jsonb_path_ops);
CREATE INDEX idx_asset_compliance ON asset USING GIN(compliance jsonb_path_ops);
-- Partial index for specific property queries
CREATE INDEX idx_asset_rated_power ON asset ((properties->>'rated_power_kva'))
    WHERE properties ? 'rated_power_kva';
```

## Network Topology

```sql
-- Connectivity nodes (relational for efficient graph traversal)
CREATE TABLE connectivity_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    utility_type    VARCHAR(50) NOT NULL,
    geom            GEOMETRY(Point, 4326),
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cn_geom ON connectivity_node USING GIST(geom);
CREATE INDEX idx_cn_utility ON connectivity_node(utility_type);

-- Terminal: asset connection point
CREATE TABLE terminal (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
    connectivity_node_id UUID REFERENCES connectivity_node(id),
    sequence_number     INTEGER NOT NULL DEFAULT 1,
    is_connected        BOOLEAN NOT NULL DEFAULT true,
    properties          JSONB NOT NULL DEFAULT '{}',     -- Phase, direction, flow info
    UNIQUE(asset_id, sequence_number)
);
CREATE INDEX idx_terminal_asset ON terminal(asset_id);
CREATE INDEX idx_terminal_cn ON terminal(connectivity_node_id);

-- Subnetwork (feeder, pressure zone, collection basin)
CREATE TABLE subnetwork (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    utility_type    VARCHAR(50) NOT NULL,
    subnetwork_type VARCHAR(50),
    source_asset_id UUID REFERENCES asset(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subnetwork_member (
    subnetwork_id   UUID NOT NULL REFERENCES subnetwork(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    PRIMARY KEY (subnetwork_id, asset_id)
);
```

## Work Order Management

```sql
CREATE TABLE work_order (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_number   VARCHAR(50) NOT NULL,
    parent_id           UUID REFERENCES work_order(id),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    asset_id            UUID REFERENCES asset(id),
    location_id         UUID REFERENCES location(id),

    -- Universal relational fields
    work_order_type     VARCHAR(50) NOT NULL,             -- 'corrective','preventive','condition_based','emergency','capital'
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    priority            INTEGER NOT NULL DEFAULT 3,
    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    reported_by         UUID REFERENCES app_user(id),
    assigned_to         UUID REFERENCES app_user(id),

    -- Scheduling
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,

    -- Cost tracking
    estimated_cost      NUMERIC(14,2),
    actual_cost         NUMERIC(14,2),

    -- JSONB for variable work order details
    tasks               JSONB NOT NULL DEFAULT '[]',
    -- Example tasks:
    -- [
    --   {"seq": 1, "description": "Isolate transformer", "status": "completed", "hours": 0.5},
    --   {"seq": 2, "description": "Replace bushing", "status": "in_progress", "hours": 3.0},
    --   {"seq": 3, "description": "Test and energise", "status": "pending", "hours": 1.0}
    -- ]

    labour              JSONB NOT NULL DEFAULT '[]',
    -- Example labour:
    -- [
    --   {"user_id": "uuid", "name": "Jane Smith", "start": "2026-04-01T08:00Z", "end": "2026-04-01T14:30Z", "hours": 6.5, "type": "regular"}
    -- ]

    materials           JSONB NOT NULL DEFAULT '[]',
    -- Example materials:
    -- [
    --   {"part_number": "BUSH-12470-A", "name": "12.47kV Bushing", "quantity_planned": 1, "quantity_used": 1, "unit_cost": 1200.00}
    -- ]

    failure_analysis    JSONB,
    -- Example failure_analysis:
    -- {
    --   "failure_class": "electrical",
    --   "failure_cause": "corrosion",
    --   "failure_remedy": "replacement",
    --   "failure_mode": "insulation_breakdown",
    --   "root_cause_notes": "Moisture ingress through cracked bushing seal"
    -- }

    custom_fields       JSONB NOT NULL DEFAULT '{}',     -- Organisation-specific custom fields

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, work_order_number)
);
CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_org ON work_order(organisation_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_type ON work_order(work_order_type);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_scheduled ON work_order(scheduled_start);
CREATE INDEX idx_wo_parent ON work_order(parent_id);
CREATE INDEX idx_wo_tasks ON work_order USING GIN(tasks jsonb_path_ops);

-- Preventive maintenance schedule
CREATE TABLE pm_schedule (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    name                VARCHAR(255) NOT NULL,
    asset_type_id       UUID REFERENCES asset_type(id),
    asset_id            UUID REFERENCES asset(id),
    work_order_type     VARCHAR(50) NOT NULL DEFAULT 'preventive',
    trigger_type        VARCHAR(50) NOT NULL,             -- 'time','meter','condition'
    trigger_config      JSONB NOT NULL,
    -- Example trigger_config for time-based:
    -- {"interval_days": 180, "lead_time_days": 14}
    -- Example trigger_config for meter-based:
    -- {"meter_type": "operating_hours", "threshold": 5000, "unit": "hours"}
    -- Example trigger_config for condition-based:
    -- {"condition_threshold": 40.0, "sensor_type": "vibration"}
    work_order_template JSONB,                           -- Template for auto-generated work orders
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_triggered      TIMESTAMPTZ,
    next_due            TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pm_org ON pm_schedule(organisation_id);
CREATE INDEX idx_pm_asset ON pm_schedule(asset_id);
CREATE INDEX idx_pm_next ON pm_schedule(next_due);
```

## Inspection & Condition Assessment

```sql
CREATE TABLE inspection (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisation(id),
    asset_id            UUID NOT NULL REFERENCES asset(id),
    work_order_id       UUID REFERENCES work_order(id),
    inspector_id        UUID NOT NULL REFERENCES app_user(id),
    inspection_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    template_name       VARCHAR(255),
    template_version    INTEGER,

    -- Computed scores
    overall_score       NUMERIC(5,2),
    overall_grade       VARCHAR(10),
    defect_count        INTEGER NOT NULL DEFAULT 0,
    critical_defects    INTEGER NOT NULL DEFAULT 0,

    status              VARCHAR(50) NOT NULL DEFAULT 'draft',
    geom                GEOMETRY(Point, 4326),

    -- All inspection responses stored as JSONB array
    responses           JSONB NOT NULL DEFAULT '[]',
    -- Example responses:
    -- [
    --   {
    --     "item": "oil_level",
    --     "category": "insulation",
    --     "question": "Oil level in sight glass",
    --     "response_type": "choice",
    --     "value": "normal",
    --     "score": 90,
    --     "weight": 1.5,
    --     "is_defect": false
    --   },
    --   {
    --     "item": "external_corrosion",
    --     "category": "structure",
    --     "question": "External corrosion observed",
    --     "response_type": "choice",
    --     "value": "moderate",
    --     "score": 45,
    --     "weight": 2.0,
    --     "is_defect": true,
    --     "severity": "minor",
    --     "notes": "Surface rust on base plate, no structural compromise"
    --   }
    -- ]

    -- Photos and attachments
    attachments         JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "file_name": "tx-4401-corrosion-01.jpg",
    --     "storage_path": "s3://bucket/inspections/2026-03/tx-4401-corrosion-01.jpg",
    --     "file_type": "image/jpeg",
    --     "file_size_bytes": 2450000,
    --     "captured_at": "2026-03-15T10:35:00Z",
    --     "ai_analysis": {
    --       "defects_detected": ["surface_corrosion"],
    --       "confidence": 0.87,
    --       "model": "defect-detect-v3"
    --     }
    --   }
    -- ]

    notes               TEXT,
    custom_fields       JSONB NOT NULL DEFAULT '{}',

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_insp_asset ON inspection(asset_id);
CREATE INDEX idx_insp_org ON inspection(organisation_id);
CREATE INDEX idx_insp_date ON inspection(inspection_date);
CREATE INDEX idx_insp_score ON inspection(overall_score);
CREATE INDEX idx_insp_geom ON inspection USING GIST(geom);
CREATE INDEX idx_insp_responses ON inspection USING GIN(responses jsonb_path_ops);
```

## IoT Sensors & Telemetry

```sql
CREATE TABLE sensor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    sensor_type     VARCHAR(100) NOT NULL,
    observable_property VARCHAR(100) NOT NULL,
    unit_of_measure VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "manufacturer": "Fluke",
    --   "model": "3561 FC",
    --   "serial_number": "FL-3561-44012",
    --   "sampling_interval_seconds": 300,
    --   "min_threshold": 0,
    --   "max_threshold": 95.0,
    --   "alert_severity": "warning",
    --   "installed_date": "2025-09-01",
    --   "calibration_due": "2026-09-01"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_reading_at TIMESTAMPTZ,
    last_value      NUMERIC(16,6),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_asset ON sensor(asset_id);
CREATE INDEX idx_sensor_type ON sensor(sensor_type);

-- Sensor observations (time-series, partitioned by month)
CREATE TABLE sensor_observation (
    sensor_id       UUID NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL,
    value           NUMERIC(16,6),
    quality         VARCHAR(20) DEFAULT 'good',
    metadata        JSONB                               -- Additional context per reading
) PARTITION BY RANGE (observed_at);
CREATE INDEX idx_obs_sensor_time ON sensor_observation(sensor_id, observed_at);

-- Sensor alerts
CREATE TABLE sensor_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_id       UUID NOT NULL REFERENCES sensor(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    observed_at     TIMESTAMPTZ NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    threshold_value NUMERIC(12,4),
    actual_value    NUMERIC(12,4),
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES app_user(id),
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_asset ON sensor_alert(asset_id);
CREATE INDEX idx_alert_unack ON sensor_alert(is_acknowledged) WHERE is_acknowledged = false;
```

## Inventory & Materials

```sql
CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    location_id     UUID REFERENCES location(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE material (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),   -- NULL = global catalogue
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    unit_of_issue   VARCHAR(50) NOT NULL,
    unit_cost       NUMERIC(12,2),
    properties      JSONB NOT NULL DEFAULT '{}',         -- Specs, compatible asset types, vendor info
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, part_number)
);

CREATE TABLE inventory_balance (
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    material_id     UUID NOT NULL REFERENCES material(id),
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 0,
    reorder_point   NUMERIC(10,2),
    reorder_quantity NUMERIC(10,2),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (storeroom_id, material_id)
);
```

## Compliance & Risk

```sql
CREATE TABLE compliance_obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    framework       VARCHAR(100) NOT NULL,
    standard_ref    VARCHAR(100) NOT NULL,
    obligation_text TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "frequency": "15_months",
    --   "evidence_requirements": ["asset_inventory", "categorization_worksheet"],
    --   "applicable_asset_types": ["substation", "control_center"],
    --   "responsible_role": "compliance_officer"
    -- }
    responsible_user UUID REFERENCES app_user(id),
    last_completed  TIMESTAMPTZ,
    next_due        TIMESTAMPTZ,
    status          VARCHAR(50) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compliance_org ON compliance_obligation(organisation_id);
CREATE INDEX idx_compliance_due ON compliance_obligation(next_due);
CREATE INDEX idx_compliance_framework ON compliance_obligation(framework);

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
```

## AI & Capital Planning

```sql
CREATE TABLE condition_prediction (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL REFERENCES asset(id),
    prediction_date     TIMESTAMPTZ NOT NULL DEFAULT now(),
    model_info          JSONB NOT NULL,
    -- Example model_info:
    -- {
    --   "model_name": "transformer_rul_v3",
    --   "model_version": "3.2.1",
    --   "remaining_useful_life_days": 1825,
    --   "failure_probability_1yr": 0.08,
    --   "failure_probability_5yr": 0.35,
    --   "confidence_score": 0.82,
    --   "input_features": {
    --     "age_years": 7,
    --     "condition_score": 72.5,
    --     "inspection_count": 5,
    --     "defect_count": 2,
    --     "sensor_alert_count": 3
    --   },
    --   "explanation": "Elevated failure risk due to moderate corrosion trend and two oil quality defects"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pred_asset ON condition_prediction(asset_id);
CREATE INDEX idx_pred_date ON condition_prediction(prediction_date);

CREATE TABLE capital_programme (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    total_budget    NUMERIC(16,2),
    status          VARCHAR(50) DEFAULT 'draft',
    config          JSONB NOT NULL DEFAULT '{}',         -- Budget allocation rules, approval workflow
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE capital_candidate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id    UUID NOT NULL REFERENCES capital_programme(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    priority_rank   INTEGER,
    estimated_cost  NUMERIC(14,2),
    risk_score      NUMERIC(5,2),
    planned_year    INTEGER,
    status          VARCHAR(50) DEFAULT 'candidate',
    justification   JSONB,                               -- Risk analysis, failure history, cost-benefit
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cap_prog ON capital_candidate(programme_id);
CREATE INDEX idx_cap_asset ON capital_candidate(asset_id);
```

## Documents & Audit

```sql
CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    title           VARCHAR(500),
    doc_type        VARCHAR(50) NOT NULL,
    file_name       VARCHAR(255) NOT NULL,
    storage_path    TEXT NOT NULL,
    file_size_bytes BIGINT,
    metadata        JSONB NOT NULL DEFAULT '{}',         -- EXIF, GPS, AI analysis results
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_doc_entity ON document(entity_type, entity_id);
CREATE INDEX idx_doc_org ON document(organisation_id);

-- Audit trail
CREATE TABLE audit_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                               -- {field: {old: ..., new: ...}}
    metadata        JSONB,                               -- IP, user agent, correlation ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

## Service Requests

```sql
CREATE TABLE service_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    request_number  VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    reporter        JSONB,                               -- Contact info (flexible format)
    -- Example reporter:
    -- {"name": "John Doe", "email": "john@example.com", "phone": "+1-555-0123"}
    geom            GEOMETRY(Point, 4326),
    priority        INTEGER DEFAULT 3,
    status          VARCHAR(50) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES app_user(id),
    work_order_id   UUID REFERENCES work_order(id),
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, request_number)
);
CREATE INDEX idx_sr_org ON service_request(organisation_id);
CREATE INDEX idx_sr_status ON service_request(status);
CREATE INDEX idx_sr_geom ON service_request USING GIST(geom);
```

---

## Example Queries

### Query transformer assets by JSONB property
```sql
SELECT a.asset_code, a.name, a.condition_score,
       a.properties->>'rated_power_kva' AS rated_power,
       a.properties->>'primary_voltage_kv' AS voltage
FROM asset a
JOIN asset_type at ON a.asset_type_id = at.id
WHERE at.category = 'transformer'
  AND (a.properties->>'rated_power_kva')::numeric > 500
  AND a.status = 'active'
ORDER BY a.condition_score ASC;
```

### Query assets with NERC CIP compliance due
```sql
SELECT a.asset_code, a.name,
       a.compliance->'nerc_cip'->>'bes_impact_rating' AS impact,
       a.compliance->'nerc_cip'->>'review_due_date' AS review_due
FROM asset a
WHERE a.compliance->'nerc_cip' IS NOT NULL
  AND (a.compliance->'nerc_cip'->>'review_due_date')::date <= CURRENT_DATE + INTERVAL '30 days'
ORDER BY (a.compliance->'nerc_cip'->>'review_due_date')::date;
```

### JSONB containment query for defect inspections
```sql
SELECT i.id, i.inspection_date, i.overall_score, a.asset_code
FROM inspection i
JOIN asset a ON i.asset_id = a.id
WHERE i.responses @> '[{"is_defect": true, "severity": "critical"}]'
ORDER BY i.inspection_date DESC;
```

### Cross-utility asset search with spatial filter
```sql
SELECT a.asset_code, a.name, at.utility_type, at.category,
       a.condition_score, ST_AsGeoJSON(a.geom) AS geojson
FROM asset a
JOIN asset_type at ON a.asset_type_id = at.id
WHERE ST_DWithin(
    a.geom,
    ST_SetSRID(ST_MakePoint(-84.388, 33.749), 4326),
    0.01  -- ~1km radius
)
AND a.status = 'active'
ORDER BY a.geom <-> ST_SetSRID(ST_MakePoint(-84.388, 33.749), 4326);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Organisation | 2 | Organisations with JSONB settings; users with JSONB permissions |
| Location & Spatial | 1 | Locations with JSONB address |
| Asset Register | 2 | Assets with JSONB properties/compliance; asset types with JSONB schema |
| Network Topology | 3 | Connectivity nodes, terminals, subnetworks |
| Work Order Management | 2 | Work orders with JSONB tasks/materials/labour; PM schedules with JSONB config |
| Inspection | 1 | Inspections with JSONB responses/attachments |
| IoT & Telemetry | 3 | Sensors with JSONB config; observations (partitioned); alerts |
| Inventory | 3 | Storerooms, materials with JSONB properties, balances |
| Compliance & Risk | 2 | Obligations with JSONB config; climate risk zones |
| AI & Capital | 3 | Predictions with JSONB model info; programmes; candidates |
| Documents & Audit | 2 | Documents with JSONB metadata; audit log (partitioned) |
| Service Requests | 1 | Requests with JSONB reporter/custom fields |
| **Total** | **25** | Significantly fewer tables than normalised model (53) |

---

## Key Design Decisions

1. **JSONB properties column on assets instead of per-type tables** -- A single `properties` JSONB column on the asset table replaces the 7+ equipment detail tables (transformer_info, pipe_info, valve_info, etc.) from the normalised model. The asset_type table holds a JSON Schema (`properties_schema`) that defines the expected structure for each asset type, enabling application-level validation without database-level column constraints.

2. **JSONB for multi-jurisdiction compliance fields** -- Rather than a fixed set of compliance columns (which would need DDL changes for each new regulation), a `compliance` JSONB column absorbs jurisdiction-specific regulatory fields. NERC CIP, ISO 55000, Ofgem, and AWWA each have their own key namespace within the JSONB object.

3. **Work order tasks, labour, and materials as JSONB arrays** -- Instead of three separate relational tables (work_order_task, work_order_labour, work_order_material), these are stored as JSONB arrays on the work_order table. This reduces joins for the most common query (display a complete work order) from 4 tables to 1. The trade-off is that you cannot efficiently query "total hours by user across all work orders" without JSONB unnesting.

4. **Inspection responses as JSONB array** -- Inspection templates define the question structure, and responses are stored as a JSONB array on the inspection record. This allows different inspection templates to have completely different question sets without requiring DDL changes. GIN indexes enable containment queries for specific defect types.

5. **Linear asset sections in JSONB** -- Section-level attributes for pipelines and cables are stored in the `linear_detail` JSONB column as an array of section objects. This keeps the table count low and avoids the need for a separate linear_asset_section table, at the cost of making section-level spatial queries more complex.

6. **Organisation settings as JSONB** -- Per-tenant configuration (utility types, custom fields, compliance frameworks, fiscal year) lives in the `settings` JSONB column on the organisation table. This supports multi-tenant SaaS without per-tenant schema customisation.

7. **GIN indexes for JSONB performance** -- All major JSONB columns have GIN indexes with `jsonb_path_ops` to support containment queries (`@>`) efficiently. For frequently queried specific properties (like `rated_power_kva`), expression indexes on extracted values provide column-level query performance.

8. **Relational core for structural integrity** -- Despite the heavy JSONB usage, the schema maintains relational foreign keys for the structural relationships that matter most: asset hierarchy (parent_id), asset-to-location, work-order-to-asset, inspection-to-asset, terminal-to-connectivity-node. These relationships are too important for referential integrity to delegate to JSONB.
