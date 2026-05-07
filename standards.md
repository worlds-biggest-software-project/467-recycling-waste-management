# Standards & API Reference

> Project: Recycling & Waste Management · Generated: 2026-05-07

---

## Industry Standards & Specifications

### ISO Standards

**ISO 14001:2026 — Environmental Management Systems**
- URL: https://www.iso.org/standard/14001
- The primary international standard for environmental management systems (EMS). Requires organisations to track waste volumes, recycling rates, and disposal routes with documented KPIs. A waste management platform that supports ISO 14001 compliance must record quantities generated, recycled, reused and sent to disposal, and produce audit-ready evidence of continual improvement.

**ISO 24161:2022 — Waste Collection and Transportation Management — Vocabulary**
- URL: https://www.iso.org/obp/ui/#iso:std:iso:24161:ed-1:v1:en
- Published by ISO/TC 297 (Waste collection and transportation management). Provides the authoritative vocabulary and term definitions for waste collection and transportation. Essential as the semantic baseline for data models, APIs, and regulatory data-exchange schemas in this domain.

**ISO/TC 297 — Waste Collection and Transportation Management (Technical Committee)**
- URL: https://www.iso.org/committee/5902445.html
- The ISO technical committee responsible for standardisation of machines, equipment and management systems for collection, temporary storage and transportation of solid and sanitary liquid waste and recyclables. Monitors and publishes all ISO standards relevant to waste collection operations.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Widely applied by waste and recycling businesses to document and improve service delivery processes. A waste management platform that generates audit trails and process documentation supports customer ISO 9001 certification programmes.

**ISO 45001:2018 — Occupational Health and Safety Management Systems**
- URL: https://www.iso.org/standard/63787.html
- Applicable to waste haulers managing driver safety, pre-trip vehicle inspections, incident reporting and PPE compliance. Software modules for driver licencing, vehicle maintenance, and incident records contribute to ISO 45001 compliance.

---

### Regulatory Frameworks

**US EPA Resource Conservation and Recovery Act (RCRA) — Hazardous Waste Regulations**
- URL: https://www.epa.gov/rcra
- Federal regulatory framework governing hazardous waste generation, transportation, treatment, storage, and disposal in the United States. Defines manifest requirements that the e-Manifest system implements electronically. A US-market waste management platform must understand RCRA-defined waste codes, generator categories, and manifest chain-of-custody rules.

**US EPA e-Manifest System**
- URL: https://www.epa.gov/e-manifest
- Mandatory electronic manifest system for hazardous waste shipments in the United States. Replaces paper manifests with a federally maintained electronic record. Integration with the e-Manifest API (see APIs section) is required for any US hazardous waste compliance workflow.

**EU CSRD/ESRS E5 — European Sustainability Reporting Standard (Waste)**
- URL: https://www.efrag.org/sustainability-reporting
- The European Corporate Sustainability Reporting Directive (CSRD) with ESRS E5 sub-standard requires large EU enterprises to report waste generation, recycling rates and circular economy metrics. A platform targeting EU customers must produce data exports aligned with ESRS E5 disclosures.

**UK Digital Waste Tracking Service**
- URL: https://www.gov.uk/government/publications/digital-waste-tracking-service/digital-waste-tracking-service
- The UK Government's mandatory digital waste tracking framework requiring electronic waste movement records (replacing paper waste transfer notes). Any UK-market platform must integrate with or produce data compatible with this service.

**Extended Producer Responsibility (EPR) — Multiple Jurisdictions**
- EPR regulations requiring manufacturers and importers to fund and report on post-consumer recycling exist in the EU, UK, Canada, and multiple US states. A recycling platform should support the collection and reporting of producer obligation data by material type and tonnage.

---

### W3C & IETF Standards

**W3C Geolocation API (Second Edition, 2025)**
- URL: https://www.w3.org/TR/geolocation/
- Browser-standard API for retrieving device geographic position. Relevant for web-based driver and citizen portal interfaces that use device GPS for location context, missed-stop recording, or real-time fleet visibility.

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the HTTP methods, status codes and headers used by all REST APIs in this space, including the AMCS REST API, EPA e-Manifest API, Geotab SDK, and Samsara REST API.

**RFC 8259 — The JavaScript Object Notation (JSON) Data Interchange Format**
- URL: https://datatracker.ietf.org/doc/html/rfc8259
- JSON is the universal data interchange format for all waste management REST APIs surveyed. The EPA e-Manifest schema (emanifest.json) and AMCS REST API both use JSON.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- OAuth 2.0 is the standard authentication mechanism for telematics APIs (Geotab, Samsara, TN360) and enterprise waste management integrations. A waste management platform integrating with third-party fleet or billing systems must implement OAuth 2.0.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 used by telematics platforms (e.g., TeletracNavman TN360) for SSO integration with customer identity management systems. Relevant when the waste platform integrates with enterprise identity providers.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry-standard schema format for REST API documentation. The AMCS REST API and WM.com API are documented in OpenAPI format. An open-source waste management platform should publish its API using OpenAPI 3.1 for tooling compatibility.

**EPA e-Manifest JSON Schema (RCRA)**
- URL: https://github.com/USEPA/e-manifest/blob/master/Services-Information/Schema/emanifest.json
- The official JSON Schema definition for US hazardous waste electronic manifests. Defines the canonical data structure for waste shipment records, generator/transporter/facility identifiers, waste codes, and handling instructions. Any platform targeting US hazardous waste compliance should align its internal manifest data model with this schema.

**UK Waste Services API Standard**
- URL: https://communitiesuk.github.io/waste-service-standards/apis/waste_services.html
- Open standard from the UK government defining a REST API for local council waste and recycling services, including collection schedules, service types, and reporting endpoints. A UK-market platform should align with this standard for interoperability with local authority systems.

**Open Telematics API**
- URL: https://opentelematicsapi.docs.apiary.io/
- Vendor-neutral telematics API standard for accessing vehicle GPS location, trip data, and diagnostic information across different hardware providers. Relevant for abstracting telematics hardware diversity in a waste fleet management system.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- See above. Required for third-party API integrations (Geotab, Samsara, accounting systems).

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- See above. Required for enterprise SSO scenarios.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Defines the most critical API security risks including broken object-level authorisation, authentication failures, and excessive data exposure. Waste management platforms handling regulated waste data and personal customer information must be designed against these risks.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr-info.eu/
- Governs collection and processing of personal data for EU residents. A platform storing customer location data, driver telemetry, and contact information must implement GDPR-compliant data handling, consent management, and deletion workflows.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is not yet directly applied in waste management software, but is relevant as an open-source AI-native platform would likely expose an MCP server to enable LLM agents to query route status, generate compliance documents, or trigger dispatch actions.

**Model Context Protocol (MCP) Specification**
- URL: https://spec.modelcontextprotocol.io/
- Anthropic-originated open protocol enabling LLM agents to interact with external tools and data sources via structured tool definitions. An AI-native waste management platform could expose route optimisation, manifest generation, and compliance querying as MCP tools, enabling integration with Claude and other LLM-based assistants.

---

## Similar Products — Developer Documentation & APIs

### AMCS REST API

- **Description:** Enterprise-grade REST API covering 80% of the AMCS Platform's Enterprise Management functionality, including routes, collections, customers, scale tickets, and billing data.
- **API Documentation:** https://amcsrestapi.github.io/
- **Developer Programme:** https://www.amcsgroup.com/resources/blogs/api-accelerator-program-how-amcs-platform-is-enabling-an-interconnected-ecosystem/
- **Standards:** REST/JSON, Personal Access Token (PAT) authentication
- **Authentication:** Personal Access Token (PAT)

---

### WM.com Customer API

- **Description:** RESTful API from Waste Management Inc. providing access to customer information, balance due, contract details, invoice history, and pickup status.
- **API Documentation:** https://api.wm.com/
- **Services Overview:** https://api.wm.com/servicesoverview/index.html
- **Standards:** REST/JSON, JWT (JSON Web Token) authorisation
- **Authentication:** JSON Web Token (JWT)

---

### US EPA e-Manifest API (RCRA)

- **Description:** Federal API for submitting, retrieving, and correcting electronic hazardous waste manifests required under RCRA. Mandatory for US hazardous waste compliance integrations.
- **API Documentation:** https://github.com/USEPA/e-manifest
- **JSON Schema:** https://github.com/USEPA/e-manifest/blob/master/Services-Information/Schema/emanifest.json
- **Test Environment:** https://rcrainfopreprod.epa.gov/rcrainfo/
- **Standards:** REST/JSON
- **Authentication:** EPA CDX account; API key via RCRAInfo registration

---

### Geotab Fleet Telematics API

- **Description:** Open telematics platform providing access to vehicle GPS location, engine diagnostics, trip history, and driver behaviour data for mixed-fleet waste collection operations.
- **API Documentation:** https://developers.geotab.com/
- **SDK/Libraries:** JavaScript, .NET, Python SDKs available via developers.geotab.com
- **Marketplace:** https://marketplace.geotab.com/ (Routeware SmartCity listed)
- **Standards:** REST/JSON, proprietary MyGeotab SDK
- **Authentication:** OAuth 2.0 / session-based token

---

### Samsara Fleet API

- **Description:** REST API for accessing Samsara telematics data including real-time vehicle location, driver logs, sensor readings, and maintenance alerts for waste fleets.
- **API Documentation:** https://developers.samsara.com/docs/rest-api-overview
- **Developer Portal:** https://developers.samsara.com/
- **Standards:** REST/JSON, OpenAPI documented endpoints
- **Authentication:** API Key (Bearer token)

---

### ArcGIS Network Analyst / Routing Services (Esri)

- **Description:** GIS-based routing and vehicle routing problem (VRP) solver with a dedicated waste collection solver (ArcGIS Pro 3.5+). Used for optimising curbside residential collection routes at municipal scale.
- **API Documentation:** https://developers.arcgis.com/rest/services-reference/enterprise/an-overview-of-routing-services/
- **Waste Collection Solver:** https://pro.arcgis.com/en/pro-app/latest/help/analysis/networks/waste-collection.htm
- **SDK/Libraries:** ArcGIS Maps SDK (JavaScript, Python arcpy, .NET)
- **Standards:** REST/JSON, ArcGIS REST API
- **Authentication:** ArcGIS Online OAuth 2.0 / API key

---

### Safe Fleet Waste & Recycling Web Services API

- **Description:** Fleet management and vehicle safety platform with a waste and recycling-specific web services API for integrating onboard camera, GPS, and route data.
- **API Documentation:** https://www.safefleet.net/products/fleet-management/waste-recycling-web-services-api/
- **Standards:** REST/JSON web services
- **Authentication:** API key

---

### QuickBooks Online API

- **Description:** Accounting integration used by CurbWaste and Wastebits for automated billing synchronisation. Enables waste management platforms to push invoices, payments, and customer records to QuickBooks without double entry.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/invoice
- **SDK/Libraries:** JavaScript, Python, Java, PHP SDKs (Intuit Developer Portal)
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0

---

## Notes

- **Jurisdiction fragmentation** remains the largest standards challenge. The US (RCRA/e-Manifest), EU (CSRD/ESRS E5, EPR), and UK (Digital Waste Tracking Service) each have distinct regulatory data models with no cross-jurisdiction interoperability standard. An open-source platform should design a pluggable compliance layer abstracting jurisdiction-specific rules.
- **Weighbridge/scale protocols** (serial RS-232, TCP/IP Modbus) are hardware-proprietary and not covered by any open standard. Adapters will be required per hardware vendor (Toledo, Avery Weigh-Tronix, etc.).
- **Telematics hardware diversity** is partially addressed by the Open Telematics API but in practice most deployments use vendor-specific SDKs (Geotab, Samsara). Supporting both via a normalisation layer is recommended.
- **MCP server exposure** for AI-native functionality is an emerging area with no incumbent. An open-source platform publishing MCP tools for route queries, manifest generation, and compliance checks would be novel and differentiating.
