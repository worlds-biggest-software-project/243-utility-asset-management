# Utility Asset Management — Feature & Functionality Survey

> Candidate #243 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| IBM Maximo Application Suite | Enterprise EAM | Commercial SaaS/on-prem | https://www.ibm.com/products/maximo |
| Esri ArcGIS Utility Network | GIS-centric EAM | Commercial SaaS/on-prem | https://developers.arcgis.com/rest/services-reference/overview-of-utility-network-services.htm |
| Trimble Unity Maintain (Cityworks) | GIS-centric EAM | Commercial SaaS | https://assetlifecycle.trimble.com/en/products/software/unity-maintain |
| SAP S/4HANA Asset Management | ERP-integrated EAM | Commercial SaaS/on-prem | https://www.sap.com/products/erp/s4hana/features/asset-management.html |
| Hexagon HxGN EAM | Enterprise EAM | Commercial SaaS/on-prem | https://hexagon.com/products/enterprise-asset-management |
| Infor EAM | Enterprise EAM | Commercial SaaS | https://www3.technologyevaluation.com/solutions/16945/infor-eam |
| eMaint CMMS (Fluke) | CMMS / Condition Monitoring | Commercial SaaS | https://www.emaint.com/cmms/emaint-cmms-software/ |
| OpenGov Enterprise Asset Management | Government/Utility EAM | Commercial SaaS | https://opengov.com/products/asset-management/ |

---

## Feature Analysis by Solution

### IBM Maximo Application Suite

**Core features**
- Centralised asset register covering equipment, vehicles, facilities, pipelines, linear assets, and network infrastructure
- Work order management: corrective, preventive, emergency, and condition-based work orders with scheduling and cost tracking
- Preventive and predictive maintenance scheduling driven by time, meter readings, or condition thresholds
- Asset Performance Management (APM) module: AI-driven anomaly detection, failure mode analysis, and remaining-useful-life modelling
- Mobile access for field technicians: work order updates, asset inspection, and barcode/QR scanning
- Inventory and MRO parts management with automatic reorder triggers
- Integration framework: REST API, OData, OSLC, and MQ messaging for ERP, SCADA, and GIS connections
- Advanced reporting and configurable dashboards; compliance reporting for regulated industries

**Differentiating features**
- Maximo Assistant (gen-AI chatbot): natural language queries against asset and work order data ("which work orders are missing job plans?")
- Deepest IoT integration of any EAM platform: connects to sensors via Maximo Monitor for real-time asset telemetry
- Industry-specific configuration packages for utilities, oil & gas, transportation, and government
- Asset investment optimisation module for capital programme planning

**UX patterns**
- Role-based workcentres with a task-list-first interface
- Embedded gen-AI assistant panel accessible without leaving context
- Progressive disclosure: summary cards expand to full asset history, linked work orders, and sensor dashboards
- Complex configuration typically requires specialist consultants; self-service is limited for non-technical users

**Integration points**
- REST API (JSON/XML) and OSLC; Node.js and Java SDKs published on GitHub
- Native connectors to SAP, Oracle ERP, Esri ArcGIS, OSIsoft PI, SCADA systems
- IoT platform connectors via IBM Watson IoT and third-party MQTT brokers
- IBM API Hub catalogue for MAS REST APIs

**Known gaps**
- Reports require significant consultant effort; out-of-box dashboards are basic
- Scheduling module needs improvement for daily/weekly resource planning
- Very high TCO; licensing perceived as expensive relative to feature utilisation
- UI described as "not intuitive" and "basic" by reviewers; onboarding steep
- Version fragmentation: many utilities still on legacy Maximo 7.x

**Licence / IP notes**
- Proprietary commercial licence (IBM); no open-source components
- No known patents on specific features relevant to this project

---

### Esri ArcGIS Utility Network

**Core features**
- GIS-native network topology model for electric, gas, water, wastewater, and telecom networks
- Asset register embedded within the spatial network: attributes, relationships, and connectivity defined in a geodatabase
- Network tracing and analysis: upstream/downstream trace, isolation trace, connected trace, and shortest path
- Subnetwork management: automatic identification, update, and export of network segments
- Quality assurance/quality control (QA/QC) built into every edit; topology validation on every change
- Offline editing for field crews; sync with central geodatabase
- Field-to-office workflows via ArcGIS Field Maps and Collector

**Differentiating features**
- Geometric Network (legacy) to Utility Network migration — industry-standard migration path for all existing Esri utility customers
- Telecom Domain support (Spring 2026 release): fibre grouping, circuit management, tracing
- Utility and Pipeline Data Model (UPDM) 2026: standardised schema for multi-utility organisations
- Network diagram management for schematic views of electrical networks independent of spatial layout

**UX patterns**
- GIS-first: all asset interactions occur within a map context
- ArcGIS Pro desktop for data management and network editing; ArcGIS Online / Enterprise portal for web access
- Role-specific apps: field workers use Field Maps; engineers use ArcGIS Pro; managers use dashboards in ArcGIS Dashboards
- High technical depth — network rules, terminal configurations, and domain assignments require GIS specialist configuration

**Integration points**
- REST API: Utility Network service endpoint (`/UtilityNetworkServer`) for tracing, topology validation, subnetwork management
- ArcGIS Maps SDK for JavaScript (UtilityNetwork class) for web app development
- ArcGIS Pro 3.x Python API (arcpy) for scripting and automation
- Native connectors to Maximo, Cityworks, and SAP via Esri partner ecosystem
- OGC-compliant WFS, WMS, WCS, and WMTS services from feature services

**Known gaps**
- Performance degrades with very large datasets (100M+ features); cloud loading latency reported
- Interface complexity is high for non-GIS users; limited self-service configuration
- Expensive licensing; ArcGIS Utility Network user type extension required on top of base ArcGIS
- Migration from Geometric Network is disruptive and multi-year in large organisations
- Reporting and financial analysis capabilities are weak compared to dedicated EAM platforms

**Licence / IP notes**
- Proprietary commercial licence (Esri); proprietary geodatabase format
- ArcGIS APIs include open developer documentation but runtime licensing is commercial

---

### Trimble Unity Maintain (Cityworks)

**Core features**
- GIS-centric work order management for asset maintenance, construction, permits, and inspections
- Asset lifecycle management from construction permit through operations, maintenance, and replacement
- Inspection programme management: structured condition observation, test outcomes, and condition scoring
- Service request intake and public-facing portals for reporting infrastructure issues
- Project management module for capital construction tracking
- Mobile app for field crews: work order assignment, inspection forms, GPS asset capture
- Integration with ArcGIS Utility Network as the GIS backbone

**Differentiating features**
- Purpose-built for local government and public utilities (water, wastewater, electric, stormwater, streets)
- Service request and permit tracking in one platform alongside maintenance work orders
- Condition scoring engine that aggregates inspection results into asset risk scores for replacement prioritisation
- Trimble App Xchange (integration marketplace) with partner connectors

**UX patterns**
- Map-centric: assets are always located on a GIS map; work orders are created by clicking assets on the map
- Wizard-based inspection form builder for creating custom inspection templates without code
- Role-based dashboards for supervisors, field crews, and managers
- Moderate technical complexity; configuration is more accessible than Maximo but still requires training

**Integration points**
- REST API documented via Trimble Developer portal (developer.trimble.com)
- FME (Feature Manipulation Engine) connectors for ETL with GIS and external databases
- ArcGIS Enterprise tight integration; ArcGIS feature services as authoritative data source
- Trimble App Xchange OpenAPI-documented partner connectors
- SAP, Salesforce, and financial system connectors available via professional services

**Known gaps**
- Less capable financial planning and capital investment optimisation than Maximo or SAP
- AI and predictive maintenance features are nascent compared to IBM Maximo APM
- Primarily North American focus; less adoption in Europe and Asia-Pacific
- Implementation timelines are long (12–24 months typical)

**Licence / IP notes**
- Proprietary commercial licence (Trimble Inc.)
- Cityworks brand being migrated to Trimble Unity; no open-source release planned

---

### SAP S/4HANA Asset Management

**Core features**
- Equipment and functional location hierarchy for entire utility asset network
- Preventive and predictive maintenance order management, integrated with plant operations
- Linear asset management: model pipelines, cables, and roads with section-level attributes and defect recording
- Equipment calibration workflows for instruments, meters, and sensors
- Integration with SAP ERP financials: maintenance costs, capital expenditure tracking, depreciation
- Mobility via SAP Asset Manager mobile app for field crew work order execution
- Integration with SAP Asset Performance Management (APM) for condition-based maintenance and RCM/FMEA

**Differentiating features**
- Native ERP integration: asset costs, procurement, and financials in one ledger — no data duplication
- Linear Data in Task Lists (S/4HANA 2022+): section-level defect records automatically copied to work orders
- 2025 enhancement: APM recommendations and indicators surfaced directly inside the Manage Maintenance Orders app
- Equipment-centric compliance tracking for OSHA, NERC CIP, and safety regulations built into work order workflow

**UX patterns**
- Fiori-based tile UI; role-based launchpads provide task-oriented entry points
- Side-panel APM recommendations in Manage Maintenance Orders avoid context switching
- Complex configuration; SAP Basis and PM module expertise required
- GIS capabilities are limited out of the box; Esri or other GIS integration required via middleware

**Integration points**
- OData V2 and V4 REST APIs; SAP Business Accelerator Hub (api.sap.com) lists 2700+ APIs
- BTP (Business Technology Platform) for integration middleware and extension development
- Standard connectors to Esri ArcGIS, SCADA/PI historians via SAP Integration Suite
- SOAP web services for legacy integrations

**Known gaps**
- GIS and spatial asset management are weak without supplementary Esri integration
- Implementation complexity and SAP expertise requirements create very high onboarding barriers
- Per-user licensing model is expensive at utility scale
- Less utility-specific out-of-box configuration than Trimble or Hexagon

**Licence / IP notes**
- Proprietary commercial licence (SAP SE)
- No open-source components; OData standard (OASIS) is open

---

### Hexagon HxGN EAM

**Core features**
- Full asset lifecycle management: acquisition, operation, maintenance, and disposal
- Work order management with preventive, corrective, and condition-based maintenance
- Inspection management with checklists and condition scoring
- Materials and spare parts inventory management
- Linear asset management for pipelines, cables, and roads
- Compliance tracking for NERC CIP, ISO 55000, 21 CFR Part 11
- GIS integration with Esri ArcGIS: search, locate, and maintain assets from a map
- Energy performance monitoring: tracks WAGES (water, air, gas, electricity, steam)

**Differentiating features**
- HxGN EAM GIS module: map-driven asset search and work order creation deeply integrated with Esri
- AI-powered predictive maintenance with IIoT sensor integration (2025 IIoT integration module)
- HxGN EAM Databridge Pro: Apache NiFi-based middleware for complex data pipeline management
- HxGN EAM Python Framework (2023): Python API for automation and custom integrations
- OpenAPI 3 compliant REST web services for all business components

**UX patterns**
- Web-based, responsive UI accessible on desktop, tablet, and mobile (iOS and Android)
- Asset hierarchy browser with drill-down to individual equipment and components
- Configurable dashboards for different roles; safety and compliance dashboards prominent
- Configuration requires HxGN EAM-certified consultants for complex workflows

**Integration points**
- REST web services (OpenAPI 3 compliant); SOAP also supported for legacy
- API key-based authentication; Swagger documentation published by Hexagon
- Python SDK (HxGN EAM Python Framework) for scripting
- Esri ArcGIS integration module
- IIoT sensor integration for real-time condition monitoring

**Known gaps**
- Less well known than IBM Maximo or SAP; smaller partner ecosystem
- Implementation timelines comparable to other enterprise EAM platforms (12–18 months)
- Advanced analytics and AI features lag behind IBM Maximo APM for complex fault prediction
- Pricing is custom enterprise; not accessible for small utilities

**Licence / IP notes**
- Proprietary commercial licence (Hexagon AB)
- OpenAPI 3 specification used for REST API (open standard); runtime is proprietary

---

### Infor EAM

**Core features**
- Asset register across equipment, vehicles, facilities, and linear infrastructure
- Work management: corrective, preventive, and predictive work orders
- Inspection management with compliance tracking
- Materials and MRO inventory management
- Procurement management integration
- Budget management and capital project tracking
- Safety and case management for incident reporting
- Fleet management module
- Energy performance monitoring (WAGES)
- GIS integration via ESRI add-on module

**Differentiating features**
- Pre-built industry configurations for utilities, oil & gas, transportation, healthcare, and government
- Reliability Planning and Analysis (RPA) module for FMEA and RCM analysis
- Built-in calibration management for instrumented assets (meters, sensors)
- iProcure web-based procurement portal for MRO supply chain management

**UX patterns**
- Web-based role-specific workcentres; mobile app for field access
- Industry configuration templates reduce initial setup effort compared to greenfield EAM implementations
- Advanced reporting requires third-party BI tools; native reporting is basic

**Integration points**
- REST API and standard integration connectors; FME and third-party integration platforms supported
- Esri ArcGIS GIS add-on module for spatial asset management
- ERP connectors to SAP, Oracle, and Infor financial modules

**Known gaps**
- GIS integration is an add-on, not native; less map-centric than Esri or Trimble solutions
- Smaller market presence and partner ecosystem than Maximo or SAP
- AI and predictive analytics features are less mature than IBM Maximo APM
- Mobile experience less polished than newer competitors

**Licence / IP notes**
- Proprietary commercial licence (Infor / Koch Industries)
- No open-source components

---

### eMaint CMMS (Fluke)

**Core features**
- Work order management: corrective, preventive, and emergency work orders
- Asset register with hierarchy and history tracking
- Preventive maintenance scheduling by time, meter, or event triggers
- Inspection planning and checklist-based inspection execution
- Spare parts and MRO inventory management
- Condition monitoring module: threshold-based alerting from Fluke IIoT wireless sensors
- Mobile app for field technicians
- SCADA, PLC, RTU, BMS, and MES data integration for production monitoring

**Differentiating features**
- Tight integration with Fluke wireless vibration and thermal sensors (first-party hardware + software stack)
- eMaint AI: machine learning on vibration FFT data identifies rotating machinery faults (misalignment, imbalance, looseness, bearing wear) and recommends maintenance actions
- Accessible entry-level pricing ($69/user/month) targeting mid-market utilities and industrial operators
- Condition Monitoring module provides automated work order creation when sensor thresholds are exceeded

**UX patterns**
- Clean, modern SaaS interface with lower learning curve than enterprise EAM platforms
- Sensor dashboard shows live asset health; anomaly alerts link directly to work order creation
- Configuration is largely self-service without specialist consultants

**Integration points**
- REST API for third-party integrations
- Native Fluke IIoT sensor connectivity (wireless vibration, temperature, and ultrasound sensors)
- SCADA/PLC data integration for production context
- Limited GIS integration

**Known gaps**
- GIS and spatial asset management are absent — not suitable for network infrastructure mapping
- Linear asset management not supported
- Less suitable for enterprise-scale utility operations (network planning, capital programmes)
- Compliance reporting for regulated utilities (NERC CIP, ISO 55000) is limited

**Licence / IP notes**
- Proprietary commercial licence (Fluke / Fortive Corporation)
- No open-source components

---

### OpenGov Enterprise Asset Management

**Core features**
- Asset register for government and utility infrastructure: roads, water, wastewater, parks, facilities
- Work order management with resource tracking (labour, equipment, materials)
- Mobile app for field crews: work orders, asset updates, GPS capture
- GIS integration for spatial asset mapping and condition tracking
- Capital improvement project planning and tracking
- Public-facing service request portal
- Reporting and dashboards for regulatory and budget reporting

**Differentiating features**
- Purpose-built for public agencies and government utilities (2000+ US public agencies)
- Integrated permitting and licensing alongside asset management
- Built-in GIS with ArcGIS partnership (Esri Authorised Reseller)
- Accessible SaaS pricing for small and mid-size municipalities

**UX patterns**
- Map-centric interface; assets are managed from a GIS map
- Role-based dashboards for operations, finance, and executive audiences
- Designed for non-technical government users; lower configuration complexity than enterprise platforms

**Integration points**
- ArcGIS integration (Esri Authorised Partner)
- Financial system connectors for government ERP platforms
- API access for data exchange with third-party systems

**Known gaps**
- Less suitable for large investor-owned utilities with complex regulatory and SCADA requirements
- AI and predictive maintenance features are limited
- Feature depth in work order scheduling and materials management is less than Maximo or SAP

**Licence / IP notes**
- Proprietary commercial SaaS licence (OpenGov Inc.)
- Formerly Cartegraph OMS; no open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Centralised asset register with hierarchy, attributes, and history
- Work order management (corrective, preventive, condition-based)
- Mobile field access (work order execution, inspection forms, GPS capture)
- Inspection and condition scoring workflows
- Spare parts and inventory management
- Basic reporting and dashboards
- Integration framework (REST API minimum)
- Role-based access control

### Differentiating Features
- GIS-native map interface for asset location and network topology
- Network tracing and connectivity analysis (upstream/downstream, isolation)
- Linear asset management with section-level attribute recording
- Real-time IoT/sensor integration for condition monitoring
- AI-powered anomaly detection and remaining-useful-life prediction
- Capital investment optimisation and multi-year programme planning
- Compliance tracking for domain-specific standards (NERC CIP, ISO 55000, AWWA)
- Embedded gen-AI natural language querying of asset and work order data
- Drone and imaging integration for remote condition assessment

### Underserved Areas / Opportunities
- Natural language querying of spatial asset data without GIS expertise ("show me all 60-year-old water mains with two or more breaks")
- Automated probabilistic remaining-useful-life scoring fusing inspection records, failure history, and sensor data
- Climate risk overlay: flood, wildfire, and extreme weather risk mapped against asset network
- Computer vision analysis of inspection imagery for defect detection without manual review
- AI-driven capital replacement prioritisation that replaces spreadsheet-based manual ranking
- Accessible open-source alternative to the proprietary GIS + EAM stack (QGIS + PostGIS based)
- Field inspection UX optimised for low-connectivity environments without full enterprise EAM overhead
- Institutional knowledge capture from retiring engineers into structured asset risk profiles

### AI-Augmentation Candidates
- Failure prediction: replacing schedule-based maintenance with ML models trained on asset history
- Automated work order creation from sensor threshold alerts with asset history attached
- Image-based defect detection on drone/photo inspection imagery
- Natural language interface for asset data queries by non-GIS field staff
- Automated subnetwork risk ranking for capital programme prioritisation
- Predictive parts demand forecasting for MRO inventory optimisation

---

## Legal & IP Summary

All solutions analysed are proprietary commercial software with no open-source components. Esri ArcGIS uses a proprietary geodatabase format that creates lock-in, though it publishes open developer documentation and REST APIs. Hexagon HxGN EAM REST services are documented using the OpenAPI 3 open standard (runtime is proprietary). No patents on specific features relevant to this project were identified in publicly available material; however, IBM holds numerous AI and asset management related patents under the Maximo brand. An open-source implementation should rely solely on open standards (IEC CIM, OGC APIs, OpenAPI, GeoJSON, GeoPackage) and avoid replicating any proprietary workflow or data model that could be subject to trade secret protection. No copyright concerns were identified with the publicly documented feature sets referenced in this survey.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Asset register with GIS spatial location (PostGIS or GeoJSON-based), hierarchy, attributes, and full history
- Work order management: corrective, preventive, and condition-based with scheduling and cost tracking
- Inspection workflow engine with configurable forms, condition scoring, and automatic risk ranking
- Map-based interface (OpenLayers or Leaflet) for asset search, selection, and work order creation
- REST API (OpenAPI 3 specification) for integration with SCADA, sensors, and ERP systems
- Role-based access control and audit trail

**Should-have (v1.1)**
- Network tracing and connectivity analysis for electric, gas, and water network topologies
- AI-assisted remaining-useful-life scoring from inspection history and failure records
- Mobile app for field crew work order execution and inspection capture (offline-capable)
- Natural language asset query interface (LLM-backed, querying spatial and maintenance data)
- Capital replacement prioritisation dashboard ranking assets by risk-weighted replacement cost

**Nice-to-have (backlog)**
- IoT / sensor telemetry ingestion with threshold-based automatic work order creation
- Computer vision defect detection on inspection images (corrosion, vegetation, structural faults)
- Climate risk overlay: flood/wildfire/extreme weather risk intersected with asset network
- Compliance report templates for ISO 55000, NERC CIP, AWWA, and GDPR audit requirements
- Multi-utility scenario planning for capital programme optimisation under budget constraints
