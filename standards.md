# Standards & API Reference

> Project: B2B Procurement Marketplace · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 20400:2017 — Sustainable Procurement Guidance**
- URL: https://www.iso.org/standard/63026.html
- Provides guidance on integrating sustainability into procurement decisions across any organisation size or sector. Not a certifiable standard; frameworks supplier evaluation criteria for environmental, social, and governance (ESG) factors. Increasingly referenced in enterprise RFP scoring criteria.

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Defines requirements for an information security management system (ISMS). Relevant as a marketplace handling supplier financial data, contracts, and purchase orders must demonstrate adequate data security controls. Commonly required by enterprise buyer procurement policies for SaaS vendors.

**ISO/IEC 29101 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/45124.html
- Provides a framework for privacy architecture in ICT systems. Relevant for any marketplace processing personal data of procurement contacts, supplier representatives, and buyer employees across jurisdictions.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Defines semantics of the HTTP/1.1 protocol including methods, status codes, and header fields. Foundational for all REST API endpoints in the platform.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the Link header field for expressing relationships between resources. Used for pagination and hypermedia navigation in RESTful procurement APIs.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The foundational authorisation framework for delegated access. Required for any procurement platform API enabling third-party ERP and system integrations.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://www.rfc-editor.org/rfc/rfc6750
- Specifies how to use OAuth 2.0 Bearer tokens in HTTP requests. Standard for API request authentication across all procurement platform integrations (Coupa, SAP Ariba, Zip all implement this).

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on OAuth 2.0 providing user authentication alongside authorisation. Essential for single sign-on (SSO) between a procurement marketplace and enterprise identity providers (Active Directory, Okta).

**W3C JSON-LD 1.1**
- URL: https://www.w3.org/TR/json-ld11/
- Provides a method of encoding Linked Data using JSON. Relevant for structured product catalog data and supplier capability descriptions that may need to interoperate with semantic web applications.

---

### Data Model & API Specifications

**cXML (Commerce XML) — Version 1.2.069**
- URL: https://cxml.org/ and https://xml.cxml.org/current/cXMLReferenceGuide.pdf
- The dominant standard for PunchOut catalog integration between procurement systems and supplier catalogs. Defines PunchOut Setup Request/Response, Order Message, and Invoice Message document types. Essential for any marketplace that needs to expose its catalog to enterprise buyers operating SAP Ariba, Coupa, or JAGGAER.

**PunchOut 2.0 Protocol**
- URL: https://punchoutcommerce.com/guides/punchout/cxml-punchout-setup-request/
- Protocol enabling buyers to browse supplier catalogs from within their procurement system and return shopping carts for PO generation. Built on cXML. Required for integration with Amazon Business and enterprise procurement platforms.

**OASIS UBL 2.1 (Universal Business Language)**
- URL: https://docs.oasis-open.org/ubl/os-UBL-2.1/
- XML schemas for electronic business documents including RFQ, Purchase Order, Invoice, Catalogue, and Dispatch Advice. The data model underpinning PEPPOL BIS Billing 3.0. Provides a standards-based schema for all transactional documents in a procurement marketplace.

**PEPPOL BIS Billing 3.0**
- URL: https://docs.peppol.eu/poacc/billing/3.0/
- Pan-European Public Procurement Online specification built on UBL 2.1 and compliant with EN 16931. Mandatory for public sector procurement in EU member states; becoming mandatory for B2B in Belgium (January 2026) and expanding across the EU. Provides a structured e-invoicing and e-ordering network framework.

**EDI ANSI X12 / EDIFACT**
- ANSI X12 URL: https://www.x12.org/
- EDIFACT URL: https://unece.org/trade/uncefact/introducing-unedifact
- Legacy but widely-deployed electronic data interchange standards for PO (X12 850 / EDIFACT ORDERS) and invoice (X12 810 / EDIFACT INVOIC) exchange. Required for integration with large enterprise buyers and suppliers in manufacturing, retail, and logistics sectors.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing REST APIs in a machine-readable format. Should be used to document all platform API endpoints, enabling auto-generation of client SDKs and Postman collections. Supports OAuth 2.0 and OpenID Connect security scheme definitions natively.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/draft/2020-12
- Standard for validating the structure and content of JSON documents. Used for request/response validation in REST APIs and for defining procurement document schemas (RFQ, PO, Invoice payloads).

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) with PKCE (RFC 7636)**
- URL: https://www.rfc-editor.org/rfc/rfc7636
- Proof Key for Code Exchange extension makes OAuth 2.0 authorisation code flow safe for public clients (mobile apps, SPAs). Required for any buyer-facing or supplier-facing web application integrating with third-party identity providers.

**FAPI 2.0 Security Profile**
- URL: https://openid.net/specs/fapi-2_0-security-profile.html
- Financial-grade API security profile providing stronger assurances for high-value transactions. Relevant for procurement platforms handling large-value purchase orders and payment approvals where stronger client authentication is warranted.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Documents the most critical API security risks including broken object-level authorisation, lack of authentication, and mass assignment. Essential reference for procurement marketplace API design, particularly where buyer and supplier data must be strictly isolated.

**GDPR (EU Regulation 2016/679)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32016R0679
- EU General Data Protection Regulation governing processing of personal data. Affects storage of procurement contact details, supplier representative information, and behavioural analytics. Requires data processing agreements with suppliers accessing the platform in EU markets.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI models to external data sources and tools. Relevant for an AI-native procurement platform exposing procurement actions (create RFQ, search suppliers, get spend summary) as MCP tools that can be consumed by AI agents and LLM-based procurement copilots.

---

## Similar Products — Developer Documentation & APIs

### SAP Ariba

- **Description:** Market-leading enterprise source-to-pay suite with the world's largest B2B supplier network (5.4M companies on SAP Business Network). Covers sourcing, contracts, P2P, and supply chain collaboration.
- **API Documentation:** https://help.sap.com/docs/ariba-apis and https://api.sap.com/products/SAPAriba/apis/REST
- **Developer Portal:** https://developer.ariba.com/
- **SDKs/Libraries:** SAP Integration Suite connectors; community samples at https://github.com/SAP-samples/ariba-extensibility-samples
- **Developer Guide:** https://apiworx.com/diy-developer-guide-building-custom-integrations-for-sap-ariba/
- **Standards:** REST/JSON, SOAP, cXML, EDI, OAuth 2.0
- **Authentication:** OAuth 2.0 (client credentials and authorisation code flows)

---

### Coupa

- **Description:** Cloud-based business spend management platform covering procurement, invoicing, expenses, and supplier risk. Known for Community Intelligence benchmarking and Navi AI agent portfolio.
- **API Documentation:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation
- **Core API Reference:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api/get-started-with-the-api
- **Open Buy API:** https://compass.coupa.com/en-us/products/product-documentation/supplier-resources/for-suppliers/integration-resources/open-buy-api-reference
- **SDKs/Libraries:** No official SDK; community wrappers at https://github.com/api-evangelist/coupa
- **Standards:** REST/JSON, cXML PunchOut, EDI, OpenAPI
- **Authentication:** OAuth 2.0 / OpenID Connect (OIDC); X-COUPA-API-KEY header for legacy integrations

---

### PEPPOL Network

- **Description:** Pan-European Public Procurement Online network for structured electronic document exchange (invoices, orders, catalogues) across EU public and private sectors. Mandatory for public sector procurement across EU member states.
- **API Documentation:** https://peppol.org/documentation/
- **Directory REST API:** https://directory.peppol.eu/public/menuitem-docs-rest-api
- **Third-party REST API (Storecove):** https://www.storecove.com/docs/
- **Third-party REST API (Peppox):** https://peppox.com/developer/
- **Third-party REST API (A-Cube):** https://docs.acubeapi.com/documentation/peppol/
- **Standards:** UBL 2.1, EN 16931, PEPPOL BIS Billing 3.0, REST/JSON (via access points)
- **Authentication:** mTLS (transport layer) between access points; OAuth 2.0 via third-party providers

---

### Zip (ZipHQ)

- **Description:** Procurement orchestration platform acting as an employee-facing intake layer above existing ERPs and procurement systems. Named a Gartner Magic Quadrant Visionary in 2026.
- **API Documentation:** https://docs.ziphq.com/
- **SDKs/Libraries:** No public SDK; custom integrations via App Studio
- **Integration Ecosystem:** https://ziphq.com/capabilities/integration-ecosystem
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0 and API key

---

### ERPNext / Frappe (Open Source)

- **Description:** 100% open-source ERP (GPL v3) with a full Buying/Procurement module covering RFQ, PO, supplier management, and three-way matching. Auto-generated REST API for all document types.
- **API Documentation:** https://docs.erpnext.com/
- **Frappe REST API:** https://frappeframework.com/docs/user/en/api/rest
- **GitHub Repository:** https://github.com/frappe/erpnext
- **Standards:** REST/JSON, JSON-RPC, XML-RPC (legacy)
- **Authentication:** API key + secret; OAuth 2.0 support via Frappe framework

---

### Odoo (Open-Core)

- **Description:** Open-core ERP with integrated Purchase module covering RFQ, PO, vendor pricelists, and AP matching. Available as Community (LGPL) or Enterprise (proprietary) editions.
- **API Documentation:** https://www.odoo.com/documentation/17.0/developer/reference/external_api.html
- **GitHub Repository:** https://github.com/odoo/odoo
- **SDKs/Libraries:** Community Python, JavaScript, Ruby clients; official `odoorpc` Python library
- **Standards:** JSON-RPC 2.0, REST (v2 API), XML-RPC
- **Authentication:** API key (v16+) and session-based authentication

---

### Amazon Business (PunchOut)

- **Description:** Enterprise procurement marketplace offering B2B-only pricing, multi-user business accounts, spend analytics, and PunchOut catalog integration for procurement system users.
- **PunchOut Integration Guide:** https://business.amazon.com/en/solutions/systems-integration/punchout
- **Technical Administration PDF:** https://images-na.ssl-images-amazon.com/images/G/01/BISS/B2B/pdfs/AmazonPunchOut_TechnicalAdministrationv1.0_2.5.15._V329709929_.pdf
- **cXML Reference:** https://cxml.org/
- **Standards:** cXML PunchOut, OGI, OAG
- **Authentication:** HTTPS Basic Authentication or certificate-based authentication for PunchOut sessions

---

## Notes

**Evolving mandatory e-invoicing landscape:** Belgium mandated structured B2B e-invoicing via PEPPOL from January 2026. France, Germany, and other EU member states are following with mandatory B2B e-invoicing rollouts through 2026–2028. Any B2B procurement marketplace targeting European customers should treat PEPPOL/UBL 2.1 compliance as a near-term requirement rather than a differentiator.

**Agentic commerce:** Gartner projects 90% of B2B purchases could be handled by AI agents by 2028. The Model Context Protocol (MCP) is emerging as the standard interface for connecting LLM-based purchasing agents to procurement systems. Exposing core marketplace actions (search suppliers, create RFQ, approve PO) as MCP tools positions a platform for the agentic commerce wave ahead of most incumbents.

**cXML version currency:** The cXML Reference Guide 1.2.069 was released February 2026. Implementors should use this version as the canonical specification for PunchOut and order message integration.
