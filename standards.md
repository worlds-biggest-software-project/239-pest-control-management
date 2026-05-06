# Standards & API Reference

> Project: Pest Control Management · Generated: 2026-05-03

---

## Industry Standards & Specifications

### Regulatory & Compliance Standards

**EPA FIFRA — Federal Insecticide, Fungicide, and Rodenticide Act**
- URL: https://www.epa.gov/enforcement/federal-insecticide-fungicide-and-rodenticide-act-fifra-and-federal-facilities
- Governs pesticide registration, product labelling, and lawful use in the United States. Any pest control management software that records pesticide applications must reference EPA Registration Numbers and product labels. Restricted-use pesticide application records must be retained by certified applicators per 7 USC § 136i-1.

**40 CFR Part 158 — Data Requirements for Pesticides**
- URL: https://www.ecfr.gov/current/title-40/chapter-I/subchapter-E/part-158
- Specifies the types of data EPA requires to evaluate pesticide products under FIFRA. Defines the data fields for chemical identity, toxicity, efficacy, and environmental fate that inform pesticide product records stored in compliance software.

**40 CFR Part 170 — Worker Protection Standard (WPS)**
- URL: https://www.ecfr.gov/current/title-40/chapter-I/subchapter-E/part-170
- Requires agricultural employers to maintain pesticide application records (crop/site treated, location, date/time, restricted-entry interval, EPA registration number, active ingredients) and to retain training records for two years. Software must produce records satisfying § 170.311 display requirements for on-site safety information.

**EPA Pesticide Product Label System (PPLS) API**
- URL: https://www.epa.gov/pesticide-labels/pesticide-product-label-system-ppls-application-program-interface-api
- RESTful public API for querying pesticide label data by EPA Registration Number, with JSON responses. Enables software to auto-populate product details (active ingredients, use-site restrictions, re-entry intervals) from official records rather than relying on manual technician entry. US government open data; no licence restriction.

**EPA Pesticide Product Information System (PPIS)**
- URL: https://www.epa.gov/ingredients-used-pesticide-products/pesticide-product-information-system-ppis
- Database of all registered US pesticide products including registrant, chemical ingredients, toxicity category, formulation code, site/pest uses, and registration status. Available as ASCII files for bulk download. Use as a reference data source for chemical catalogues within pest control management applications.

**National Pesticide Information Retrieval System (NPIRS)**
- URL: https://www.npirs.org/public
- Multi-state pesticide registration and licensing data repository. Relevant for tracking technician licensing status across state jurisdictions.

**DOT 49 CFR — Hazardous Materials Transportation**
- URL: https://www.ecfr.gov/current/title-49
- Governs the transportation of restricted-use and hazmat-classified pesticides in service vehicles. Software that manages vehicle inventory and route dispatch should surface DOT classification flags for products carried.

### Industry Certification Standards

**NPMA QualityPro Accreditation**
- URL: https://www.npmaqualitypro.org/available-credentials/qualitypro/
- Industry accreditation programme requiring companies to meet 18 professionalism standards exceeding state and federal mandates, including background checks, drug-free workplace, and documented service standards. Software supporting QualityPro-certified companies must facilitate employee credential tracking and documentation.

**NPMA GreenPro Certification**
- URL: https://www.npmaqualitypro.org/available-credentials/greenpro/
- Certification for companies employing Integrated Pest Management (IPM) strategies. Requires QualityPro accreditation as a prerequisite. IPM principles (prevention, monitoring, targeted treatment, minimum chemical use) should be expressible as service-plan types within pest control management software.

### ISO Standards

**ISO 55000:2024 — Asset Management: Vocabulary, Overview and Principles**
- URL: https://www.iso.org/standard/83053.html
- Provides the foundational vocabulary and principles for managing physical assets. Relevant to managing trap and bait-station inventories as field-placed assets with lifecycle tracking, inspection intervals, and activity records. The 2024 revision includes enhanced guidance on IoT, sensors, and digital twins.

**ISO 55001:2024 — Asset Management System: Requirements**
- URL: https://www.iso.org/standard/83054.html
- Specifies requirements for an asset management system applicable to any asset type. Provides a framework for building trap/device management features that meet enterprise requirements for documented maintenance cycles and compliance evidence.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- International standard for quality management applicable to pest control service businesses. Relevant when designing audit trails, non-conformance reporting, and service-quality documentation features.

### W3C & IETF Standards

**RFC 5545 — iCalendar**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard format for calendar event data (VCALENDAR, VEVENT). Relevant for scheduling export/import between pest control software and customer or technician calendar applications (Google Calendar, Outlook). Enables interoperability for appointment confirmation flows.

**RFC 6350 — vCard**
- URL: https://datatracker.ietf.org/doc/html/rfc6350
- Standard format for electronic business cards and contact data. Relevant for CRM data export and import of customer records into or out of pest control management platforms.

**RFC 7231 — HTTP/1.1 Semantics**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP method semantics (GET, POST, PUT, DELETE, PATCH) underpinning all REST API implementations referenced in this document.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Standard authorisation framework used by Jobber (GraphQL API), ServiceTitan, and WorkWave APIs. Required for any third-party integration with pest control platforms.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, URL-safe claims representation used in API authentication tokens. Relevant for securing API endpoints in a custom pest control management system.

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.x**
- URL: https://www.openapis.org/
- Vendor-neutral, open specification for describing REST APIs. Recommended for documenting the pest control management system's own API surface to enable third-party integrations and developer onboarding.

**GraphQL Specification**
- URL: https://spec.graphql.org/
- Query language for APIs used by Jobber (the most developer-friendly pest control platform). An AI-native pest control system could adopt GraphQL for flexible client-driven queries over job, customer, and service data.

**JSON Schema**
- URL: https://json-schema.org/
- Vocabulary for describing and validating JSON data structures. Relevant for defining canonical data models for pesticide application records, service orders, and customer profiles.

**GeoJSON (RFC 7946)**
- URL: https://datatracker.ietf.org/doc/html/rfc7946
- Standard format for encoding geographic data in JSON. Relevant for representing service locations, route waypoints, and trap placement coordinates in API responses and data exports.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Required for integrating with ServiceTitan, Jobber, and WorkWave/PestPac APIs.

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 for federated authentication. Relevant for SSO integrations with enterprise pest control company identity providers.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- Industry reference for identifying and mitigating the most critical API security risks. Critical for any pest control platform exposing customer PII, payment data, and chemical application records via an API.

**PCI DSS**
- URL: https://www.pcisecuritystandards.org/
- Payment Card Industry Data Security Standard. Required compliance framework for any pest control software accepting card payments in-field or via customer portal.

**GDPR / CCPA**
- URL (GDPR): https://gdpr.eu/
- URL (CCPA): https://oag.ca.gov/privacy/ccpa
- Data protection frameworks applicable to storage and processing of customer PII (name, address, contact history). Relevant for platforms serving EU customers (GDPR) or California residents (CCPA).

---

## Similar Products — Developer Documentation & APIs

### PestPac by WorkWave
- **Description:** Purpose-built pest control platform covering scheduling, routing, chemical tracking, billing, and compliance reporting. Largest installed base among pest-specific software.
- **API Documentation:** https://developer.workwave.com/
- **SDKs/Libraries:** No published SDK; HTTP/HTTPS from any language
- **Developer Guide:** https://www.pestpac.com/api-integrations
- **Standards:** REST/JSON; proprietary authentication
- **Authentication:** API key (one-time setup fee; usage-based additional cost)
- **Notes:** API covers servicing, scheduling, billing, and payments. Access requires active PestPac subscription or approved developer status.

### FieldRoutes
- **Description:** Pest and lawn-care operations platform with industry-leading route clustering. Acquired by ServiceTitan. Operates as a separate product line.
- **API Documentation:** https://api.fieldroutes.com/
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.fieldroutes.com/operations-suite/api-integrations
- **Standards:** REST/JSON
- **Authentication:** API key
- **Notes:** Open API described as available for custom integrations. Limited public developer documentation compared with ServiceTitan.

### ServiceTitan
- **Description:** Enterprise FSM platform used by large pest control operators for multi-location scheduling, dispatch, analytics, and compliance. Parent company of FieldRoutes.
- **API Documentation:** https://developer.servicetitan.io/
- **SDKs/Libraries:** PHP SDK on GitHub (community-maintained): https://github.com/compwright/servicetitan
- **Developer Guide:** https://developer.servicetitan.io/docs/get-going-first-api-call/
- **Standards:** REST/JSON; OpenAPI-described endpoint catalogue
- **Authentication:** OAuth 2.0
- **Notes:** Comprehensive API catalogue with CRUD across most data models and batch record retrieval. Passthrough requests allow access to endpoints beyond the unified model.

### Jobber
- **Description:** General field service platform (pest, lawn, cleaning, etc.) with polished UX and a developer-friendly GraphQL API. 250,000+ active home-service businesses.
- **API Documentation:** https://developer.getjobber.com/docs/
- **SDKs/Libraries:** Ruby on Rails example app: https://github.com/GetJobber/Jobber-AppTemplate-RailsAPI
- **Developer Guide:** https://developer.getjobber.com/docs/getting_started/
- **Standards:** GraphQL; cursor-based Relay pagination; application/json content type required
- **Authentication:** OAuth 2.0; Bearer token in Authorization header
- **Notes:** Query endpoint at https://api.getjobber.com/api/graphql. Rate-limited via DDoS middleware and GraphQL query-cost calculation. App Marketplace available for third-party monetisation.

### Briostack
- **Description:** All-in-one pest control and lawn care platform with integrated marketing automation, barcode scanning, and a public API included in base subscriptions.
- **API Documentation:** https://www.briostack.com/public-api
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.briostack.com/public-api (developer portal sign-up)
- **Standards:** REST (assumed); specifics not fully public
- **Authentication:** API key (free tier included with subscription)
- **Notes:** API positioned as a competitive differentiator for integrations. QuickBooks and Azuga GPS integrations confirmed.

### GorillaDesk
- **Description:** SMB-focused pest, lawn, and cleaning platform with strong chemical tracking and trap device monitoring. Integrations library with Zapier and payment providers.
- **API Documentation:** Not publicly documented
- **SDKs/Libraries:** N/A
- **Developer Guide:** https://gorilladesk.com/integrations-library/
- **Standards:** Zapier integration available
- **Authentication:** N/A (no public API)
- **Notes:** Integrations achieved primarily via Zapier triggers/actions rather than direct API. Chemical and device data not accessible via public API.

### EPA Pesticide Product Label System (PPLS)
- **Description:** US EPA's RESTful public API for querying official pesticide product label data by EPA Registration Number. Returns JSON with product details, active ingredients, re-entry intervals, and site restrictions.
- **API Documentation:** https://www.epa.gov/pesticide-labels/pesticide-product-label-system-ppls-application-program-interface-api
- **SDKs/Libraries:** N/A (standard HTTP/JSON)
- **Developer Guide:** Same as API documentation URL above
- **Standards:** REST/JSON; US government open data
- **Authentication:** None (public endpoint)
- **Notes:** Query by EPA Registration Number. Returns label metadata in JSON. No rate-limit documentation found; treat as a reference data service rather than a high-throughput real-time service.

### QuickBooks Online API (Intuit)
- **Description:** Accounting API used by PestPac, FieldRoutes, ServiceTitan, Jobber, GorillaDesk, Briostack, and Fieldwork for financial data synchronisation. Standard integration requirement for any pest control management platform.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **SDKs/Libraries:** Official SDKs for Java, PHP, Python, .NET, Ruby; community SDKs for JavaScript and Go
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/get-started
- **Standards:** REST/JSON; OpenAPI-described
- **Authentication:** OAuth 2.0
- **Notes:** Required integration for any pest control platform targeting SMBs or enterprise customers. Invoice and payment record sync is a table-stakes expectation.

---

## Notes

**Emerging Area — IoT Trap Sensors:** No mainstream pest control management platform has published a standards-based integration with IoT-connected smart traps (e.g., Anticimex SMART or Rentokil's connected devices). This represents an emerging integration surface. Relevant protocols to monitor include MQTT (ISO/IEC 20922) for IoT messaging and the Matter smart-home standard (Connectivity Standards Alliance), though neither has pest-control-specific profiles yet.

**Gap — No Unified FSM Data Standard:** The field service management category lacks a vendor-neutral data interchange standard (analogous to HL7 FHIR in healthcare). All major platforms use proprietary data models exposed via REST or GraphQL. An open-source pest control platform has an opportunity to define and publish a reference data model (OpenAPI + JSON Schema) that could attract community adoption.

**Regulatory Variability:** US state pesticide applicator licensing requirements vary significantly. The National Pesticide Information Retrieval System (NPIRS) at npirs.org is the best available multi-state data aggregator, but it does not offer a free public API. Direct state agriculture department integrations may be required for full licensing-compliance automation.
