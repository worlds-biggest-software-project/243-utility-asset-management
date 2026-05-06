# Utility Asset Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for utility infrastructure GIS, condition assessment, and replacement planning.

Utility Asset Management is a candidate project for an open alternative to the proprietary GIS + EAM stacks that dominate electric, gas, and water utilities. It targets utility CIOs, asset managers, GIS analysts, and municipal public works teams who need spatial asset registers, work order management, and risk-based replacement planning without enterprise-scale licensing and consulting costs.

---

## Why Utility Asset Management?

- Incumbent platforms (IBM Maximo, Esri ArcGIS Utility Network, Trimble Unity, SAP S/4HANA, Hexagon HxGN EAM) carry custom enterprise pricing — typically hundreds of thousands to millions annually, with implementation costs adding 50–100% on top of licences.
- Esri ArcGIS uses a proprietary geodatabase format that creates lock-in; migrations from Geometric Network to Utility Network are disruptive and multi-year for large organisations.
- Mid-market options like eMaint ($69/user/month) lack GIS, linear asset management, and compliance support for regulated utilities (NERC CIP, ISO 55000, AWWA).
- AI and predictive maintenance features outside IBM Maximo APM are nascent; capital replacement planning is still largely spreadsheet-based across the sector.
- No credible open-source stack exists that combines spatial asset management, network tracing, and condition-based work order workflows on open standards (PostGIS, OGC, OpenAPI, IEC CIM).

---

## Key Features

### Spatial Asset Register & Network Model

- Asset register with GIS spatial location (PostGIS / GeoJSON), hierarchy, attributes, and full history
- Linear asset management for pipelines, cables, and roads with section-level attributes
- Network tracing and connectivity analysis (upstream/downstream, isolation, shortest path) for electric, gas, and water topologies
- Map-based interface (OpenLayers or Leaflet) for asset search, selection, and work order creation

### Work Order & Inspection Management

- Corrective, preventive, and condition-based work orders with scheduling and cost tracking
- Configurable inspection workflow engine with condition scoring and automatic risk ranking
- Mobile field crew app with offline-capable work order execution and inspection capture
- Role-based access control with audit trail

### AI-Assisted Planning

- Remaining-useful-life scoring derived from inspection history, failure records, and sensor data
- Capital replacement prioritisation dashboard ranking assets by risk-weighted replacement cost
- Natural language asset query interface (LLM-backed) over spatial and maintenance data
- Computer vision defect detection on drone and inspection imagery (corrosion, vegetation, structural faults)

### Telemetry, Risk & Compliance

- IoT / sensor telemetry ingestion with threshold-based automatic work order creation
- Climate risk overlay intersecting flood, wildfire, and extreme-weather exposure with the asset network
- Compliance report templates for ISO 55000, NERC CIP, AWWA, and GDPR audit requirements
- Multi-utility scenario planning for capital programmes under budget constraints

---

## AI-Native Advantage

AI shifts utility asset management from schedule-based maintenance and spreadsheet-driven capital planning to probabilistic, data-fused decision-making. The platform fuses inspection records, sensor telemetry, and historical failure data to produce remaining-useful-life scores per asset; ranks capital replacements by risk-weighted cost and service impact; analyses drone and ground-level imagery for defects without manual review; and lets field engineers query the asset database in natural language (for example, "which water mains over 60 years old have had two or more main breaks in the past five years"). Climate risk modelling automatically flags high-risk segments for hardening investment.

---

## Tech Stack & Deployment

The reference architecture targets open standards throughout: PostGIS for spatial storage, OGC services (WFS, WMS, GeoPackage) for spatial interchange, IEC CIM (61968/61970) for electric utility data modelling, and an OpenAPI 3 REST surface for SCADA, sensor, and ERP integration. Map UX is delivered through OpenLayers or Leaflet, with an offline-capable mobile client for field crews. The project is intended to run self-hosted or in cloud deployments, deliberately avoiding proprietary geodatabase formats and trade-secret-protected workflows.

---

## Market Context

The global utility asset management market was valued at USD 5.51 billion in 2025 and is projected to reach USD 6.05 billion in 2026, growing at a 9.9% CAGR toward USD 8.78 billion by 2030 (Research and Markets, 2026). The broader GIS software market is expected to reach USD 14 billion by 2026. Buyers include CIOs and asset management directors at electric, gas, and water utilities, GIS analysts and network planners, municipal public works directors, and infrastructure investment decision-makers at regulated utilities. The space is dominated by Esri, IBM, SAP, Trimble, and Hexagon; no major recent VC-backed entrants have emerged in the pure utility GIS/EAM segment.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
