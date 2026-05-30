# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity gets its own table with strict foreign keys, check constraints, and covering indexes. This is the most battle-tested pattern for transactional procurement systems where referential integrity, audit trails, and ACID compliance are non-negotiable.

## Why This Suits B2B Procurement

Procurement workflows are inherently transactional: a purchase order references a supplier, an RFQ, approved line items, and a budget. Three-way matching (PO, goods receipt, invoice) demands that relationships between documents are enforced at the database level. A normalized schema prevents data anomalies that could lead to duplicate payments, orphaned invoices, or mismatched quantities. PostgreSQL's mature support for row-level security, partitioning, and full-text search makes it well-suited for multi-tenant procurement platforms.

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
    website         VARCHAR(500),
    logo_url        VARCHAR(500),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('pending', 'active', 'suspended', 'deactivated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_addresses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    address_type    VARCHAR(20) NOT NULL CHECK (address_type IN ('billing', 'shipping', 'headquarters', 'warehouse')),
    line_1          VARCHAR(255) NOT NULL,
    line_2          VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_addresses_org ON organization_addresses(organization_id);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(320) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(30),
    job_title       VARCHAR(150),
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('invited', 'active', 'suspended', 'deactivated')),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users(organization_id);

CREATE TABLE roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL,
    description TEXT,
    org_scoped  BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    role_id UUID NOT NULL REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);
```

### Supplier Management and Compliance

```sql
CREATE TABLE supplier_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL UNIQUE REFERENCES organizations(id),
    business_category   VARCHAR(100),
    employee_count      INTEGER,
    annual_revenue      NUMERIC(18, 2),
    revenue_currency    CHAR(3) DEFAULT 'USD',
    year_established    SMALLINT,
    certification_iso   VARCHAR(50)[],
    esg_score           NUMERIC(5, 2),
    sustainability_tier VARCHAR(20) CHECK (sustainability_tier IN ('bronze', 'silver', 'gold', 'platinum')),
    verified_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier_profiles(id),
    unspsc_code     VARCHAR(20) NOT NULL,
    category_name   VARCHAR(200) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false
);
CREATE INDEX idx_supp_cat_supplier ON supplier_categories(supplier_id);
CREATE INDEX idx_supp_cat_unspsc ON supplier_categories(unspsc_code);

CREATE TABLE compliance_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    doc_type        VARCHAR(50) NOT NULL,
    file_name       VARCHAR(255) NOT NULL,
    file_url        VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    expires_at      DATE,
    verified        BOOLEAN NOT NULL DEFAULT false,
    verified_by     UUID REFERENCES users(id),
    verified_at     TIMESTAMPTZ,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compliance_org ON compliance_documents(organization_id);

CREATE TABLE supplier_ratings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier_profiles(id),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    purchase_order_id UUID,  -- FK added after PO table creation
    quality_score   NUMERIC(3, 1) CHECK (quality_score BETWEEN 1 AND 5),
    delivery_score  NUMERIC(3, 1) CHECK (delivery_score BETWEEN 1 AND 5),
    comm_score      NUMERIC(3, 1) CHECK (comm_score BETWEEN 1 AND 5),
    overall_score   NUMERIC(3, 1) CHECK (overall_score BETWEEN 1 AND 5),
    comments        TEXT,
    rated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ratings_supplier ON supplier_ratings(supplier_id);
```

### Sourcing: RFQ / RFP / RFI and Bidding

```sql
CREATE TABLE sourcing_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id    UUID NOT NULL REFERENCES organizations(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    event_type      VARCHAR(10) NOT NULL CHECK (event_type IN ('RFQ', 'RFP', 'RFI')),
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    reference_number VARCHAR(50) NOT NULL UNIQUE,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'closed', 'evaluating', 'awarded', 'cancelled')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    published_at    TIMESTAMPTZ,
    response_deadline TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sourcing_buyer ON sourcing_events(buyer_org_id);
CREATE INDEX idx_sourcing_status ON sourcing_events(status);

CREATE TABLE sourcing_event_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    line_number     SMALLINT NOT NULL,
    item_description VARCHAR(500) NOT NULL,
    unspsc_code     VARCHAR(20),
    quantity        NUMERIC(14, 4) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    target_price    NUMERIC(18, 4),
    delivery_date   DATE,
    specifications  TEXT,
    UNIQUE (sourcing_event_id, line_number)
);
CREATE INDEX idx_sel_event ON sourcing_event_lines(sourcing_event_id);

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

CREATE TABLE bids (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    submitted_by    UUID NOT NULL REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'submitted', 'withdrawn', 'awarded', 'rejected')),
    total_amount    NUMERIC(18, 4),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    notes           TEXT,
    submitted_at    TIMESTAMPTZ,
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
    notes           TEXT
);
CREATE INDEX idx_bid_lines_bid ON bid_lines(bid_id);
```

### eAuction Support

```sql
CREATE TABLE auctions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    auction_type    VARCHAR(20) NOT NULL CHECK (auction_type IN ('reverse', 'forward', 'dutch', 'japanese')),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    auto_extend     BOOLEAN NOT NULL DEFAULT true,
    extend_minutes  SMALLINT DEFAULT 5,
    min_decrement   NUMERIC(18, 4),
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled', 'active', 'extended', 'closed', 'cancelled')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE auction_bids (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    auction_id  UUID NOT NULL REFERENCES auctions(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    amount      NUMERIC(18, 4) NOT NULL,
    rank        SMALLINT,
    placed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_auction_bids_auction ON auction_bids(auction_id);
CREATE INDEX idx_auction_bids_time ON auction_bids(placed_at);
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
                    CHECK (status IN ('draft', 'pending_approval', 'approved', 'sent', 'acknowledged',
                                      'partially_received', 'received', 'closed', 'cancelled')),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18, 4),
    tax_amount      NUMERIC(18, 4),
    total_amount    NUMERIC(18, 4),
    payment_terms   VARCHAR(50),
    ship_to_address_id UUID REFERENCES organization_addresses(id),
    bill_to_address_id UUID REFERENCES organization_addresses(id),
    expected_delivery DATE,
    created_by      UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
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
    unspsc_code     VARCHAR(20),
    quantity        NUMERIC(14, 4) NOT NULL,
    unit_price      NUMERIC(18, 4) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    line_total      NUMERIC(18, 4) NOT NULL,
    tax_rate        NUMERIC(5, 2) DEFAULT 0,
    UNIQUE (purchase_order_id, line_number)
);
CREATE INDEX idx_pol_po ON purchase_order_lines(purchase_order_id);

CREATE TABLE goods_receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_orders(id),
    received_by     UUID NOT NULL REFERENCES users(id),
    receipt_number  VARCHAR(50) NOT NULL UNIQUE,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    notes           TEXT
);
CREATE INDEX idx_gr_po ON goods_receipts(purchase_order_id);

CREATE TABLE goods_receipt_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goods_receipt_id UUID NOT NULL REFERENCES goods_receipts(id),
    po_line_id      UUID NOT NULL REFERENCES purchase_order_lines(id),
    quantity_received NUMERIC(14, 4) NOT NULL,
    quantity_rejected NUMERIC(14, 4) DEFAULT 0,
    notes           TEXT
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
    payment_date    DATE,
    match_status    VARCHAR(20) DEFAULT 'unmatched'
                    CHECK (match_status IN ('unmatched', 'two_way', 'three_way', 'exception')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_org_id, invoice_number)
);
CREATE INDEX idx_inv_po ON invoices(purchase_order_id);
CREATE INDEX idx_inv_status ON invoices(status);

CREATE TABLE invoice_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    po_line_id      UUID REFERENCES purchase_order_lines(id),
    description     VARCHAR(500) NOT NULL,
    quantity        NUMERIC(14, 4) NOT NULL,
    unit_price      NUMERIC(18, 4) NOT NULL,
    line_total      NUMERIC(18, 4) NOT NULL,
    tax_rate        NUMERIC(5, 2) DEFAULT 0
);
CREATE INDEX idx_invl_invoice ON invoice_lines(invoice_id);
```

### Approval Workflows

```sql
CREATE TABLE approval_chains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(200) NOT NULL,
    document_type   VARCHAR(30) NOT NULL CHECK (document_type IN ('purchase_order', 'invoice', 'sourcing_event', 'contract')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_chain_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    approval_chain_id UUID NOT NULL REFERENCES approval_chains(id),
    step_order      SMALLINT NOT NULL,
    approver_user_id UUID REFERENCES users(id),
    approver_role_id UUID REFERENCES roles(id),
    min_amount      NUMERIC(18, 4),
    max_amount      NUMERIC(18, 4),
    UNIQUE (approval_chain_id, step_order)
);

CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chain_id        UUID NOT NULL REFERENCES approval_chains(id),
    step_id         UUID NOT NULL REFERENCES approval_chain_steps(id),
    document_type   VARCHAR(30) NOT NULL,
    document_id     UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'escalated')),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decided_by      UUID REFERENCES users(id),
    decided_at      TIMESTAMPTZ,
    comments        TEXT
);
CREATE INDEX idx_appr_doc ON approval_requests(document_type, document_id);
```

### Contracts and Budgets

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
    renewal_type    VARCHAR(20) CHECK (renewal_type IN ('manual', 'auto', 'none')),
    alert_days_before_expiry SMALLINT DEFAULT 30,
    file_url        VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contracts_buyer ON contracts(buyer_org_id);
CREATE INDEX idx_contracts_end ON contracts(end_date);

CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(200) NOT NULL,
    fiscal_year     SMALLINT NOT NULL,
    category        VARCHAR(100),
    allocated_amount NUMERIC(18, 4) NOT NULL,
    spent_amount    NUMERIC(18, 4) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_budgets_org_year ON budgets(organization_id, fiscal_year);
```

### Spend Analytics and Audit

```sql
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_spend_org_date ON spend_records(organization_id, spend_date);
CREATE INDEX idx_spend_category ON spend_records(category);
CREATE INDEX idx_spend_maverick ON spend_records(is_maverick) WHERE is_maverick = true;

CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID REFERENCES users(id),
    action      VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id   UUID NOT NULL,
    old_values  JSONB,
    new_values  JSONB,
    ip_address  INET,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);
```

---

## Trade-offs

**Strengths:**
- Referential integrity enforced at the database level prevents orphaned records and data corruption.
- Well-understood by most engineering teams; vast ecosystem of ORMs, migration tools, and monitoring.
- Three-way matching logic maps naturally to JOINs across PO, goods receipt, and invoice tables.
- PostgreSQL's row-level security enables multi-tenant isolation without application-layer hacks.
- Strong audit trail via the `audit_log` table and TIMESTAMPTZ columns on every entity.

**Weaknesses:**
- Schema migrations for wide tables (e.g., adding custom fields per tenant) require ALTER TABLE and can be disruptive at scale.
- Deeply normalized models mean many JOINs for common read paths like "show me a PO with all its lines, receipts, and invoices."
- Flexible/custom attributes per organization are hard to model without resorting to EAV patterns or JSONB (see Suggestion 3).
- High-write auction scenarios may encounter contention on the `auction_bids` table without careful partitioning.

## Scalability Considerations

- Partition `spend_records` and `audit_log` by date range for time-series query performance.
- Use read replicas for analytics dashboards to avoid loading the OLTP primary.
- Consider materialized views for spend aggregation queries that power dashboards.
- Connection pooling (PgBouncer) is essential for multi-tenant workloads with many concurrent users.

## Migration Path

This schema works as the initial foundation. If performance or flexibility requirements grow, consider migrating specific domains (e.g., auction bidding, spend analytics) to specialized stores while keeping the relational core for transactional procurement workflows. The normalized schema also serves as the source of truth if you later adopt event sourcing (Suggestion 2) or add JSONB flexibility (Suggestion 3).
