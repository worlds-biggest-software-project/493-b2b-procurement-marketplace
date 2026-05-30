# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where every state change in the procurement lifecycle is captured as an immutable domain event. The write side processes commands and appends events to an append-only event store. The read side builds purpose-built projections (materialized views) optimized for each query pattern. Commands and queries flow through separate pipelines (CQRS).

## Why This Suits B2B Procurement

Procurement is an inherently event-driven domain. An RFQ is created, published, receives bids, gets evaluated, and results in an award. A purchase order moves through draft, approval, acknowledgment, partial receipt, and closure. Every transition is a meaningful business event that auditors, compliance teams, and analytics engines need to inspect. Event sourcing captures this complete history as a first-class concept rather than deriving it from mutable row updates. The built-in audit trail satisfies regulatory requirements without a separate audit log. Three-way matching becomes a matter of correlating events across PO, goods receipt, and invoice streams. Real-time spend analytics and maverick detection can consume the event stream directly.

---

## Event Store Schema

The event store is the single source of truth. All domain events are appended here.

```sql
-- Core event store table (append-only)
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,   -- 'SourcingEvent', 'PurchaseOrder', 'Invoice', etc.
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,   -- 'RfqCreated', 'BidSubmitted', 'PoApproved', etc.
    event_version   INTEGER NOT NULL,        -- Monotonically increasing per aggregate
    payload         JSONB NOT NULL,          -- Full event data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, user_id, ip
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);

-- Optimized for replaying a single aggregate
CREATE INDEX idx_es_aggregate ON event_store(aggregate_type, aggregate_id, event_version);
-- Optimized for building projections from a global position
CREATE INDEX idx_es_created ON event_store(created_at);
-- Optimized for finding events by type across all aggregates
CREATE INDEX idx_es_event_type ON event_store(event_type);

-- Snapshot table for aggregates with long event histories
CREATE TABLE event_snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);
```

---

## Domain Events

### Organization & User Aggregate

```
OrganizationRegistered    { org_id, name, org_type, tax_id, currency }
OrganizationVerified      { org_id, verified_by, verified_at }
OrganizationSuspended     { org_id, reason }
UserCreated               { user_id, org_id, email, first_name, last_name, role }
UserRoleAssigned          { user_id, role_id }
UserDeactivated           { user_id, reason }
```

### Supplier Profile Aggregate

```
SupplierProfileCreated    { supplier_id, org_id, categories[], certifications[] }
SupplierCategoryAdded     { supplier_id, unspsc_code, category_name }
ComplianceDocUploaded     { supplier_id, doc_type, file_url, expires_at }
ComplianceDocVerified     { supplier_id, doc_id, verified_by }
EsgScoreUpdated           { supplier_id, esg_score, sustainability_tier }
SupplierRated             { supplier_id, buyer_org_id, po_id, quality, delivery, communication, overall }
```

### Sourcing Event Aggregate

```
RfqCreated                { event_id, buyer_org_id, created_by, event_type, title, currency }
RfqLineAdded              { event_id, line_id, line_number, description, quantity, uom, target_price }
RfqPublished              { event_id, published_at, response_deadline }
SupplierInvited           { event_id, supplier_org_id, invited_by }
InvitationAccepted        { event_id, supplier_org_id }
InvitationDeclined        { event_id, supplier_org_id, reason }
BidSubmitted              { event_id, bid_id, supplier_org_id, total_amount, lines[] }
BidWithdrawn              { event_id, bid_id, reason }
BidEvaluated              { event_id, bid_id, scores, ai_recommendation }
EventAwarded              { event_id, winning_bid_id, supplier_org_id, awarded_by }
EventCancelled            { event_id, reason, cancelled_by }
```

### Auction Aggregate

```
AuctionCreated            { auction_id, sourcing_event_id, auction_type, start_time, end_time }
AuctionStarted            { auction_id }
AuctionBidPlaced          { auction_id, supplier_org_id, amount, rank }
AuctionExtended           { auction_id, new_end_time, reason }
AuctionClosed             { auction_id, winning_supplier_id, winning_amount }
```

### Purchase Order Aggregate

```
PoCreated                 { po_id, buyer_org_id, supplier_org_id, sourcing_event_id, lines[] }
PoLineAdded               { po_id, line_id, description, quantity, unit_price }
PoSubmittedForApproval    { po_id, submitted_by }
PoApproved                { po_id, approved_by, step_id }
PoRejected                { po_id, rejected_by, reason }
PoSentToSupplier          { po_id, sent_at }
PoAcknowledgedBySupplier  { po_id, acknowledged_by }
GoodsReceived             { po_id, receipt_id, lines[{po_line_id, qty_received, qty_rejected}] }
PoClosed                  { po_id, closed_by }
PoCancelled               { po_id, reason, cancelled_by }
```

### Invoice Aggregate

```
InvoiceReceived           { invoice_id, supplier_org_id, buyer_org_id, po_id, invoice_number, lines[], total }
InvoiceTwoWayMatched      { invoice_id, po_id, match_result }
InvoiceThreeWayMatched    { invoice_id, po_id, receipt_id, match_result }
InvoiceMatchException     { invoice_id, discrepancies[] }
InvoiceDisputed           { invoice_id, reason, disputed_by }
InvoiceApproved           { invoice_id, approved_by }
InvoicePaid               { invoice_id, payment_date, payment_reference }
```

### Contract Aggregate

```
ContractCreated           { contract_id, buyer_org_id, supplier_org_id, title, start_date, end_date, value }
ContractActivated         { contract_id }
ContractAmended           { contract_id, changes[], amended_by }
ContractExpiryAlertSent   { contract_id, days_remaining }
ContractRenewed           { contract_id, new_end_date, new_value }
ContractTerminated        { contract_id, reason, terminated_by }
```

### Spend & Compliance Events

```
SpendRecorded             { spend_id, org_id, supplier_org_id, amount, category, unspsc, business_unit }
MaverickSpendDetected     { spend_id, policy_id, violation_type, amount }
BudgetAllocated           { budget_id, org_id, fiscal_year, category, amount }
BudgetConsumed            { budget_id, amount, po_id }
BudgetExceeded            { budget_id, attempted_amount, remaining }
```

---

## Command Handlers

```python
# Pseudocode illustrating command handler pattern

class SubmitBidCommandHandler:
    def handle(self, cmd: SubmitBidCommand):
        # 1. Load aggregate from event stream
        sourcing_event = self.repository.load(SourcingEventAggregate, cmd.sourcing_event_id)

        # 2. Validate business rules
        sourcing_event.validate_bid_submission(
            supplier_org_id=cmd.supplier_org_id,
            lines=cmd.lines,
            deadline=sourcing_event.response_deadline
        )

        # 3. Apply domain event
        sourcing_event.apply(BidSubmitted(
            event_id=cmd.sourcing_event_id,
            bid_id=uuid4(),
            supplier_org_id=cmd.supplier_org_id,
            total_amount=cmd.total_amount,
            lines=cmd.lines
        ))

        # 4. Persist new events
        self.repository.save(sourcing_event)


class ApprovePurchaseOrderHandler:
    def handle(self, cmd: ApprovePurchaseOrderCommand):
        po = self.repository.load(PurchaseOrderAggregate, cmd.po_id)
        approval_chain = self.approval_service.get_chain(po.buyer_org_id, 'purchase_order')

        po.approve(
            approved_by=cmd.approver_id,
            chain=approval_chain,
            step_id=cmd.step_id
        )

        self.repository.save(po)


class RecordThreeWayMatchHandler:
    def handle(self, cmd: MatchInvoiceCommand):
        invoice = self.repository.load(InvoiceAggregate, cmd.invoice_id)
        po = self.repository.load(PurchaseOrderAggregate, invoice.po_id)
        receipts = self.receipt_service.get_receipts_for_po(invoice.po_id)

        match_result = self.matching_engine.three_way_match(invoice, po, receipts)

        if match_result.is_clean:
            invoice.apply(InvoiceThreeWayMatched(...))
        else:
            invoice.apply(InvoiceMatchException(discrepancies=match_result.discrepancies))

        self.repository.save(invoice)
```

---

## Read-Side Projections

Projections are rebuilt from the event stream and optimized for specific query patterns.

```sql
-- Projection: Active Sourcing Events (for buyer dashboard)
CREATE TABLE proj_sourcing_events (
    id              UUID PRIMARY KEY,
    buyer_org_id    UUID NOT NULL,
    buyer_org_name  VARCHAR(255),
    event_type      VARCHAR(10) NOT NULL,
    title           VARCHAR(300) NOT NULL,
    reference_number VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    currency        CHAR(3) NOT NULL,
    line_count      INTEGER NOT NULL DEFAULT 0,
    bid_count       INTEGER NOT NULL DEFAULT 0,
    published_at    TIMESTAMPTZ,
    response_deadline TIMESTAMPTZ,
    last_updated    TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_se_buyer ON proj_sourcing_events(buyer_org_id, status);

-- Projection: Bid Comparison (for side-by-side evaluation)
CREATE TABLE proj_bid_comparison (
    sourcing_event_id UUID NOT NULL,
    event_line_id   UUID NOT NULL,
    line_description VARCHAR(500),
    bid_id          UUID NOT NULL,
    supplier_org_id UUID NOT NULL,
    supplier_name   VARCHAR(255),
    unit_price      NUMERIC(18, 4),
    quantity        NUMERIC(14, 4),
    lead_time_days  INTEGER,
    ai_score        NUMERIC(5, 2),
    PRIMARY KEY (sourcing_event_id, event_line_id, bid_id)
);

-- Projection: Purchase Order Tracker (for buyer/supplier PO list)
CREATE TABLE proj_purchase_orders (
    id              UUID PRIMARY KEY,
    po_number       VARCHAR(50) NOT NULL,
    buyer_org_id    UUID NOT NULL,
    buyer_org_name  VARCHAR(255),
    supplier_org_id UUID NOT NULL,
    supplier_org_name VARCHAR(255),
    status          VARCHAR(30) NOT NULL,
    total_amount    NUMERIC(18, 4),
    currency        CHAR(3),
    line_count      INTEGER DEFAULT 0,
    received_pct    NUMERIC(5, 2) DEFAULT 0,
    invoiced_pct    NUMERIC(5, 2) DEFAULT 0,
    match_status    VARCHAR(20),
    created_at      TIMESTAMPTZ,
    last_updated    TIMESTAMPTZ
);
CREATE INDEX idx_proj_po_buyer ON proj_purchase_orders(buyer_org_id, status);
CREATE INDEX idx_proj_po_supplier ON proj_purchase_orders(supplier_org_id, status);

-- Projection: Spend Dashboard (pre-aggregated for analytics)
CREATE TABLE proj_spend_by_category (
    org_id          UUID NOT NULL,
    fiscal_year     SMALLINT NOT NULL,
    fiscal_month    SMALLINT NOT NULL,
    category        VARCHAR(100) NOT NULL,
    supplier_org_id UUID,
    business_unit   VARCHAR(100),
    total_amount    NUMERIC(18, 4) NOT NULL DEFAULT 0,
    maverick_amount NUMERIC(18, 4) NOT NULL DEFAULT 0,
    transaction_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (org_id, fiscal_year, fiscal_month, category, supplier_org_id, business_unit)
);

-- Projection: Supplier Directory (for search and discovery)
CREATE TABLE proj_supplier_directory (
    supplier_id     UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    categories      TEXT[] NOT NULL DEFAULT '{}',
    unspsc_codes    VARCHAR(20)[] NOT NULL DEFAULT '{}',
    esg_score       NUMERIC(5, 2),
    sustainability_tier VARCHAR(20),
    avg_quality     NUMERIC(3, 1),
    avg_delivery    NUMERIC(3, 1),
    avg_overall     NUMERIC(3, 1),
    rating_count    INTEGER DEFAULT 0,
    verified        BOOLEAN DEFAULT false,
    certifications  TEXT[] DEFAULT '{}',
    last_updated    TIMESTAMPTZ
);
CREATE INDEX idx_proj_sd_categories ON proj_supplier_directory USING GIN(categories);
CREATE INDEX idx_proj_sd_unspsc ON proj_supplier_directory USING GIN(unspsc_codes);

-- Projection: Contract Alerts (for expiry notifications)
CREATE TABLE proj_contracts (
    id              UUID PRIMARY KEY,
    buyer_org_id    UUID NOT NULL,
    supplier_org_id UUID NOT NULL,
    supplier_name   VARCHAR(255),
    contract_number VARCHAR(50),
    title           VARCHAR(300),
    status          VARCHAR(20),
    start_date      DATE,
    end_date        DATE,
    total_value     NUMERIC(18, 4),
    days_until_expiry INTEGER,
    last_updated    TIMESTAMPTZ
);
CREATE INDEX idx_proj_contracts_expiry ON proj_contracts(buyer_org_id, end_date);

-- Projection: Auction Live View (for real-time bidding UI)
CREATE TABLE proj_auction_live (
    auction_id      UUID PRIMARY KEY,
    sourcing_event_id UUID NOT NULL,
    auction_type    VARCHAR(20),
    status          VARCHAR(20),
    current_best    NUMERIC(18, 4),
    bid_count       INTEGER DEFAULT 0,
    end_time        TIMESTAMPTZ,
    last_bid_at     TIMESTAMPTZ
);
```

---

## Trade-offs

**Strengths:**
- Complete, immutable audit trail is intrinsic to the model. Every state change is recorded with who, when, and why. This satisfies compliance and regulatory requirements without a separate audit system.
- Time-travel queries are trivial: replay events to any point to see the exact state of a PO, RFQ, or invoice at that moment.
- Read-side projections can be independently optimized, rebuilt, and scaled without touching the write model.
- Natural fit for real-time event streaming (e.g., Kafka) to power spend analytics, maverick detection, and notification services.
- Supports complex workflows like three-way matching by correlating events across aggregates.
- New read models (e.g., a new analytics dashboard) can be added by replaying existing events without schema changes.

**Weaknesses:**
- Significantly higher complexity than a CRUD relational model. The team needs experience with event sourcing patterns, eventual consistency, and projection management.
- Eventual consistency between the event store and projections means read models may lag behind writes. This can be confusing for users who expect immediate consistency after saving.
- Event schema evolution requires careful versioning. Changing an event's payload format must be backward-compatible or require an upcaster.
- Debugging is harder: the current state is not directly visible in a single table row -- it must be reconstructed from events.
- Query patterns not anticipated at design time require building a new projection and replaying the event history.

## Scalability Considerations

- The event store is append-only and partitionable by aggregate_type or time range.
- Projections can be hosted on separate read replicas or entirely different databases (e.g., Elasticsearch for supplier search, TimescaleDB for spend analytics).
- Event consumption can be parallelized by partitioning on aggregate_id.
- Snapshots should be taken for aggregates with more than ~100 events (e.g., long-running contracts with many amendments).
- Consider a dedicated event streaming platform (Kafka, EventStoreDB) for high-throughput scenarios like live auctions.

## Migration Path

Event sourcing can be adopted incrementally. Start with one aggregate (e.g., Purchase Orders) where the audit trail is most valuable, and keep the rest in a traditional relational model. Events can be projected into the same relational tables used by the CRUD system, allowing gradual migration without a big-bang rewrite. The event store also serves as a foundation for building new AI features: the complete event history is ideal training data for spend pattern models and anomaly detection.
