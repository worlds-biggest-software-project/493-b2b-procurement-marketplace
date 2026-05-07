# B2B Procurement Marketplace

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native procurement platform that connects buyers and suppliers through structured RFQ workflows, automated bid analysis, and transparent spend management.

B2B Procurement Marketplace is a multi-supplier procurement platform designed for mid-market and enterprise buyers who need structured sourcing, purchase management, and supplier collaboration without the six-figure contracts and 12-month implementations demanded by incumbent platforms. It targets CPOs, procurement directors, finance teams, and IT procurement managers who want a modern, extensible alternative built on open standards.

---

## Why B2B Procurement Marketplace?

- **Enterprise platforms are prohibitively expensive.** SAP Ariba and Coupa command USD 100k--1M+ annual contracts with implementations that take 6--18 months, putting structured procurement out of reach for mid-market companies.
- **Mid-market tools are too narrow.** Products like Prokuria cover sourcing events but lack marketplace features, supplier networks, and AP automation. Buyers outgrow them quickly.
- **No credible open-source option exists.** Odoo and ERPNext offer basic purchasing modules but lack RFQ distribution, supplier discovery, eAuction capability, and spend analytics at the level procurement teams require.
- **AI is bolted on, not built in.** Incumbents are retrofitting AI copilots onto legacy architectures (SAP's Joule, Coupa's Navi). An AI-native design can embed intelligence into the core workflow from day one.
- **Buyer UX remains poor.** Enterprise procurement systems are criticised for dated interfaces, steep learning curves, and heavy configuration overhead. Buyers expect consumer-grade experiences.

---

## Key Features

### Sourcing and RFQ Management
- RFQ, RFP, and RFI creation with digital tender management and structured response collection
- Distribution to invited suppliers with email-based and portal-based quote submission
- Side-by-side bid comparison with manual and AI-assisted award workflows
- Reverse auction and eAuction module for competitive bidding scenarios

### Purchase Order and Invoice Automation
- Purchase order generation from awarded sourcing events with approval chains and status tracking
- Three-way matching: purchase order, goods receipt, and invoice
- AP automation with invoice capture and matching against approved orders
- Multi-currency and multi-jurisdiction support for cross-border procurement

### Supplier Management
- Buyer and supplier account creation with role-based access and approval hierarchies
- Supplier profile management with compliance document upload and verification
- Supplier performance scoring and rating after order completion
- ESG credential verification and sustainability scoring during onboarding

### Spend Analytics and Compliance
- Spend analytics dashboards with category, vendor, and business unit breakdowns
- Maverick spend detection against approved procurement policies
- Contract repository with expiry alerts and renewal workflows
- Budget guardrails surfacing available budget at the point of request

### Integration and Interoperability
- REST API with OAuth 2.0 authentication for ERP integration
- cXML PunchOut support for buyers using external procurement systems
- EDI (ANSI X12 / EDIFACT) and UBL document exchange
- Pre-built connectors for SAP, Oracle, NetSuite, and Workday

---

## AI-Native Advantage

Unlike incumbents that bolt AI assistants onto legacy procurement workflows, B2B Procurement Marketplace is designed AI-first. The platform generates structured RFQ documents from plain-language buyer requirements, reducing sourcing cycle time from days to minutes. Embedding-based supplier matching identifies best-fit vendors from large catalogs using capability, compliance status, and past performance data. AI summarises and scores competing bids with confidence-scored shortlist recommendations, while NLP-driven contract risk extraction flags non-standard terms and missing clauses before sign-off. Spend pattern anomaly detection operates in real time against approved procurement policies to catch off-contract purchasing and duplicate orders.

---

## Tech Stack & Deployment

The platform targets self-hosted, cloud, and hybrid deployment models. It is built on open B2B standards: cXML and PunchOut 2.0 for catalog integration, UBL (OASIS) for electronic business documents, EDI (ANSI X12 / EDIFACT) for purchase order and invoice exchange, and PEPPOL for cross-border document interoperability. The REST API uses OAuth 2.0 authentication. ERP integration is a baseline requirement, with connectors planned for SAP S/4HANA, Oracle, NetSuite, and Workday. The legal and IP landscape is clear: ERPNext (GPL v3) and Odoo Community (LGPL v3) demonstrate that open-source procurement tooling is viable, and the core protocols are all open standards.

---

## Market Context

Global B2B ecommerce is projected to reach USD 36 trillion in 2026, growing at approximately 14.5% CAGR, with the procurement software segment sustaining consistent double-digit growth. Enterprise platforms command USD 100k--1M+ annual contracts (Coupa was acquired for USD 8 billion in 2023), while mid-market SaaS tools range from USD 350--1,000/month. Primary buyers are CPOs and procurement directors at manufacturing, retail, and logistics companies, finance teams seeking AP automation and spend visibility, and sustainability officers requiring ESG-compliant supplier sourcing.

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
