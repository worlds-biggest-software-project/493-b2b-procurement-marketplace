# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core transactional entities in normalized relational columns for referential integrity and query performance, while using JSONB columns to store flexible, tenant-specific, and deeply nested data that would otherwise require EAV tables or frequent schema migrations. GIN indexes on JSONB columns enable efficient querying without sacrificing the schemaless flexibility.

## Why This Suits B2B Procurement

B2B procurement platforms serve diverse organizations with wildly different requirements. A manufacturing CPO needs custom fields for material specs, tolerances, and certifications. A retail procurement director tracks seasonal terms, minimum order quantities, and promotional pricing. A SaaS company's IT procurement manager cares about license counts and SLA tiers. A pure relational model forces every tenant's custom fields into the same rigid schema or into unmanageable EAV tables. JSONB columns let each organization define its own attributes on RFQ lines, supplier profiles, and PO metadata without schema migrations. Meanwhile, the core relationships (who ordered what from whom, at what price, with what approval) stay in properly constrained relational columns where referential integrity and indexing matter most.

---

## Schema Definition

### Organizations and Users

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    org_type        VARCHAR(20) NOT NULL CHECK (org_type IN ('buyer', 'supplier', 'both')),
    tax_id          VARCHAR(50),
    duns_number     VARCHAR(13),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('pending', 'active', 'suspended', 'deactivated')),
    -- JSONB: website, logo_url, social links, industry, locale preferences,
    -- onboarding progress, tenant-specific config
    profile         JSONB NOT NULL DEFAULT '{}',
    -- JSONB: array of address objects with type, lines, city, country, is_primary
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- JSONB: tenant-level settings - approval thresholds, notification prefs, integrations config
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_type ON organizations(org_type);
CREATE INDEX idx_org_profile ON organizations USING GIN(profile jsonb_path_ops);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('invited', 'active', 'suspended', 'deactivated')),
    roles           VARCHAR(50)[] NOT NULL DEFAULT '{}',
    -- JSONB: phone, job_title, department, avatar_url, notification preferences,
    -- dashboard layout, timezone, locale
    profile         JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_roles ON users USING GIN(roles);
```

### Supplier Management

```sql
CREATE TABLE supplier_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL UNIQUE REFERENCES organizations(id),
    verified        BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    esg_score       NUMERIC(5, 2),
    sustainability_tier VARCHAR(20)
                    CHECK (sustainability_tier IN ('bronze', 'silver', 'gold', 'platinum')),
    -- JSONB: business_category, employee_count, annual_revenue, year_established,
    -- certifications (ISO, etc.), manufacturing_capabilities, lead_time_ranges,
    -- payment_terms_accepted, minimum_order_values, geographic_coverage
    capabilities    JSONB NOT NULL DEFAULT '{}',
    -- JSONB: array of {unspsc_code, category_name, is_primary, subcategories[]}
    categories      JSONB NOT NULL DEFAULT '[]',
    -- JSONB: array of {doc_type, file_url, file_name, expires_at, verified, verified_by}
    compliance_docs JSONB NOT NULL DEFAULT '[]',
    -- JSONB: esg details - carbon_footprint, diversity_certs, sustainability_reports[]
    esg_details     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sp_categories ON supplier_profiles USING GIN(categories jsonb_path_ops);
CREATE INDEX idx_sp_capabilities ON supplier_profiles USING GIN(capabilities jsonb_path_ops);
CREATE INDEX idx_sp_esg ON supplier_profiles(esg_score) WHERE esg_score IS NOT NULL;

CREATE TABLE supplier_ratings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier_profiles(id),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    purchase_order_id UUID,
    quality_score   NUMERIC(3, 1) CHECK (quality_score BETWEEN 1 AND 5),
    delivery_score  NUMERIC(3, 1) CHECK (delivery_score BETWEEN 1 AND 5),
    overall_score   NUMERIC(3, 1) CHECK (overall_score BETWEEN 1 AND 5),
    -- JSONB: communication_score, responsiveness, flexibility, custom rating dimensions per org
    detail_scores   JSONB NOT NULL DEFAULT '{}',
    comments        TEXT,
    rated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sr_supplier ON supplier_ratings(supplier_id);
```

### Sourcing Events (RFQ / RFP / RFI)

```sql
CREATE TABLE sourcing_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    event_type      VARCHAR(10) NOT NULL CHECK (event_type IN ('RFQ', 'RFP', 'RFI')),
    title           VARCHAR(300) NOT NULL,
    reference_number VARCHAR(50) NOT NULL UNIQUE,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'closed', 'evaluating', 'awarded', 'cancelled')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    published_at    TIMESTAMPTZ,
    response_deadline TIMESTAMPTZ,
    -- JSONB: detailed description, terms_and_conditions, evaluation_criteria[],
    -- required_certifications[], delivery_requirements, custom_questions[],
    -- ai_generated_spec (from plain-language input), attachments[]
    details         JSONB NOT NULL DEFAULT '{}',
    -- JSONB: buyer-defined scoring weights, evaluation methodology config
    evaluation_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_se_buyer ON sourcing_events(buyer_org_id);
CREATE INDEX idx_se_status ON sourcing_events(status);
CREATE INDEX idx_se_details ON sourcing_events USING GIN(details jsonb_path_ops);

CREATE TABLE sourcing_event_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    line_number     SMALLINT NOT NULL,
    item_description VARCHAR(500) NOT NULL,
    quantity        NUMERIC(14, 4) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    target_price    NUMERIC(18, 4),
    delivery_date   DATE,
    -- JSONB: unspsc_code, technical_specs{}, material_requirements{},
    -- quality_standards[], tolerance_ranges{}, custom_attributes per buyer org
    specifications  JSONB NOT NULL DEFAULT '{}',
    UNIQUE (sourcing_event_id, line_number)
);
CREATE INDEX idx_sel_event ON sourcing_event_lines(sourcing_event_id);
CREATE INDEX idx_sel_specs ON sourcing_event_lines USING GIN(specifications jsonb_path_ops);

CREATE TABLE sourcing_invitations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    invited_by      UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'viewed', 'accepted', 'declined')),
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    responded_at    TIMESTAMPTZ,
    UNIQUE (sourcing_event_id, supplier_org_id)
);
```

### Bids and Auctions

```sql
CREATE TABLE bids (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    submitted_by    UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'submitted', 'withdrawn', 'awarded', 'rejected')),
    total_amount    NUMERIC(18, 4),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    submitted_at    TIMESTAMPTZ,
    -- JSONB: notes, payment_terms_offered, delivery_schedule, warranty_terms,
    -- custom_question_responses[], attachments[], discount_tiers[]
    bid_details     JSONB NOT NULL DEFAULT '{}',
    -- JSONB: ai_score, ai_confidence, scoring_breakdown{}, evaluator_notes[]
    evaluation      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (sourcing_event_id, supplier_org_id)
);
CREATE INDEX idx_bids_event ON bids(sourcing_event_id);
CREATE INDEX idx_bids_supplier ON bids(supplier_org_id);

CREATE TABLE bid_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bid_id          UUID NOT NULL REFERENCES bids(id),
    event_line_id   UUID NOT NULL REFERENCES sourcing_event_lines(id),
    unit_price      NUMERIC(18, 4) NOT NULL,
    quantity        NUMERIC(14, 4) NOT NULL,
    lead_time_days  INTEGER,
    -- JSONB: volume_discounts[], alternative_options[], notes, compliance_responses{}
    line_details    JSONB NOT NULL DEFAULT '{}'
);
CREATE INDEX idx_bl_bid ON bid_lines(bid_id);

CREATE TABLE auctions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    auction_type    VARCHAR(20) NOT NULL CHECK (auction_type IN ('reverse', 'forward', 'dutch', 'japanese')),
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled', 'active', 'extended', 'closed', 'cancelled')),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    -- JSONB: auto_extend config, min_decrement, visibility_rules, lot_structure,
    -- reserve_price, bid_increment_rules{}
    rules           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE auction_bids (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    auction_id      UUID NOT NULL REFERENCES auctions(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    amount          NUMERIC(18, 4) NOT NULL,
    rank            SMALLINT,
    placed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ab_auction ON auction_bids(auction_id, placed_at);
```

### Purchase Orders and Three-Way Matching

```sql
CREATE TABLE purchase_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    sourcing_event_id UUID REFERENCES sourcing_events(id),
    bid_id          UUID REFERENCES bids(id),
    po_number       VARCHAR(50) NOT NULL UNIQUE,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'pending_approval', 'approved', 'sent',
                                      'acknowledged', 'partially_received', 'received',
                                      'closed', 'cancelled')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18, 4),
    tax_amount      NUMERIC(18, 4),
    total_amount    NUMERIC(18, 4),
    created_by      UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    -- JSONB: payment_terms, incoterms, ship_to_address{}, bill_to_address{},
    -- expected_delivery, special_instructions, cxml_punchout_data{},
    -- edi_references{}, custom_fields per org
    terms           JSONB NOT NULL DEFAULT '{}',
    -- JSONB: approval chain snapshot, approval_history[], current_step
    approval_state  JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_po_buyer ON purchase_orders(buyer_org_id);
CREATE INDEX idx_po_supplier ON purchase_orders(supplier_org_id);
CREATE INDEX idx_po_status ON purchase_orders(status);

CREATE TABLE purchase_order_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_orders(id),
    line_number     SMALLINT NOT NULL,
    item_description VARCHAR(500) NOT NULL,
    quantity        NUMERIC(14, 4) NOT NULL,
    unit_price      NUMERIC(18, 4) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    line_total      NUMERIC(18, 4) NOT NULL,
    tax_rate        NUMERIC(5, 2) DEFAULT 0,
    -- JSONB: unspsc_code, specs{}, delivery_schedule, custom_attributes{}
    line_details    JSONB NOT NULL DEFAULT '{}',
    UNIQUE (purchase_order_id, line_number)
);
CREATE INDEX idx_pol_po ON purchase_order_lines(purchase_order_id);

CREATE TABLE goods_receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_orders(id),
    received_by     UUID NOT NULL REFERENCES users(id),
    receipt_number  VARCHAR(50) NOT NULL UNIQUE,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- JSONB: notes, warehouse_location, quality_inspection_result{}, photos[]
    receipt_details JSONB NOT NULL DEFAULT '{}'
);
CREATE INDEX idx_gr_po ON goods_receipts(purchase_order_id);

CREATE TABLE goods_receipt_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goods_receipt_id UUID NOT NULL REFERENCES goods_receipts(id),
    po_line_id      UUID NOT NULL REFERENCES purchase_order_lines(id),
    quantity_received NUMERIC(14, 4) NOT NULL,
    quantity_rejected NUMERIC(14, 4) DEFAULT 0,
    -- JSONB: rejection_reason, inspection_notes, serial_numbers[], lot_numbers[]
    line_details    JSONB NOT NULL DEFAULT '{}'
);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    purchase_order_id UUID REFERENCES purchase_orders(id),
    invoice_number  VARCHAR(100) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'received'
                    CHECK (status IN ('received', 'matched', 'disputed', 'approved', 'paid', 'cancelled')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18, 4) NOT NULL,
    tax_amount      NUMERIC(18, 4),
    total_amount    NUMERIC(18, 4) NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE,
    match_status    VARCHAR(20) DEFAULT 'unmatched'
                    CHECK (match_status IN ('unmatched', 'two_way', 'three_way', 'exception')),
    -- JSONB: payment_details{}, tax_breakdown[], discrepancies[], ocr_extracted_data{},
    -- edi_reference{}, ubl_document_id, peppol_endpoint
    invoice_metadata JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_org_id, invoice_number)
);
CREATE INDEX idx_inv_po ON invoices(purchase_order_id);
CREATE INDEX idx_inv_status ON invoices(status);
CREATE INDEX idx_inv_meta ON invoices USING GIN(invoice_metadata jsonb_path_ops);

CREATE TABLE invoice_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    po_line_id      UUID REFERENCES purchase_order_lines(id),
    description     VARCHAR(500) NOT NULL,
    quantity        NUMERIC(14, 4) NOT NULL,
    unit_price      NUMERIC(18, 4) NOT NULL,
    line_total      NUMERIC(18, 4) NOT NULL,
    tax_rate        NUMERIC(5, 2) DEFAULT 0,
    -- JSONB: tax_code, gl_account, cost_center, custom_coding{}
    line_details    JSONB NOT NULL DEFAULT '{}'
);
CREATE INDEX idx_invl_invoice ON invoice_lines(invoice_id);
```

### Contracts, Budgets, and Spend

```sql
CREATE TABLE contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    contract_number VARCHAR(50) NOT NULL UNIQUE,
    title           VARCHAR(300) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'expired', 'terminated', 'renewed')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    total_value     NUMERIC(18, 4),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB: terms{}, renewal_config{}, alert_config{}, sla_terms{},
    -- pricing_schedules[], volume_commitments[], ai_risk_flags[],
    -- amendment_history[], attached_files[]
    contract_details JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contracts_buyer ON contracts(buyer_org_id);
CREATE INDEX idx_contracts_end ON contracts(end_date);
CREATE INDEX idx_contracts_details ON contracts USING GIN(contract_details jsonb_path_ops);

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(200) NOT NULL,
    fiscal_year     SMALLINT NOT NULL,
    category        VARCHAR(100),
    allocated_amount NUMERIC(18, 4) NOT NULL,
    spent_amount    NUMERIC(18, 4) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    -- JSONB: breakdown by quarter, department allocations, approval_thresholds{},
    -- rollover_policy, alerts_config{}
    budget_config   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budgets_org ON budgets(organization_id, fiscal_year);

CREATE TABLE spend_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    purchase_order_id UUID REFERENCES purchase_orders(id),
    invoice_id      UUID REFERENCES invoices(id),
    category        VARCHAR(100),
    unspsc_code     VARCHAR(20),
    business_unit   VARCHAR(100),
    amount          NUMERIC(18, 4) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    amount_usd      NUMERIC(18, 4),
    is_maverick     BOOLEAN NOT NULL DEFAULT false,
    spend_date      DATE NOT NULL,
    -- JSONB: cost_center, gl_account, project_code, department,
    -- maverick_violation_details{}, policy_id, custom_dimensions{}
    dimensions      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (spend_date);

-- Create annual partitions
CREATE TABLE spend_records_2025 PARTITION OF spend_records
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE spend_records_2026 PARTITION OF spend_records
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
CREATE TABLE spend_records_2027 PARTITION OF spend_records
    FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');

CREATE INDEX idx_spend_org_date ON spend_records(organization_id, spend_date);
CREATE INDEX idx_spend_dims ON spend_records USING GIN(dimensions jsonb_path_ops);
CREATE INDEX idx_spend_maverick ON spend_records(is_maverick) WHERE is_maverick = true;
```

### Approval Workflows and Audit

```sql
CREATE TABLE approval_chains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(200) NOT NULL,
    document_type   VARCHAR(30) NOT NULL
                    CHECK (document_type IN ('purchase_order', 'invoice', 'sourcing_event', 'contract')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: steps[] with {step_order, approver_user_id or approver_role,
    -- min_amount, max_amount, auto_approve_below, escalation_config{}}
    chain_config    JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ac_org ON approval_chains(organization_id, document_type);

CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chain_id        UUID NOT NULL REFERENCES approval_chains(id),
    document_type   VARCHAR(30) NOT NULL,
    document_id     UUID NOT NULL,
    current_step    SMALLINT NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'escalated')),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decided_by      UUID REFERENCES users(id),
    decided_at      TIMESTAMPTZ,
    -- JSONB: step_history[], comments, delegation_info{}
    approval_data   JSONB NOT NULL DEFAULT '{}'
);
CREATE INDEX idx_ar_doc ON approval_requests(document_type, document_id);

CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id),
    action      VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id   UUID NOT NULL,
    changes     JSONB NOT NULL DEFAULT '{}',
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2025 PARTITION OF audit_log
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
CREATE TABLE audit_log_2026 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_changes ON audit_log USING GIN(changes jsonb_path_ops);
```

### Integration and EDI/cXML Support

```sql
CREATE TABLE integration_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    integration_type VARCHAR(30) NOT NULL
                    CHECK (integration_type IN ('erp_sap', 'erp_oracle', 'erp_netsuite', 'erp_workday',
                                                 'cxml_punchout', 'edi_x12', 'edi_edifact', 'peppol', 'custom_api')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: connection_url, credentials_ref (vault path, never raw secrets),
    -- field_mappings{}, sync_schedule, error_handling_config{},
    -- cxml_identity, peppol_endpoint_id, edi_interchange_config{}
    config          JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ic_org ON integration_configs(organization_id);
```

---

## JSONB Query Examples

```sql
-- Find suppliers certified for ISO 9001
SELECT sp.*, o.name
FROM supplier_profiles sp
JOIN organizations o ON o.id = sp.organization_id
WHERE sp.capabilities @> '{"certifications": ["ISO 9001"]}';

-- Find sourcing events requiring specific delivery terms
SELECT * FROM sourcing_events
WHERE details @> '{"delivery_requirements": {"incoterms": "DDP"}}';

-- Query spend by custom cost center dimension
SELECT SUM(amount) as total, dimensions->>'cost_center' as cost_center
FROM spend_records
WHERE organization_id = :org_id
  AND spend_date BETWEEN :start AND :end
GROUP BY dimensions->>'cost_center';

-- Find contracts with AI-flagged risks
SELECT c.*, c.contract_details->'ai_risk_flags' as risks
FROM contracts c
WHERE jsonb_array_length(c.contract_details->'ai_risk_flags') > 0
  AND c.buyer_org_id = :org_id;
```

---

## Trade-offs

**Strengths:**
- Combines the referential integrity of relational modeling with the flexibility of document stores. Core financial data (amounts, statuses, foreign keys) is fully constrained while tenant-specific extensions live in JSONB without schema changes.
- GIN indexes on JSONB columns provide efficient querying for containment and path-based lookups, avoiding full table scans.
- Eliminates the need for EAV tables or per-tenant schema branching. Each organization's custom fields are stored inline with the entity they describe.
- Single database engine (PostgreSQL) simplifies operations, backups, and consistency guarantees compared to polyglot persistence.
- JSONB columns naturally accommodate data from external systems (cXML, EDI, UBL) where the structure varies by trading partner.
- Partitioning on time-series tables (spend_records, audit_log) keeps query performance stable as data grows.

**Weaknesses:**
- JSONB columns lack schema enforcement at the database level. Application-layer validation (JSON Schema) is required to prevent malformed data. A typo in a key name goes undetected by the database.
- JSONB updates rewrite the entire column value, not individual keys. Frequent partial updates to large JSONB documents can cause write amplification and TOAST fragmentation.
- Reporting and BI tools may struggle to query deeply nested JSONB structures. Materialized views or ETL pipelines may be needed to flatten JSONB data for analytics.
- Foreign key references cannot point into JSONB arrays or objects. If a JSONB array contains user IDs, the database cannot enforce their existence.
- GIN indexes on large JSONB columns consume significant disk space and slow down writes.

## Scalability Considerations

- Use table partitioning for high-volume tables (spend_records, audit_log, auction_bids).
- Keep JSONB documents reasonably sized (under 1 MB). If a JSONB column grows unbounded (e.g., amendment_history on a long-running contract), consider moving historical entries to a separate table.
- Monitor GIN index sizes and consider expression indexes (e.g., on specific JSONB paths) instead of full-document GIN indexes for frequently queried paths.
- Use `jsonb_path_ops` operator class for containment queries (@>) which creates smaller indexes than the default.

## Migration Path

This model is the easiest to evolve from a normalized relational schema (Suggestion 1). Start with all relational columns and introduce JSONB columns only where tenant customization or external data variability is needed. The relational columns can be promoted to JSONB or vice versa as requirements stabilize. If a JSONB field turns out to be queried or joined on frequently, promote it to a dedicated relational column with a backfill migration.
