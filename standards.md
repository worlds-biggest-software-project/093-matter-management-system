# Standards & API Reference

> Project: Matter Management System · Generated: 2026-05-06

## Industry Standards & Specifications

### Legal Data & Billing Standards

**LEDES (Legal Electronic Data Exchange Standard) — XML Ebilling**
- Official URL: https://ledes.org/
- Specification versions: LEDES XML 2.0 (2006), XML 2.1 (2008), XML 2.2 (2020)
- The de-facto standard for electronic legal invoicing between law firms and corporate clients. Maintained by the LEDES Oversight Committee (LOC), a California nonprofit. XML 2.2 adds tiered-tax support (e.g., Japanese Withholding Tax). Any matter management billing module that targets corporate or insurer clients must export LEDES-compliant invoices. Freely implementable; no licensing fee.
- LEDES format guide: https://ledes.org/xml-format-differences/

**UTBMS (Uniform Task-Based Management System) Codes**
- Official URL: https://utbms.com/
- Maintained by the LEDES Oversight Committee (the same body that governs LEDES billing).
- Standardized task, phase, activity, and expense codes required by most US corporate legal departments and insurers. Five code sets cover Litigation, Intellectual Property, Counseling, Project, and Bankruptcy matters. Codes must be applied to individual time entries in LEDES invoice lines; corporate clients use them for billing analytics and outside counsel guideline enforcement. Freely available and open to implement.

**SALI Alliance — Legal Matter Standard Specification (LMSS) v2**
- Official URL: https://sali.org/
- GitHub repository (MIT licence): https://github.com/sali-legal/LMSS
- LMSS v2 published March 2022; SALI API Standard v1.0 published May 2024.
- An OWL ontology with 18,000+ controlled tags for describing legal matters: matter type, area of law, practice area, industry, jurisdiction, legal entity type, and more. SALI-compliant matter classification enables data portability between systems, structured reporting to corporate law departments, and interoperability with CLM (Contract Lifecycle Management) platforms. The SALI API standard is documented on SwaggerHub at https://app.swaggerhub.com/apis/SALI-LMSS-V1/LMSS-API/1.0.0 and on GitHub at https://github.com/sali-legal/api. Using SALI classification in the data model is the single most important standard for data portability and corporate-client adoption.

### Security & Compliance Standards

**SOC 2 Type II (AICPA Trust Services Criteria)**
- Official URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-tools-aids/soc2
- The primary US security certification expected by enterprise law firm clients when evaluating cloud SaaS vendors. Assesses the design and operating effectiveness of controls across Security, Availability, Processing Integrity, Confidentiality, and Privacy over a 6–12 month audit period. A hosted matter management SaaS tier should target SOC 2 Type II. An open-source self-hosted deployment should document equivalent controls in a published security guide.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- Official URL: https://www.iso.org/standard/82875.html
- The internationally recognised ISMS standard; approximately 80% overlap with SOC 2 criteria. More commonly required by European law firm clients than SOC 2. A project targeting EMEA law firms should pursue ISO 27001 certification or align internal controls to its Annex A control set. Freely available as a reference framework; certification requires a third-party audit.

**HIPAA — Health Insurance Portability and Accountability Act (45 CFR Parts 160 & 164)**
- Official URL: https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- Relevant for matter management systems used by healthcare law practices and firms acting as HIPAA Business Associates. Requires encryption of PHI at rest and in transit, role-based access controls, complete audit logs of PHI access, and Business Associate Agreements (BAAs) with any sub-processors. ArkCase's existing HIPAA compliance is a significant differentiator for healthcare-sector deployments. A new OSS matter management platform targeting healthcare legal should plan for BAA execution and HIPAA-aligned data handling from the outset.

**PCI DSS v4.0.1 — Payment Card Industry Data Security Standard**
- Official URL: https://www.pcisecuritystandards.org/standards/pci-dss/
- Required for any matter management billing module that accepts client credit card payments. The current version (v4.0.1) went into full effect on March 31, 2025 and tightens password requirements (minimum 12 characters, alphanumeric). In practice, most legal billing platforms delegate card processing to PCI-compliant payment processors (LawPay, Stripe, Clio Payments), which significantly limits the platform's in-scope PCI footprint. An OSS matter management billing module should integrate with a PCI-compliant third-party payment processor rather than attempting to self-certify card processing.

**FedRAMP (Federal Risk and Authorization Management Program)**
- Official URL: https://www.fedramp.gov/
- US government certification for cloud products used by federal agencies. Based on NIST SP 800-53 Rev. 5 moderate or high impact baselines. Required for any matter management platform targeting government legal departments (DOJ, military, state attorneys general). ArkCase is the only current open-source case management platform with FedRAMP authorization, making it the reference for government-sector deployments. Achieving FedRAMP authorization is a multi-year, multi-million dollar investment; a new OSS project should consider ArkCase as a foundation for government-sector use cases rather than pursuing FedRAMP from scratch.

**GDPR — General Data Protection Regulation (Regulation (EU) 2016/679)**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016R0679
- Applies to any matter management platform with EU-based law firm clients or that processes personal data of EU data subjects. Requires privacy by design, data minimisation, right to erasure, audit logs, and Data Processing Agreements with sub-processors. Any hosted platform targeting European law firms must document GDPR compliance, appoint a DPO if processing large-scale personal data, and ensure data residency options for EU clients.

**ABA Model Rules of Professional Conduct**
- Official URL: https://www.americanbar.org/groups/professional_responsibility/publications/model_rules_of_professional_conduct/
- Rule 1.15 (Safekeeping of Property): Governs trust accounting (IOLTA) requirements — mandatory separate client trust accounts, monthly three-way reconciliation (bank balance, book balance, client ledger sum), and complete record-keeping. Any billing module with trust accounting must implement three-way reconciliation correctly. Rule 1.4 (Communication): Governs timely client communication obligations that shape client portal feature requirements.

### Authentication & API Standards

**OAuth 2.0 — RFC 6749 (Authorization Framework)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard authorization framework used by Clio, Filevine, Smokeball, and MyCase APIs. Defines Authorization Code, Client Credentials, and other grant types. All legal practice management API integrations (court filing, accounting, e-signature, document management) use OAuth 2.0. An OSS matter management platform's API must implement OAuth 2.0 Authorization Code flow for user-delegated access and Client Credentials flow for server-to-server integrations.

**OpenID Connect Core 1.0**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication identity layer built on OAuth 2.0. Used for Single Sign-On (SSO) in legal enterprise environments (Microsoft Entra ID, Okta, Google Workspace). A matter management system targeting mid-market and enterprise law firms must support OIDC-based SSO for corporate identity provider integration.

**OpenAPI Specification v3.1 (OAS 3.1)**
- Official URL: https://spec.openapis.org/oas/v3.1.0
- The standard for documenting REST APIs. OAS 3.1 is fully compatible with JSON Schema Draft 2020-12. Used by Clio (Swagger/OpenAPI), MyCase (Stoplight), and Smokeball (Stoplight) to publish their API reference documentation. The SALI API Standard is also documented as an OpenAPI specification on SwaggerHub. An OSS matter management platform should publish its REST API as an OAS 3.1 document, enabling automatic client SDK generation and developer tooling.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- Official URL: https://www.rfc-editor.org/rfc/rfc6750
- Defines how access tokens are presented in API requests (Authorization: Bearer header). All major legal practice management APIs use Bearer tokens for API authentication. Required companion to RFC 6749 for any API implementation.

### Data Model & Interoperability

**JSON Schema Draft 2020-12**
- Official URL: https://json-schema.org/specification
- Used with OAS 3.1 to define request and response data models. Enables schema validation, auto-generated documentation, and client SDK generation. Should be used to define the canonical matter, contact, time entry, invoice, and document data models in an OSS matter management API.

**Model Context Protocol (MCP) — Anthropic, November 2025 Specification**
- Official URL: https://modelcontextprotocol.io/specification/2025-11-25
- An emerging open standard defining how AI agents connect to external data sources and tools via a "USB-C-like" standardised interface. Legal tech adoption is accelerating: iManage released an MCP server in 2025, enabling AI agents to autonomously query documents, create tasks, and schedule meetings across systems. An AI-native matter management platform should expose an MCP server as the primary AI agent integration surface, enabling AI coding agents (Claude, Copilot) to access matter data without custom integration code.

---

## Similar Products — Developer Documentation & APIs

### Clio Manage API v4

- **Description:** The dominant legal practice management platform, used by 150,000+ law firms. API v4 covers matters, contacts, time entries, bills, documents, tasks, calendar events, and the client portal.
- **API Documentation:** https://docs.developers.clio.com/clio-manage/api-reference/
- **Developer Hub:** https://docs.developers.clio.com/
- **Authorization Guide:** https://docs.developers.clio.com/api-docs/clio-manage/authorization/
- **SDKs/Libraries:** Official Ruby gem; community libraries in Python and JavaScript (third-party). Example Rails application available on GitHub.
- **Developer Guide (Getting Started):** https://docs.developers.clio.com/handbook/getting-started/example-api-application/
- **Standards:** REST/JSON; OAuth 2.0 Authorization Code flow; responses follow JSON:API conventions
- **Authentication:** OAuth 2.0 (Authorization Code grant); access tokens included as Bearer tokens. Developer accounts available for free after 7-day trial.
- **Webhook support:** Yes — event-driven webhooks for matter, contact, activity, and billing events.
- **Notes:** Clio also exposes a separate Clio Grow (intake/CRM) API at https://docs.developers.clio.com/clio-grow/api-reference/. The 250+ Clio App Marketplace integrations all consume this API.

### Filevine API v2

- **Description:** Enterprise litigation-focused matter management with highly customisable workflows. API v2 covers projects (matters), contacts, documents, tasks, and custom fields.
- **API Documentation:** https://developer.filevine.io/docs/v2-ca/branches/main/31e991e1bfac1-filevine-api-v2
- **Developer Portal:** https://developer.filevine.io/
- **SDKs/Libraries:** No official SDK; unofficial Python wrapper (Treillage) at https://github.com/W1ndst0rm/Treillage
- **Help Center (API section):** https://support.filevine.com/hc/en-us/sections/28543097895835-API
- **Standards:** REST/JSON; OpenAPI documented via Stoplight
- **Authentication:** Personal Access Token (PAT) exchanged for a short-lived Bearer token via the Filevine Identity endpoint. OAuth 2.0 Client Credentials flow for server-to-server integrations.
- **Webhook support:** Yes — Webhook Subscription endpoint for matter, task, and document events; payload includes HMAC signature for verification.
- **Notes:** API access is available only to enterprise customers and certified integration partners. Regional API endpoints (US, CA) differ; documentation links are region-specific.

### MyCase API

- **Description:** All-in-one matter management for small-to-mid-size law firms. Public API launched November 2023. Covers cases, contacts, documents, time entries, billing, and tasks.
- **API Documentation:** https://mycaseapi.stoplight.io/docs/mycase-api-documentation/k5xpc4jyhkom7-getting-started
- **Developer Portal:** https://mycaseapi.stoplight.io/
- **SDKs/Libraries:** No official SDK; REST endpoints documented via Stoplight (OpenAPI format).
- **Standards:** REST/JSON; OpenAPI 3.x via Stoplight
- **Authentication:** API key (Bearer token); available only on MyCase Advanced subscription tier ($89/user/month).
- **Webhook support:** Not confirmed in available documentation.
- **Notes:** The API was released in late 2023; integration partners can apply for certified consultant status to support API implementations.

### Smokeball API

- **Description:** Matter management with automatic time capture (deep Microsoft Office/Outlook integration). API covers matters, contacts, documents, time entries, staff, and billing.
- **API Documentation:** https://docs.smokeball.com/docs/api-docs/1e13a13124aee-introduction
- **GitHub (API docs source):** https://github.com/smokeballdev/api-docs
- **SDKs/Libraries:** No official SDK; REST API with JSON payloads documented via Stoplight.
- **Standards:** REST/JSON; OpenAPI documented via Stoplight
- **Authentication:** OAuth 2.0 — both Client Credentials (server-to-server) and Authorization Code (user-delegated) flows. Developer program registration required.
- **Webhook support:** Yes — event-driven webhooks for matter and document events.
- **Notes:** API access requires registration as a Smokeball API partner. The automatic time capture feature is not exposed via API (proprietary desktop integration only).

### Centerbase Developer API

- **Description:** Mid-market matter management with native accounting. API covers matters, contacts, documents, time entries, and billing.
- **API Documentation:** https://support.centerbase.com/hc/en-us/articles/360031534111-Centerbase-Developer-s-API
- **SDKs/Libraries:** No official SDK documented publicly.
- **Standards:** REST/JSON
- **Authentication:** API key; details in support portal.
- **Webhook support:** Not confirmed in available documentation.
- **Notes:** Centerbase's API is less comprehensively documented than Clio or Filevine. A 2026 integration with NetDocuments ndMAX introduces AI-powered document workflows triggered from within the Centerbase platform.

### iManage Universal API v2

- **Description:** Enterprise legal document management (DMS) widely used in large law firms and corporate legal departments. Not a matter management platform per se, but the primary document repository that matter management systems must integrate with.
- **API Documentation:** https://imanage.docs.apiary.io/
- **Postman Collection:** https://www.postman.com/planetary-robot-656518/workspace/team-planetary/api/a4df4397-6e5a-4f5f-9f13-845196b68519
- **Developer Registration:** https://registration.imanage.com/pages/uapi-developer
- **SDKs/Libraries:** iManage SDK (for certified tech partner add-ons); REST API for general integrations.
- **Standards:** REST/JSON (Universal API v2); OpenAPI documented via Apiary
- **Authentication:** OAuth 2.0
- **MCP Server:** iManage released an MCP server in 2025, enabling AI agents to access documents and workspaces via standardised MCP protocol.
- **Notes:** iManage is installed in approximately 4,000 law firms globally. Integration with iManage is expected by any enterprise matter management solution targeting large firms. The MCP server release signals iManage's strategy to become the AI-agent-accessible document layer for legal workflows.

### ArkCase (Open Source)

- **Description:** Open-source, FedRAMP-authorised case management platform (Apache 2.0). The closest OSS reference implementation for a matter management system. Built with Spring Boot (Java).
- **GitHub:** https://github.com/ArkCase/ArkCase
- **Wiki / Architecture:** https://github.com/ArkCase/ArkCase/wiki/
- **Architecture docs:** https://www.arkcase.com/developer-support/architecture/
- **SDKs/Libraries:** Spring Boot Java; extensible plugin/module architecture; LDAP/Active Directory integration.
- **Standards:** REST/JSON; Apache 2.0 licence; FedRAMP authorized; HIPAA compliant
- **Authentication:** LDAP/Active Directory; session-based; no published OAuth 2.0 server.
- **Webhook support:** Not confirmed; extensible via custom module development.
- **Notes:** Legal-specific features (LEDES billing, trust accounting, client portal) are absent from ArkCase. It is the best OSS reference for case/matter data model, workflow engine architecture, and FedRAMP-compliant deployment patterns. Forking or extending ArkCase is a viable technical strategy for a government-sector matter management deployment.

### CourtListener REST API v4 / PACER Integration

- **Description:** Free Law Project's CourtListener provides a public REST API for federal and state case law, PACER dockets, filings, and oral argument recordings. PACER (Public Access to Court Electronic Records) is the US federal courts' own API for case and document access.
- **CourtListener API Documentation:** https://www.courtlistener.com/help/api/rest/
- **PACER Developer Resources:** https://pacer.uscourts.gov/file-case/developer-resources
- **PACER Authentication API:** https://pacer.uscourts.gov/help/pacer/pacer-authentication-api-user-guide
- **PACER Case Locator API:** https://pacer.uscourts.gov/help/pacer/pacer-case-locator-pcl-api-user-guide
- **RECAP APIs (PACER data in CourtListener):** https://www.courtlistener.com/help/api/rest/recap/
- **Standards:** REST/JSON (CourtListener v4); PACER uses proprietary authentication API
- **Authentication:** CourtListener — API token; PACER — PACER account credentials via Authentication API
- **Notes:** CourtListener offers free public access to millions of federal court opinions and PACER dockets. For an OSS matter management system, CourtListener's API enables court deadline import, docket monitoring, and case law research linking without requiring paid legal research subscriptions. PACER direct API access requires a registered PACER account; costs are capped at $3.00/document.

---

## Notes

**SALI API Standard adoption:** The SALI API v1.0 was published in May 2024 and is not yet widely adopted in commercial platforms. Building SALI-compliant matter classification into an OSS matter management data model now would give it a significant interoperability advantage as corporate legal departments and CLM platforms begin requiring SALI codes in outside-counsel billing and matter data.

**LEDES 1998B vs XML formats:** Many smaller law firms and legacy billing systems still use the older LEDES 1998B tab-delimited format rather than XML 2.0/2.1/2.2. A complete implementation should support both: LEDES XML 2.2 for modern corporate clients and LEDES 1998B for backward compatibility with older e-billing platforms (Passport, Collaborati, TyMetrix).

**MCP as the AI integration layer:** The Model Context Protocol (MCP) is becoming the standard interface for AI agents to access external systems. An AI-native matter management platform should prioritise publishing an MCP server alongside its REST API. This enables AI assistants (Claude, Copilot, Gemini) to access matter data, time entries, documents, and billing records via a single standardised protocol without requiring custom integrations for each AI system.

**IOLTA trust accounting is jurisdictional:** Trust accounting requirements under ABA Model Rule 1.15 are implemented differently in each US state. Some states require mandatory IOLTA participation; others permit optional participation. A trust accounting module must accommodate jurisdiction-specific rules, interest rate reporting, and compliance documentation — or explicitly disclaim trust accounting capability and direct users to a dedicated trust accounting tool.

**Emerging: AI agents and court deadline rules:** Jurisdictional court calendar rules (deadline calculation from filing dates) are currently implemented via proprietary rule sets in products like LawToolBox and CompuLaw. An OSS matter management platform could leverage public court rules data from CourtListener and PACER to build an open deadline engine, eliminating reliance on expensive proprietary calendar rules subscriptions.
