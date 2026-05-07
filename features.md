# Recycling & Waste Management — Feature & Functionality Survey

> Candidate #467 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| AMCS Platform | Enterprise end-to-end ERP + operations | Commercial SaaS / on-premise | https://www.amcsgroup.com/solutions/amcs-platform/ |
| Rubicon | AI-powered waste operations marketplace | Commercial SaaS | https://www.rubicon.com/ |
| CurbWaste | Cloud platform for haulers | Commercial SaaS | https://www.curbwaste.com/ |
| Wastebits | Compliance and waste lifecycle tracking | Commercial SaaS | https://wastebits.com/ |
| Re-TRAC | Recycling programme management & reporting | Commercial SaaS | https://www.re-trac.com/ |
| Routeware | Municipal waste collection software | Commercial SaaS | https://routeware.com/ |
| Starlight Software | Roll-off and commercial waste management | Commercial SaaS (cloud / AWS) | https://www.starlightsoftwaresolutions.com/ |
| Waste Logics | Skip hire and waste broker management | Commercial SaaS | https://wastelogics.com/ |

---

## Feature Analysis by Solution

### AMCS Platform

**Core features**
- Route optimisation with 5–25% reductions in CO2, mileage and driving time
- Driver mobile app with turn-by-turn navigation, ad-hoc order handling, digital signatures
- Weighbridge integration for electronic weigh-ticket generation
- ERP modules: customer management, contracts, pricing, billing, accounts payable/receivable
- Compliance reporting for environmental, health and safety (EHS) obligations
- Asset tracking (bins, containers, vehicles)
- ESG reporting and sustainability dashboards
- Telematics integration (map overview, real-time fleet visibility)
- Recycling material tracking by stream and grade

**Differentiating features**
- Agentic AI framework introduced in 2026 release for operational decision support
- API Accelerator Program providing 80–90% pre-built integration scaffolding for partners
- End-to-end coverage from collection through processing and billing in a single platform
- Configurable compliance rules by jurisdiction

**UX patterns**
- Enterprise dashboard with multi-site, multi-fleet operations view
- Role-based views for drivers, supervisors, finance, and compliance teams
- Progressive disclosure: driver app focused solely on route execution; management layer handles configuration

**Integration points**
- REST API with 80% coverage of the Enterprise Management suite (amcsrestapi.github.io)
- API Accelerator Program with Integration Hub and dedicated developer support
- Telematics hardware agnostic — integrates with Geotab, Samsara and others
- Weighbridge hardware via TCP/IP and serial adapters

**Known gaps**
- Complexity and cost remain barriers for small/mid-size haulers
- Users report steep learning curve for initial configuration
- Limited self-service customisation without professional services engagement

**Licence / IP notes**
- Commercial licence; pricing on request. No open-source components identified. Patent-level IP not publicly disclosed.

---

### Rubicon

**Core features**
- Route optimisation and real-time tracking for waste collection fleets
- Automated billing and customer communication in a single platform
- Sustainability reporting: carbon output, diversion rates, waste-stream analytics
- Marketplace connecting waste generators with haulers and recyclers
- Predictive route optimisation using historical data and traffic patterns
- Compliance tracking integrated with billing

**Differentiating features**
- Network-effect marketplace model: unique in connecting generators, haulers, and recyclers in one platform
- Strong sustainability analytics and automated carbon reporting
- RUBICONSmartCity (now part of Routeware) extended municipal operations to street sweeping and snow removal

**UX patterns**
- Self-service portal for waste generators to request services and view reports
- API integrations with enterprise systems (SAP, Oracle, NetSuite, QuickBooks)
- Dashboards emphasising sustainability KPIs alongside operational metrics

**Integration points**
- API connections to SAP, Oracle, NetSuite and QuickBooks for enterprise billing reconciliation
- IoT sensor integrations for smart container fill-level monitoring

**Known gaps**
- Marketplace model less suited to vertically integrated haulers who do not need a broker layer
- Documentation on the developer API is limited compared to AMCS

**Licence / IP notes**
- Commercial SaaS. Publicly traded (RUBI). No open-source SDK found.

---

### CurbWaste

**Core features**
- Route creation, management and live editing with real-time driver app updates
- Dispatch Scheduler with flexible advance planning (day-of or weeks ahead)
- Three dispatch views: map view, project management board, list view
- Automated invoicing with direct QuickBooks sync (no double entry)
- Mobile app for drivers on iOS and Android
- Container and roll-off inventory tracking
- Customer portal and service requests

**Differentiating features**
- Built "by haulers, for haulers" — strong product-market fit for small and mid-size curbside operators
- Selected as exclusive operational management platform for the Recycling Certification Institute (RCI)
- Live route edits propagate instantly to driver app with full audit trail

**UX patterns**
- Minimal, task-focused driver interface
- Drag-and-drop route and order management on dispatch board
- QuickBooks integration simplifies back-office adoption

**Integration points**
- QuickBooks Online integration for billing
- CurbWaste Apps ecosystem for extensions
- GPS tracking integration (specific hardware not publicly documented)

**Known gaps**
- Limited compliance reporting for regulatory submission workflows
- No weighbridge integration publicly documented
- Primarily curbside and roll-off focused — less suited to transfer stations or MRF operations

**Licence / IP notes**
- Commercial SaaS; pricing not publicly listed. No open-source components found.

---

### Wastebits

**Core features**
- Digital waste approval workflows replacing paper profiles
- EPA e-Manifest generation for hazardous and non-hazardous waste shipments
- Customer self-service portal for submitting waste profiles and checking approval status
- Scale integration for weighing waste shipments
- QuickBooks integration for billing reconciliation
- Automated alerts for expiring profiles, approvals, renewals, rejections
- Clone function for creating new waste streams based on similar approved profiles
- Carbon footprint calculation per waste type (CO2e impact)
- Multi-location compliance standardisation and audit readiness

**Differentiating features**
- Specialist focus on regulatory compliance and chain-of-custody documentation
- EPA e-Manifest system integration (the gold standard for US hazardous waste compliance)
- Customer portal enabling self-service waste profile submission and status tracking
- Running history of waste stream approvals with intelligent change highlighting

**UX patterns**
- Workflow-centric interface built around approval stages
- Role-differentiated access: generators, haulers, treatment facilities
- Audit trail and compliance status prominently surfaced

**Integration points**
- EPA e-Manifest API (RCRA system)
- QuickBooks for billing
- Scale/weighbridge hardware integration

**Known gaps**
- Limited route optimisation or dispatch functionality
- Less suited to residential collection operations
- Primarily US-focused compliance framework

**Licence / IP notes**
- Commercial SaaS. No open-source components found.

---

### Re-TRAC

**Core features**
- Configurable survey-based data collection with skip logic, validation and pre-population
- Municipal waste and recycling programme reporting consolidation
- Diversion and disposal facility reporting (landfills, MRFs, transfer stations, composting)
- Hauler permit management and reporting
- Grant programme management (application processing, progress reporting, payment disbursement)
- Compliance monitoring: inspection scheduling, diversion performance measurement
- Built-in analytics: diversion rates, tonnage breakdowns, trend analysis, GHG emissions, financial performance
- Assigned Customer Success Manager for each account

**Differentiating features**
- Government and programme-manager focused — manages entire data collection pipelines across many reporters
- Highly configurable survey engine with complex data validation built in
- Not an operational tool — a data governance and reporting platform for regulators and programme managers

**UX patterns**
- Survey-driven data entry for reporting parties; portal aggregates submissions centrally
- Analytical dashboards for programme managers measuring aggregate diversion performance

**Integration points**
- Data exports for regulatory submissions
- No public developer API documented

**Known gaps**
- Not an operational tool — no route planning, dispatch, or billing
- API/integration documentation not publicly available
- Limited real-time operations visibility

**Licence / IP notes**
- Commercial SaaS. Pricing on request.

---

### Routeware

**Core features**
- AI-enabled, cloud-based route and service management for refuse collection
- Comprehensive route sequencing for residential, commercial, ad hoc, bulk, and cart delivery
- GPS and telematics integration with real-time fleet visibility
- Automatic route rebalancing when a vehicle goes out of service (CRM and maintenance system integration)
- Municipal citizen portals and government reporting
- Snow removal and street sweeping operational support (via former RUBICONSmartCity acquisition)
- Supervisor real-time service status monitoring as operators complete routes

**Differentiating features**
- Municipal-first product with citizen-facing portal and government reporting built in
- Acquisition of RUBICONSmartCity broadens scope to all municipal fleet operations (waste, street cleaning, snow)
- Demonstrated 30% reduction in stops per route in Loveland CO deployment

**UX patterns**
- Operations-centre dashboard for supervisors; simple driver interface
- Citizen self-service portal for service requests and collection day look-up

**Integration points**
- Geotab Marketplace integration (Routeware SmartCity listed)
- CRM and vehicle maintenance system integrations for automatic route rebalancing
- Government reporting data exports

**Known gaps**
- Primarily municipal-focused — less suited to commercial or industrial waste haulers
- Billing and invoicing functionality not prominently documented
- Weighbridge integration not documented

**Licence / IP notes**
- Commercial SaaS. No open-source components found.

---

### Starlight Software

**Core features**
- Real-time dispatch and route management for roll-off, commercial and residential lines
- Container inventory and fleet management with instant performance analytics
- Driver and contractor mobile app (iOS and Android) with customer account access, pricing, job sites, orders, payments, and material/diversion reports
- Automated billing triggered on work order, job and route completion
- Cloud-based drag-and-drop reporting with customisable KPI dashboards
- Hosted on AWS for elastic scalability

**Differentiating features**
- Exclusive Contractor App enabling subcontractor self-service access to their specific pricing and job data
- Billing automation triggers immediately on route completion — no batch billing cycle
- Covers all three business lines (roll-off, commercial, residential) in one platform
- Cloud-native AWS architecture with no on-premise IT requirements

**UX patterns**
- Role-separated interfaces: dispatcher, driver, contractor, customer
- Drag-and-drop report builder for non-technical users
- Real-time inventory view with performance analytics

**Integration points**
- Cloud-hosted; API documentation not publicly detailed
- Mobile apps on both iOS and Android

**Known gaps**
- Compliance reporting (EPA manifests, hazardous waste) not prominently documented
- Weighbridge integration not documented
- Limited publicly available API documentation for third-party developers

**Licence / IP notes**
- Commercial SaaS. No open-source components found.

---

### Waste Logics

**Core features**
- Order creation and tracking for skip hire, brokerage, and recycling operations
- Driver management and job assignment with permit and licence expiry tracking
- Customer relationship management (CRM) integrated with operations
- Subcontractor management for brokered collections
- KPI reporting and personalised role-based dashboards
- Guided UI prompts to prevent missed workflow steps
- Multi-channel support (phone, chat, email, knowledge base, 24×7)

**Differentiating features**
- Purpose-built for skip hire and waste broker operations — a niche not served by larger platforms
- Contextual prompts guiding users through required steps reduce operator errors
- Covers the full broker workflow: customer order → subcontractor assignment → compliance → invoice

**UX patterns**
- Clean, intuitive interface praised by users for low learning curve
- Role-based dashboards with relevant KPIs surfaced per job function
- Guided completion prompts for order and compliance workflows

**Integration points**
- API not publicly documented
- Accounting integrations not specified in available sources

**Known gaps**
- Limited route optimisation compared to fleet-centric platforms
- No weighbridge integration documented
- Smaller ecosystem than enterprise competitors

**Licence / IP notes**
- Commercial SaaS. Pricing not publicly listed. No open-source components found.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Route creation and sequencing for collection fleets
- Driver mobile app (iOS and Android) with job status updates
- Customer and contract management with service history
- Invoicing and billing (weight-based, volume-based, flat-rate)
- Basic compliance documentation (waste transfer notes, collection manifests)
- GPS and telematics visibility for supervisors
- Reporting and KPI dashboards

### Differentiating Features
- AI/ML-driven route optimisation that improves over time from operational data
- Agentic AI for real-time operational decision support (AMCS 2026)
- Marketplace model connecting generators, haulers, and recyclers (Rubicon)
- EPA e-Manifest and hazardous waste compliance workflows (Wastebits)
- Municipal citizen portals and government reporting (Routeware)
- Contractor/subcontractor self-service app (Starlight)
- Aggregate programme-level data governance for regulators (Re-TRAC)
- Smart container fill-level IoT sensor integration

### Underserved Areas / Opportunities
- **AI-native contamination detection** — photo evidence capture and ML classification of contaminated loads is requested but minimally addressed
- **Cross-jurisdiction compliance automation** — automatically applying the correct regulatory workflow (manifest type, diversion target, reporting cadence) by detected jurisdiction
- **Material commodity price integration** — dynamic pricing per stream updated from live commodity markets (aluminium, cardboard, glass prices) rather than static contract rates
- **Predictive maintenance for collection vehicles** — few platforms expose vehicle health data from telematics for predictive scheduling
- **Carbon accounting per collection event** — granular per-stop or per-route carbon footprint, not just aggregate sustainability dashboards
- **Open data export and interoperability** — users report difficulty extracting data for regulatory submissions without platform lock-in
- **Small operator affordability** — enterprise platforms are cost-prohibitive for independent haulers with 1–10 trucks

### AI-Augmentation Candidates
- **Route optimisation** — replacing rule-based VRP solvers with reinforcement learning that adapts to actual traffic, stop duration variance, and seasonal demand patterns
- **Contamination image classification** — ML models classifying bin photos at point of collection, automatically generating contamination reports and customer notifications
- **Compliance document generation** — LLM-assisted generation of waste transfer notes, manifests, and regulatory submissions from structured operational data
- **Customer billing exception handling** — AI reviewing over/under-weight load exceptions and proposing billing adjustments with supporting evidence
- **Demand forecasting** — predicting collection volume by stream and location to optimise fleet deployment and material recovery facility capacity planning

---

## Legal & IP Summary

All solutions reviewed are commercial SaaS products with proprietary licences. No open-source waste management platforms with comparable feature depth were identified. AMCS holds the most documented IP through its API Accelerator Program and published REST API surface. No specific patents affecting an open-source implementation were found in publicly available sources, though comprehensive patent searches were not conducted. The EPA e-Manifest schema (USEPA/e-manifest on GitHub) is publicly released under a US government open licence and freely usable. The UK Waste Services API standard (communitiesuk.github.io) is also open. Developers building an open-source alternative should be careful not to reproduce proprietary UI patterns or data models, but the core functional scope (routing, compliance, billing) is unencumbered.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Route planning and daily dispatch with driver mobile app (iOS/Android)
- Customer and contract management with service configuration
- Weight and material stream recording (manual entry + weighbridge import)
- Basic compliance document generation (waste transfer notes, collection manifests)
- Invoicing engine with weight-based, volume-based, and flat-rate billing
- GPS fleet tracking and supervisor dashboard

**Should-have (v1.1)**
- AI-assisted route optimisation using historical stop duration and traffic data
- Contamination recording with photo capture and customer notification
- EPA e-Manifest integration (US market) and configurable jurisdiction compliance rules
- QuickBooks / accounting system integration
- Sustainability and diversion rate reporting dashboard
- Container asset tracking (bin and roll-off inventory)

**Nice-to-have (backlog)**
- IoT smart bin fill-level sensor integration
- Predictive vehicle maintenance alerts via telematics API
- Live commodity price integration for dynamic material stream pricing
- Citizen self-service portal for residential collection day look-up and service requests
- Subcontractor / broker workflow module
- Grant programme and regulatory programme management (Re-TRAC-style aggregate reporting)
