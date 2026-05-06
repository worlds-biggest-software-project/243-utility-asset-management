# Standards & API Reference

> Project: Utility Asset Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 55000:2024 — Asset Management: Vocabulary, Overview and Principles**
- URL: https://www.iso.org/standard/83053.html
- Defines the vocabulary, overview, and foundational principles for asset management systems. The 2024 edition (revised from 2014) adds stronger requirements on decision-making, value realisation, risk management, and data/knowledge management. Required alignment for any utility asset management platform targeting ISO certification.

**ISO 55001:2024 — Asset Management System: Requirements**
- URL: https://www.iso.org/standard/83054.html
- Specifies requirements for an asset management system (AMS) covering planning, support, operation, performance evaluation, and improvement. This is the certifiable standard that utility operators use to demonstrate mature asset management practice; any software platform should support generating the evidence artefacts required for certification.

**ISO 55002:2018 — Asset Management: Guidelines for the Application of ISO 55001**
- URL: https://www.iso.org/standard/65061.html
- Provides guidance on how to implement ISO 55001 requirements in practice. Useful as a reference for the feature design of lifecycle cost analysis, strategic asset management plans, and evidence documentation within the platform.

---

### W3C & IETF Standards

**OGC WFS (Web Feature Service) — OGC Standard**
- URL: https://www.ogc.org/standards/wfs/
- Defines a protocol for querying and editing geographic vector features over HTTP. The predecessor to OGC API Features; still widely deployed in utility GIS infrastructure. Any platform consuming or serving GIS data should support WFS for backward compatibility.

**OGC API Features (successor to WFS)**
- URL: https://ogcapi.ogc.org/features/
- Modern RESTful replacement for WFS, targeting both GIS and non-GIS web developers. Designed to make geospatial data accessible as standard web resources (JSON/GeoJSON over HTTPS). Preferred standard for new API design in utility GIS platforms.

**OGC WMS (Web Map Service)**
- URL: https://www.ogc.org/standards/wms/
- Standard for serving pre-rendered map tiles from a GIS server. Used for background map layers in utility asset management web applications. Interoperable with Esri, QGIS, GeoServer, and MapServer.

**OGC GeoPackage**
- URL: https://www.ogc.org/standards/geopackage/
- SQLite-based open format for packaging vector features, tile rasters, and metadata in a single portable file. The preferred offline field data format; enables field crews to work disconnected and sync back to the central asset register.

**OGC GeoSPARQL 1.1 (2024)**
- URL: https://www.ogc.org/standards/geosparql/
- Standard for representing and querying geospatial data in RDF/Linked Data environments. Relevant for AI-native platforms that use knowledge graphs to model asset relationships and network topology. The 2024 edition adds SHACL validation support.

**RFC 7946 — GeoJSON**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- IETF standard format for encoding geographic data structures (Point, LineString, Polygon, Feature, FeatureCollection) in JSON. The universal interchange format for utility network asset geometries in REST APIs. All platform endpoints exchanging spatial data should use GeoJSON.

**RFC 7231 / HTTP/1.1 (IETF)**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the semantics of HTTP methods, status codes, and headers used by all REST APIs in this domain. Compliance with HTTP semantics is a prerequisite for any standards-conformant API implementation.

---

### Data Model & API Specifications

**IEC 61970 — Common Information Model (CIM): Energy Management System APIs**
- URL: https://webstore.ansi.org/industry/smartgrid/iec-61968-61970
- Defines the abstract data model (CIM) for energy management system (EMS) applications. IEC 61970-301 specifies core CIM packages covering network topology, equipment classes, measurements, and connectivity — the vendor-neutral data model for electric transmission and distribution assets. Any electric utility asset management platform targeting interoperability with SCADA and EMS should align its asset data model to CIM.

**IEC 61968 — Common Information Model (CIM): Distribution Management System APIs**
- URL: https://webstore.ansi.org/industry/smartgrid/iec-61968-61970
- Extends the CIM (IEC 61970) for distribution management system applications, covering asset management, work management, meter reading and control, GIS location, and customer information. IEC 61968-11 defines the message payload formats used for system integration. Core reference for the asset data model in electric distribution utility platforms.

**Esri Utility and Pipeline Data Model (UPDM) 2026**
- URL: https://community.esri.com/t5/gas-and-pipeline-blog/utility-and-pipeline-data-model-2026-is-released/ba-p/1693885
- Esri's canonical geodatabase schema for multi-utility organisations (electric, gas, water, wastewater, telecom). While proprietary to Esri's geodatabase format, it is widely adopted and its entity and attribute definitions are a practical reference for designing a vendor-neutral utility asset data model.

**OpenAPI Specification 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The world standard for describing RESTful HTTP APIs in a machine-readable format. All platform REST endpoints should be described in an OpenAPI 3.x document to enable automated SDK generation, client validation, and API gateway integration. Hexagon HxGN EAM REST services are already documented in OpenAPI 3 format; this should be the baseline for any new utility asset management API.

**GeoJSON Schema (RFC 7946 + JSON Schema)**
- URL: https://geojson.org/schema/
- JSON Schema definitions for GeoJSON feature types. Useful for validating spatial payloads in API requests and responses when building or consuming utility asset geometry data.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 | https://openid.net/connect/
- Industry-standard authorisation (OAuth 2.0) and authentication (OIDC) protocols. All SaaS asset management platforms (Maximo, Esri, Trimble) use OAuth 2.0 / OIDC for API and user authentication. Required for any platform integrating with enterprise identity providers (Active Directory, Okta, Azure AD).

**NERC CIP Standards (CIP-002 through CIP-015)**
- URL: https://www.nerc.com/standards/reliability-standards/cip
- North American reliability standards for cybersecurity of Bulk Electric System (BES) Cyber Assets. The 2025 updates (CIP-003-9, CIP-005-7, CIP-010-4, CIP-013-2) expand requirements to substations and distributed energy resources. CIP-015-1 (FERC Order 907, June 2025) mandates internal network security monitoring for high/medium impact BES cyber systems. Asset management platforms storing BES asset inventories must support CIP compliance evidence generation and audit trails.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- OWASP's reference list of the most critical API security risks (broken object level authorisation, excessive data exposure, lack of resource/rate limiting, etc.). Relevant for API design and security review of utility asset management REST endpoints that expose critical infrastructure data.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- US federal framework for managing cybersecurity risk across Identify, Protect, Detect, Respond, and Recover functions. Many regulated US utilities map their cyber asset management practices to NIST CSF; a platform supporting compliance evidence export against NIST CSF categories is commercially valuable.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- International standard for information security management. Utility asset management platforms storing critical infrastructure data must demonstrate alignment with ISO 27001 controls for data access, audit logging, and incident response.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant to AI-native utility asset management tooling. A platform that exposes its asset register and work order data via an MCP server would allow LLM-powered agents to answer natural language questions, generate replacement schedules, and create work orders without custom API integration.

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI assistants to data sources and tools. An MCP server exposing asset data, work order CRUD, network trace operations, and inspection records would enable LLM agents (Claude, GPT-4, etc.) to act as intelligent asset management assistants. This is a key differentiator for an AI-native open-source implementation.

---

## Similar Products — Developer Documentation & APIs

### IBM Maximo Application Suite

- **Description:** Enterprise EAM platform with IoT integration, AI-powered asset performance management, and natural language querying via Maximo Assistant. Industry coverage includes utilities, oil & gas, and transportation.
- **API Documentation:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/
- **SDKs/Libraries:** Node.js: https://github.com/ibm-maximo-dev/maximo-nodejs-rest-client · Java: https://github.com/ibm-maximo-dev/maximo-java-rest-client
- **Developer Guide:** https://developer.ibm.com/apis/catalog/maximo--maximo-manage-rest-api/
- **Standards:** REST/JSON and XML; OSLC (Open Services for Lifecycle Collaboration); OData for some endpoints
- **Authentication:** OAuth 2.0 / OpenID Connect; API key support for legacy integrations

---

### Esri ArcGIS Utility Network Service

- **Description:** GIS-native utility network topology management service providing tracing, subnetwork management, topology validation, and offline editing for electric, gas, water, and telecom utility networks.
- **API Documentation:** https://developers.arcgis.com/rest/services-reference/overview-of-utility-network-services.htm
- **SDKs/Libraries:** ArcGIS Maps SDK for JavaScript (UtilityNetwork class): https://developers.arcgis.com/javascript/latest/api-reference/esri-networks-UtilityNetwork.html · ArcPy (Python): https://pro.arcgis.com/en/pro-app/arcpy/
- **Developer Guide:** https://developers.arcgis.com/documentation/ · https://developers.arcgis.com/rest/
- **Standards:** OGC-compliant WFS, WMS, WMTS, WCS; REST/JSON; GeoJSON
- **Authentication:** ArcGIS OAuth 2.0; ArcGIS token-based authentication; requires licensed ArcGIS Utility Network user type extension

---

### Trimble Unity Maintain (Cityworks)

- **Description:** GIS-centric public asset management platform for local government and public utilities covering work orders, inspections, permits, and asset lifecycle management; built on ArcGIS Enterprise.
- **API Documentation:** https://developer.trimble.com/docs/unity-construct/reference/query-parameters/
- **SDKs/Libraries:** Trimble App Xchange integration connectors: https://appxchange.trimble.com/connectors/trimble-unity-maintain · FME connectors: https://support.safe.com/hc/en-us/articles/25407469587981
- **Developer Guide:** https://www.trimble.com/en/developer/docs
- **Standards:** REST/JSON; OpenAPI documented connectors via App Xchange; ArcGIS service consumption
- **Authentication:** Single Sign-On via ArcGIS/Trimble identity; API key for system integrations

---

### SAP S/4HANA Asset Management

- **Description:** ERP-integrated asset management for plant maintenance, linear assets, and equipment calibration; native financial integration for asset lifecycle costing. Utility-specific extensions for NERC CIP and OSHA compliance.
- **API Documentation:** https://api.sap.com/ (SAP Business Accelerator Hub — search "Plant Maintenance" or "Asset Management")
- **SDKs/Libraries:** SAP Cloud SDK (Java / JavaScript): https://sap.github.io/cloud-sdk/ · BTP SDK for extension development
- **Developer Guide:** https://www.apideck.com/blog/guide-to-sap-4-hana-rest-and-soap-api · https://learning.sap.com/
- **Standards:** OData V2 and V4 (REST); SOAP for legacy integrations; OpenAPI descriptions available on SAP API Hub
- **Authentication:** OAuth 2.0 (SAP BTP identity provider); SAP Logon Tickets for on-premise

---

### Hexagon HxGN EAM

- **Description:** Enterprise asset management for utilities, oil & gas, transportation, and government with GIS integration, compliance tracking (NERC CIP, ISO 55000), and IIoT sensor connectivity. REST services defined in OpenAPI 3.
- **API Documentation:** https://docs.hexagonppm.com/api/khub/documents/MmP3HEXid3DhIBwLBvsZXg/content (HxGN EAM REST Web Services) · https://docs.hexagonali.com (GIS integration docs)
- **SDKs/Libraries:** HxGN EAM Python Framework: https://hexagon.com/company/newsroom/press-releases/2023/hxgn-eam-python-framework-now-available-from-hexagon · HxGN EAM Databridge Pro (Apache NiFi-based): https://hexagon.com/company/newsroom/press-releases/2024/hexagon-introduces-hxgn-eam-databridge-pro-a-module-for-enhanced-data-integration
- **Developer Guide:** https://docs.hexagonppm.com/ · Swagger documentation available per installation
- **Standards:** OpenAPI 3 (REST/JSON); SOAP also supported; OGC-compliant GIS layer consumption via Esri
- **Authentication:** API key generation per installation; OAuth 2.0 for cloud deployments

---

### Infor EAM

- **Description:** Enterprise asset management with 30+ years of deployment in utilities, oil & gas, and government. Pre-built industry configuration packages, reliability planning and analysis (RPA/FMEA), and calibration management.
- **API Documentation:** https://www3.technologyevaluation.com/solutions/16945/infor-eam (feature reference) · Official Infor documentation via Infor OS portal
- **SDKs/Libraries:** Infor OS integration platform; FME connector support
- **Developer Guide:** Available via Infor customer portal; Infor ION integration bus for REST/API orchestration
- **Standards:** REST/JSON; Infor ION (messaging bus); OData for some integrations
- **Authentication:** Infor OS OAuth 2.0; LDAP/Active Directory federation

---

### eMaint CMMS (Fluke)

- **Description:** Cloud CMMS with mobile inspections, condition monitoring via Fluke IIoT wireless sensors, and AI-powered vibration fault detection. Accessible pricing for mid-market utility and industrial operators.
- **API Documentation:** https://www.emaint.com/ (API documentation available to customers via support portal)
- **SDKs/Libraries:** SCADA/PLC integration via standard data connectors (Modbus, OPC-UA); Fluke wireless sensor SDK
- **Developer Guide:** Customer support portal documentation; Fluke sensor integration guides at https://www.fluke.com/
- **Standards:** REST/JSON API; OPC-UA for sensor/SCADA connectivity
- **Authentication:** API key authentication; SSO via customer identity providers

---

### OpenGov Enterprise Asset Management

- **Description:** Cloud EAM for US public agencies and municipal utilities covering infrastructure assets (roads, water, parks, facilities) with map-centric GIS, work orders, capital improvement planning, and public service request portals.
- **API Documentation:** https://opengov.com/products/asset-management/ (customer API access via support)
- **SDKs/Libraries:** ArcGIS integration (Esri Authorised Partner)
- **Developer Guide:** Via OpenGov customer portal; Esri ArcGIS integration documentation
- **Standards:** REST/JSON; ArcGIS/OGC-compliant GIS services
- **Authentication:** OAuth 2.0 / SSO via government identity providers

---

## Notes

**Emerging standards to monitor:**
- OGC API Features Part 2 (CRS extension) is maturing and will be important for multi-coordinate-reference-system utility data exchange
- IEC 61968-13 (Customer Operations CIM) and IEC 61968-14 (AMI CIM) are gaining adoption as smart metering integrates with asset management systems
- NERC CIP-015-1 enforcement dates (beginning 2026) are driving demand for asset management platforms that generate INSM compliance evidence
- W3C SOSA/SSN (Sensor, Observation, Sample, and Actuator) ontology is becoming the preferred standard for IoT sensor data modelling in semantic utility platforms

**Gaps in available developer documentation:**
- Infor EAM developer documentation is largely behind a customer login wall; public API reference is limited
- eMaint public API documentation is not published; only available to licensed customers
- SAP S/4HANA plant maintenance API search on the SAP Business Accelerator Hub requires navigation through 2700+ APIs to locate the relevant asset management endpoints
