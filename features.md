# B2B Procurement Marketplace — Feature & Functionality Survey

> Candidate #493 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| SAP Ariba | Enterprise S2P suite | Commercial — enterprise pricing | https://www.ariba.com/ |
| Coupa | Business spend management | Commercial — enterprise pricing | https://www.coupa.com/ |
| JAGGAER One | Source-to-pay suite | Commercial — enterprise pricing | https://www.jaggaer.com/ |
| Ivalua | S2P platform | Commercial — enterprise pricing | https://www.ivalua.com/ |
| Zip | Procurement orchestration | Commercial — SaaS | https://ziphq.com/ |
| Prokuria | RFI/RFP/RFQ management | Commercial — from $349/mo | https://prokuria.com/ |
| Amazon Business | Enterprise procurement marketplace | Commercial — free + fees | https://business.amazon.com/ |
| Odoo (Purchase module) | Open-core ERP with procurement | Open core — LGPL/enterprise | https://www.odoo.com/ |
| ERPNext (Buying module) | Open-source ERP with procurement | Open source — GPL v3 | https://frappe.io/erpnext/ |
| Zapro.ai | AI-native procurement platform | Commercial — from $15k/year | https://zapro.ai/ |

---

## Feature Analysis by Solution

### SAP Ariba

**Core features**
- SAP Business Network: 5.4 million-company global supplier directory with onboarding workflows
- Sourcing and e-auction tools (RFI, RFP, RFQ, reverse auction)
- Contract lifecycle management with compliance and audit trails
- Purchase order and invoice automation integrated with SAP ERP
- Spend analytics and category management dashboards
- Direct materials procurement with bill-of-materials (BOM) support
- Supply chain collaboration portals for manufacturing partners

**Differentiating features**
- Deepest integration with SAP S/4HANA for direct materials and manufacturing use cases
- Joule AI copilot (Feb 2026 next-gen rebuild) embedded across all workflows, supporting 11 languages
- World's largest structured B2B supplier network by company count

**UX patterns**
- Enterprise-grade UI with guided workflows and approval chains
- Role-based dashboards for procurement, finance, and supply-chain teams
- Significant configuration overhead — implementations typically take 6–18 months

**Integration points**
- Native SAP ERP/S4HANA connectors
- cXML PunchOut catalog integration with external supplier systems
- EDI (ANSI X12 / EDIFACT) for PO and invoice exchange
- REST and SOAP APIs via SAP Ariba Developer Portal (developer.ariba.com)
- OAuth 2.0 authentication for all API access

**Known gaps**
- Steep learning curve and slow adoption by end-users outside of core procurement teams
- High total cost of ownership; out of reach for mid-market companies
- UI criticised as dated compared to consumer-grade experiences
- Poor self-service onboarding for new suppliers unfamiliar with cXML

**Licence / IP notes**
- Proprietary; acquired by SAP. No open-source components. Enterprise licence agreements.

---

### Coupa

**Core features**
- Unified spend management across procurement, invoicing, expenses, and travel
- Community Intelligence — benchmarking and predictive recommendations drawn from aggregate customer spend data
- Supplier risk management with financial health scoring and ESG screening
- Contract management with obligation tracking and renewal alerts
- Budgeting module with real-time budget availability visible to requesters
- AP automation including invoice capture (OCR), matching, and payment
- Navi AI agent portfolio (launched 2024, multi-agent expansion 2025) for workflow automation

**Differentiating features**
- Community spend data network enables predictive category pricing benchmarks
- Budget guardrails surface available budget to employees at point of request
- Navi multi-agent orchestration for automated procurement decisions

**UX patterns**
- More consumer-friendly UI than SAP Ariba
- Progressive disclosure: simple intake forms expand only when thresholds are exceeded
- Mobile-ready for approvals and expense claims

**Integration points**
- REST API with OAuth 2.0 (OIDC) authentication (compass.coupa.com)
- Open Buy API for marketplace catalog integration
- ERP connectors for SAP, Oracle, NetSuite, Workday
- AWS Marketplace PunchOut / cXML integration

**Known gaps**
- Community Intelligence requires a large customer base to be meaningful for niche categories
- AP automation quality can vary for non-standard invoice formats
- Less capable than SAP Ariba for direct materials and manufacturing-specific procurement

**Licence / IP notes**
- Proprietary; acquired by Thoma Bravo for $8B in 2023. No open-source components.

---

### JAGGAER One

**Core features**
- Unified source-to-pay covering sourcing, contracts, supplier management, and P2P
- AI-powered supplier discovery and capability matching
- eAuction tools (reverse, forward, Dutch, Japanese, combinatorial)
- Direct materials procurement with BOM and engineering change management
- Supplier collaboration portals for forecast sharing and consignment management
- Advanced spend analytics with category tree and contract compliance reporting

**Differentiating features**
- Strongest eAuction variety among mid-enterprise platforms
- Direct materials and engineering-grade supplier collaboration features

**UX patterns**
- Feature-rich but complex; administrators face a steep configuration curve
- Highly customisable workflows — advantage and liability simultaneously

**Integration points**
- REST API with OAuth 2.0
- ERP connectors (SAP, Oracle, Coupa, Workday)
- EDI and cXML PunchOut support

**Known gaps**
- Users report JAGGAER's workflows are rigid and difficult to adapt to non-standard processes
- Support response times criticised in user reviews (G2, Capterra)
- Data migration and ongoing maintenance complexity
- Platform reportedly slows transactions in some supplier portal interactions

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Ivalua

**Core features**
- Full source-to-pay suite on a single data model and UX layer
- No-code workflow and data structure configurability
- Supplier lifecycle management including onboarding, risk scoring, and performance reviews
- Integrated contract management with clause library
- Direct and indirect spend coverage including services procurement
- Add-On Store for modular capability extensions without custom code

**Differentiating features**
- No-code configurability enabling process customisation without software development
- Single unified data model avoiding cross-module data synchronisation issues

**UX patterns**
- Administrator-facing configurability is a key strength; end-user UX can feel complex
- Role-specific portals for buyers, suppliers, and approvers

**Integration points**
- REST API with ERP-ready pre-built connectors
- Certified integrations with SAP, Oracle, Microsoft Dynamics
- Supports EDI, cXML, and PEPPOL for document exchange

**Known gaps**
- Implementation can be lengthy and expensive even by enterprise standards
- Requires technical support for ongoing configuration changes; not truly self-service
- Learning curve is steep for administrators and end-users without training

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Zip

**Core features**
- Intake-to-pay orchestration with a single purchase-request entry point
- No-code approval workflow builder with dynamic routing by amount, vendor, department
- Supplier onboarding with parallel legal, compliance, and security review tracks
- Contract tracking with renewal alerts and risk flagging
- AI intake agents that auto-populate request details and recommend appropriate workflows
- App Studio: low-code native integration platform for custom connectors

**Differentiating features**
- Employee-facing intake experience designed to be as simple as a consumer app
- Named a Visionary in the 2026 Gartner Magic Quadrant for S2P
- Orchestration layer that sits above existing ERPs and procurement tools rather than replacing them

**UX patterns**
- Consumer-grade intake UX; strong progressive disclosure model
- Minimal training required for end-users — complex rules are hidden behind a simple request form
- Mobile-first approval flows

**Integration points**
- Comprehensive REST API (docs.ziphq.com) with API key and OAuth 2.0 options
- Pre-built connectors to Coupa, SAP Ariba, NetSuite, Workday, Slack, Teams
- App Studio for building custom low-code integrations

**Known gaps**
- Positioned as an orchestration layer rather than a full S2P suite — lacks deep sourcing and auction capabilities
- Supplier-facing portal is less developed than buyer-facing intake
- Smaller supplier network than SAP Ariba or Amazon Business

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Prokuria

**Core features**
- RFI, RFP, and RFQ creation with digital tender management
- Supplier evaluation scorecards and side-by-side bid comparison
- Vendor database with performance history and compliance document storage
- Purchase order generation linked to awarded sourcing events
- Invoice repository for approved supplier transactions
- Reverse auction capability for competitive bidding

**Differentiating features**
- Fastest-to-deploy sourcing tool in the mid-market segment
- Clean, opinionated UX focused purely on sourcing events

**UX patterns**
- Simple and fast onboarding for procurement teams with limited IT resources
- Guided sourcing event creation wizard reduces setup errors

**Integration points**
- API access available (not publicly documented in depth)
- ERP integration via CSV export/import and Zapier

**Known gaps**
- Not a full marketplace — no supplier catalog browsing or public supplier network
- Limited AP automation and no native invoicing
- Limited analytics compared to enterprise platforms

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Amazon Business

**Core features**
- Vast product catalog with B2B pricing, quantity discounts, and business-only offers
- Guided buying and approval workflows for purchase compliance
- PunchOut catalog integration via cXML for ERP/procurement system users
- Spend analytics with category and vendor breakdown
- Multi-user business accounts with role-based purchasing controls
- Tax exemption management for eligible organisations
- ESG supplier labeling (small business, diversity-certified, sustainable)

**Differentiating features**
- Consumer-grade product discovery UX applied to enterprise purchasing
- Largest single supplier catalog accessible via PunchOut from any procurement system

**UX patterns**
- Familiar Amazon consumer experience — minimal training required
- Comparison tables for competing product listings from multiple sellers
- Notifications and tracking integrated with existing account management

**Integration points**
- cXML PunchOut integration (setup guide at business.amazon.com)
- OGI and OAG document exchange languages for order transmission
- Integration with SAP Ariba, Coupa, Jaggaer, and other procurement platforms

**Known gaps**
- Not a private-label or white-label marketplace — no support for buyer-only supplier networks
- Limited support for services procurement, contracts, and custom RFQ workflows
- No native supplier performance management or compliance tracking

**Licence / IP notes**
- Proprietary platform operated by Amazon. PunchOut integration is standards-based (cXML).

---

### Odoo (Purchase module)

**Core features**
- Request for Quotation (RFQ) creation and supplier comparison
- Purchase Order (PO) management with approval chains
- Vendor pricelist and contract management
- Automatic reorder rules based on stock levels
- Three-way matching (PO, receipt, invoice) for AP automation
- Supplier portal for quote submission
- Integration with Inventory, Accounting, and Manufacturing modules

**Differentiating features**
- All-in-one ERP suite means procurement data flows natively into inventory, accounting, and production
- Marketplace of 30,000+ community and enterprise apps extending functionality

**UX patterns**
- Modern web UI with Kanban and list views
- Activity and chatter threads on every document for stakeholder collaboration
- Low-code customisation via Odoo Studio (enterprise tier)

**Integration points**
- REST and JSON-RPC APIs for all Odoo models
- XML-RPC legacy API
- EDI connectors available via community modules
- Webhook support for real-time event notification

**Known gaps**
- Open core model — advanced features (e-signature, IoT, marketing automation) require paid enterprise licences
- Limited native sourcing/auction capabilities; no reverse auction support
- Supplier network is not managed — buyers must build their own supplier relationships
- Weaker analytics compared to dedicated procurement platforms

**Licence / IP notes**
- Community edition: LGPL v3. Enterprise edition: proprietary commercial licence. Some modules are AGPL.

---

### ERPNext (Buying module)

**Core features**
- Request for Quotation (RFQ) via supplier portal with email-based quote collection
- Supplier Quotation comparison tool
- Purchase Order creation and tracking
- Material Request auto-generation based on reorder thresholds
- Supplier rating and performance scorecard
- Batch inventory and tax automation
- Full integration with ERPNext Accounts and Inventory modules

**Differentiating features**
- 100% free and open source (GPL v3) with no feature-gating
- Frappe Framework REST API is consistent and auto-generated for all doctypes

**UX patterns**
- Frappe UI framework provides consistent list, form, and report views
- Print formats and email templates for supplier communication built-in
- Community-driven feature development with active GitHub issue tracker

**Integration points**
- Auto-generated REST API for all document types (doctype-based endpoints)
- Webhooks for real-time event notifications
- Community integration modules for EDI, PunchOut, and eCommerce connectors

**Known gaps**
- No built-in marketplace catalog browsing or public supplier network
- No reverse auction or eAuction capability
- UI is functional but less polished than commercial alternatives
- Community support model means slower issue resolution for non-standard requirements

**Licence / IP notes**
- GPL v3. Fully open source. No proprietary lock-in. Active GitHub: github.com/frappe/erpnext

---

### Zapro.ai

**Core features**
- AI-driven vendor discovery with embedding-based supplier matching
- Automated RFQ generation from plain-language buyer requirements
- Side-by-side bid comparison with AI scoring and shortlist recommendations
- Purchase order automation and AP invoice matching
- Spend analytics with maverick spend detection
- Vendor performance tracking and contract repository

**Differentiating features**
- Natively AI-first — AI is embedded in the core workflow, not bolted on as a copilot
- Natural language RFQ generation reducing sourcing cycle time significantly

**UX patterns**
- Conversational intake for purchase requests
- AI-generated summary cards for each supplier bid
- Dashboard-first design with anomaly highlighting

**Integration points**
- REST API with OAuth 2.0
- ERP connectors for NetSuite, QuickBooks, and Xero
- Zapier and webhook integration

**Known gaps**
- Newer entrant with less proven scale at large enterprise volumes
- Supplier network is smaller than SAP Ariba or Amazon Business
- Limited eAuction capabilities
- Direct materials and manufacturing procurement not yet supported

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Supplier onboarding with compliance document collection and verification
- RFQ/RFP creation, distribution, and response collection
- Purchase order creation, approval workflows, and status tracking
- Side-by-side bid comparison with award management
- Contract storage with expiry alerts and renewal workflows
- Three-way matching: PO, goods receipt, invoice
- Role-based access control with approval hierarchy configuration
- ERP integration (SAP, Oracle, NetSuite, Workday at minimum)
- Spend analytics by category, vendor, and business unit

### Differentiating Features
- Public or semi-public supplier network (SAP Business Network scale advantage)
- Direct materials and BOM-linked procurement (SAP Ariba, JAGGAER)
- Community intelligence / benchmarking from cross-customer spend data (Coupa)
- No-code workflow configurability without technical support (Ivalua, Zip)
- AI-native RFQ generation from natural language (Zapro)
- Consumer-grade intake UX with minimal training requirement (Zip, Amazon Business)
- eAuction variety: reverse, Dutch, Japanese, combinatorial formats (JAGGAER)
- ESG supplier labeling and sustainability scoring (Amazon Business, ISO 20400 aligned tools)

### Underserved Areas / Opportunities
- Self-service supplier discovery for SME buyers who lack a large existing vendor network
- AI-powered contract risk extraction before award (mostly manual in current tools)
- Transparent AI reasoning — most platforms offer AI recommendations with no explainability
- Real-time maverick spend detection against approved procurement policies
- Lightweight, low-cost RFQ tools for mid-market buyers not served by enterprise platforms
- Automated ESG/sustainability credential verification during supplier onboarding
- Seamless multi-currency, multi-jurisdiction compliance for cross-border B2B procurement
- Agentic procurement: fully autonomous AI purchasing agents for low-risk, high-frequency orders

### AI-Augmentation Candidates
- RFQ document generation from plain-language requirements (currently manual)
- Supplier shortlisting and matching based on capability and past performance (currently rule-based)
- Bid scoring and comparison summary (currently manual analysis by procurement analysts)
- Contract clause risk identification (currently manual or requires expensive CLM add-ons)
- Anomaly detection in spend patterns and invoice data (partially addressed by Coupa, Zapro)
- Supplier onboarding data extraction from unstructured documents (certificates, insurance)

---

## Legal & IP Summary

No patents were identified in the research that would specifically block an open-source implementation of a B2B procurement marketplace. The core protocols — cXML, UBL, PEPPOL, and EDI — are open standards. The feature set of incumbent platforms (SAP Ariba, Coupa, JAGGAER) is protected by copyright in their specific implementations but the underlying workflows and data models are well-established industry practice. ERPNext (GPL v3) and Odoo Community (LGPL v3) demonstrate that open-source procurement tooling is legally viable. Any implementation must take care not to copy proprietary database schemas, UI elements, or source code from commercial vendors.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Buyer and supplier account creation with role-based access and approval hierarchies
- RFQ creation, distribution to invited suppliers, and structured response collection
- Side-by-side bid comparison with manual award workflow
- Purchase order generation from awarded RFQ with status tracking
- Basic supplier profile management with compliance document upload
- REST API with OAuth 2.0 authentication for ERP integration

**Should-have (v1.1)**
- AI-assisted RFQ generation from plain-language buyer requirements
- Automated three-way matching (PO, receipt, invoice)
- Contract repository with expiry alerts
- Spend analytics dashboard with category and vendor breakdowns
- cXML PunchOut support for buyers using external procurement systems
- Supplier performance scoring and rating after order completion

**Nice-to-have (backlog)**
- Reverse auction / eAuction module
- AI-powered supplier discovery and matching from a managed supplier catalog
- ESG credential verification and sustainability scoring during supplier onboarding
- Community spend benchmarking across buyer organisations
- Agentic purchasing for low-risk, repeat, commodity orders
- Multi-currency and multi-jurisdiction compliance for cross-border procurement
