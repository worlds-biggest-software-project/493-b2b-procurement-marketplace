# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A property graph model in Neo4j where organizations, sourcing events, purchase orders, contracts, and categories are nodes, and the relationships between them (SUPPLIES_TO, BID_ON, ISSUED_PO, RATED, CATEGORIZED_AS) are first-class edges with their own properties. This approach treats the procurement network as a connected graph rather than a collection of flat tables.

## Why a Graph Database Suits B2B Procurement

A B2B procurement marketplace is fundamentally a network problem. Buyers are connected to suppliers through RFQs, bids, purchase orders, invoices, contracts, and ratings. Supplier discovery ("find me a certified titanium fastener supplier in the EU who has delivered to companies in my industry with a quality rating above 4.0") is a multi-hop traversal that requires expensive self-joins in relational databases but is a natural graph query. The AI-native features described in the README -- embedding-based supplier matching, spend pattern anomaly detection, and contract risk analysis -- benefit from a model where relationships are explicit and traversable. Graph databases excel at answering questions like "what is the shortest supply chain path?", "which suppliers are shared across business units?", and "what category spend is concentrated in a single supplier (risk)?".

---

## Node Definitions

```cypher
// ============================================================
// ORGANIZATION NODES
// ============================================================

// Buyer or supplier organization
CREATE CONSTRAINT org_id_unique IF NOT EXISTS
FOR (o:Organization) REQUIRE o.id IS UNIQUE;

// Example node creation
CREATE (o:Organization {
    id: randomUUID(),
    name: 'Acme Manufacturing',
    orgType: 'buyer',           // 'buyer', 'supplier', 'both'
    taxId: 'US-12-3456789',
    dunsNumber: '123456789',
    defaultCurrency: 'USD',
    status: 'active',
    website: 'https://acme.example.com',
    country: 'US',
    city: 'Chicago',
    employeeCount: 2500,
    createdAt: datetime(),
    updatedAt: datetime()
})

// User within an organization
CREATE CONSTRAINT user_email_unique IF NOT EXISTS
FOR (u:User) REQUIRE u.email IS UNIQUE;

CREATE CONSTRAINT user_id_unique IF NOT EXISTS
FOR (u:User) REQUIRE u.id IS UNIQUE;

// ============================================================
// CATEGORY NODES (UNSPSC taxonomy)
// ============================================================

CREATE CONSTRAINT category_code_unique IF NOT EXISTS
FOR (c:Category) REQUIRE c.unspscCode IS UNIQUE;

// Categories form a hierarchical tree
// Segment > Family > Class > Commodity
CREATE (c:Category {
    unspscCode: '31160000',
    name: 'Nuts and Bolts',
    level: 'class',            // 'segment', 'family', 'class', 'commodity'
    parentCode: '31000000'
})

// ============================================================
// SOURCING EVENT NODES
// ============================================================

CREATE CONSTRAINT se_id_unique IF NOT EXISTS
FOR (se:SourcingEvent) REQUIRE se.id IS UNIQUE;

CREATE (se:SourcingEvent {
    id: randomUUID(),
    referenceNumber: 'RFQ-2026-001234',
    eventType: 'RFQ',          // 'RFQ', 'RFP', 'RFI'
    title: 'Titanium Fasteners Q3 2026',
    status: 'published',
    currency: 'USD',
    description: 'Seeking quotes for Grade 5 titanium hex bolts...',
    publishedAt: datetime('2026-05-15T09:00:00Z'),
    responseDeadline: datetime('2026-06-01T17:00:00Z'),
    evaluationCriteria: ['price', 'quality', 'lead_time', 'esg_score'],
    createdAt: datetime()
})

// Line items within a sourcing event
CREATE CONSTRAINT sel_id_unique IF NOT EXISTS
FOR (sl:SourcingLine) REQUIRE sl.id IS UNIQUE;

CREATE (sl:SourcingLine {
    id: randomUUID(),
    lineNumber: 1,
    itemDescription: 'M10 x 30mm Titanium Hex Bolt, Grade 5',
    quantity: 50000,
    unitOfMeasure: 'EA',
    targetPrice: 0.85,
    deliveryDate: date('2026-08-15'),
    specifications: '{"material": "Ti-6Al-4V", "finish": "plain", "thread": "coarse"}'
})

// ============================================================
// BID NODES
// ============================================================

CREATE CONSTRAINT bid_id_unique IF NOT EXISTS
FOR (b:Bid) REQUIRE b.id IS UNIQUE;

CREATE (b:Bid {
    id: randomUUID(),
    status: 'submitted',
    totalAmount: 42500.00,
    currency: 'USD',
    paymentTerms: 'Net 30',
    leadTimeDays: 21,
    warrantyMonths: 12,
    aiScore: 87.5,
    aiConfidence: 0.92,
    submittedAt: datetime(),
    createdAt: datetime()
})

// Bid line items
CREATE CONSTRAINT bl_id_unique IF NOT EXISTS
FOR (bl:BidLine) REQUIRE bl.id IS UNIQUE;

// ============================================================
// AUCTION NODES
// ============================================================

CREATE CONSTRAINT auction_id_unique IF NOT EXISTS
FOR (a:Auction) REQUIRE a.id IS UNIQUE;

CREATE (a:Auction {
    id: randomUUID(),
    auctionType: 'reverse',
    status: 'active',
    startTime: datetime('2026-06-10T14:00:00Z'),
    endTime: datetime('2026-06-10T16:00:00Z'),
    autoExtend: true,
    extendMinutes: 5,
    minDecrement: 100.00,
    currentBest: 38000.00,
    bidCount: 12
})

// ============================================================
// PURCHASE ORDER NODES
// ============================================================

CREATE CONSTRAINT po_id_unique IF NOT EXISTS
FOR (po:PurchaseOrder) REQUIRE po.id IS UNIQUE;

CREATE (po:PurchaseOrder {
    id: randomUUID(),
    poNumber: 'PO-2026-005678',
    status: 'approved',
    currency: 'USD',
    subtotal: 42500.00,
    taxAmount: 3400.00,
    totalAmount: 45900.00,
    paymentTerms: 'Net 30',
    expectedDelivery: date('2026-08-15'),
    createdAt: datetime(),
    approvedAt: datetime()
})

// PO Line items
CREATE CONSTRAINT pol_id_unique IF NOT EXISTS
FOR (pl:POLine) REQUIRE pl.id IS UNIQUE;

// ============================================================
// GOODS RECEIPT, INVOICE, CONTRACT NODES
// ============================================================

CREATE CONSTRAINT gr_id_unique IF NOT EXISTS
FOR (gr:GoodsReceipt) REQUIRE gr.id IS UNIQUE;

CREATE CONSTRAINT inv_id_unique IF NOT EXISTS
FOR (inv:Invoice) REQUIRE inv.id IS UNIQUE;

CREATE CONSTRAINT contract_id_unique IF NOT EXISTS
FOR (c:Contract) REQUIRE c.id IS UNIQUE;

// Budget nodes
CREATE CONSTRAINT budget_id_unique IF NOT EXISTS
FOR (b:Budget) REQUIRE b.id IS UNIQUE;

// Compliance document nodes
CREATE CONSTRAINT doc_id_unique IF NOT EXISTS
FOR (d:ComplianceDoc) REQUIRE d.id IS UNIQUE;
```

## Relationship Definitions

```cypher
// ============================================================
// ORGANIZATION RELATIONSHIPS
// ============================================================

// User belongs to organization
(u:User)-[:BELONGS_TO]->(o:Organization)

// Organization has addresses
(o:Organization)-[:HAS_ADDRESS {
    addressType: 'headquarters',  // 'billing', 'shipping', 'warehouse'
    line1: '123 Industrial Blvd',
    city: 'Chicago',
    stateProvince: 'IL',
    postalCode: '60601',
    countryCode: 'US',
    isPrimary: true
}]->(o)  // self-referential with properties, or use Address nodes

// Supplier supplies to buyer (aggregate relationship)
(supplier:Organization)-[:SUPPLIES_TO {
    since: date('2024-03-01'),
    totalOrders: 47,
    totalValue: 1250000.00,
    avgQualityScore: 4.3,
    avgDeliveryScore: 4.1,
    lastOrderDate: date('2026-05-10'),
    status: 'preferred'          // 'approved', 'preferred', 'restricted', 'blocked'
}]->(buyer:Organization)

// Supplier categorized under UNSPSC taxonomy
(supplier:Organization)-[:CATEGORIZED_AS {
    isPrimary: true
}]->(c:Category)

// Category hierarchy
(child:Category)-[:CHILD_OF]->(parent:Category)

// ============================================================
// SOURCING RELATIONSHIPS
// ============================================================

// Buyer creates sourcing event
(buyer:Organization)-[:CREATED_EVENT]->(se:SourcingEvent)
(u:User)-[:AUTHORED]->(se:SourcingEvent)

// Sourcing event contains line items
(se:SourcingEvent)-[:HAS_LINE]->(sl:SourcingLine)
(sl:SourcingLine)-[:IN_CATEGORY]->(c:Category)

// Supplier invited to sourcing event
(se:SourcingEvent)-[:INVITED {
    status: 'accepted',
    invitedAt: datetime(),
    respondedAt: datetime(),
    invitedBy: 'user-uuid'
}]->(supplier:Organization)

// Supplier submits bid
(supplier:Organization)-[:SUBMITTED_BID]->(b:Bid)
(b:Bid)-[:BID_ON]->(se:SourcingEvent)
(b:Bid)-[:HAS_LINE]->(bl:BidLine)
(bl:BidLine)-[:PRICES]->(sl:SourcingLine)

// Bid awarded
(se:SourcingEvent)-[:AWARDED_TO {
    awardedAt: datetime(),
    awardedBy: 'user-uuid'
}]->(b:Bid)

// Auction relationships
(se:SourcingEvent)-[:HAS_AUCTION]->(a:Auction)
(supplier:Organization)-[:PLACED_AUCTION_BID {
    amount: 38500.00,
    rank: 2,
    placedAt: datetime()
}]->(a:Auction)

// ============================================================
// PURCHASE ORDER RELATIONSHIPS
// ============================================================

// PO issued by buyer to supplier
(buyer:Organization)-[:ISSUED_PO]->(po:PurchaseOrder)
(po:PurchaseOrder)-[:PLACED_WITH]->(supplier:Organization)
(po:PurchaseOrder)-[:ORIGINATED_FROM]->(se:SourcingEvent)
(po:PurchaseOrder)-[:BASED_ON]->(b:Bid)

// PO contains line items
(po:PurchaseOrder)-[:HAS_LINE]->(pl:POLine)

// Approval chain
(u:User)-[:APPROVED {
    stepOrder: 1,
    approvedAt: datetime(),
    comments: 'Within budget'
}]->(po:PurchaseOrder)

// ============================================================
// THREE-WAY MATCHING RELATIONSHIPS
// ============================================================

// Goods receipt against PO
(gr:GoodsReceipt)-[:RECEIVED_FOR]->(po:PurchaseOrder)
(u:User)-[:RECORDED_RECEIPT]->(gr:GoodsReceipt)
(gr:GoodsReceipt)-[:RECEIVED_LINE {
    quantityReceived: 48000,
    quantityRejected: 200,
    inspectionNotes: 'Minor surface defects on 200 units'
}]->(pl:POLine)

// Invoice against PO
(inv:Invoice)-[:INVOICES]->(po:PurchaseOrder)
(supplier:Organization)-[:SENT_INVOICE]->(inv:Invoice)
(inv:Invoice)-[:INVOICE_LINE {
    description: 'M10 x 30mm Ti Hex Bolt',
    quantity: 48000,
    unitPrice: 0.83,
    lineTotal: 39840.00
}]->(pl:POLine)

// Three-way match result
(inv:Invoice)-[:MATCHED_WITH {
    matchType: 'three_way',
    matchResult: 'clean',
    matchedAt: datetime(),
    priceVariance: 0.02,
    quantityVariance: 0.0
}]->(gr:GoodsReceipt)

// ============================================================
// CONTRACT RELATIONSHIPS
// ============================================================

(c:Contract)-[:BETWEEN_BUYER]->(buyer:Organization)
(c:Contract)-[:BETWEEN_SUPPLIER]->(supplier:Organization)
(po:PurchaseOrder)-[:UNDER_CONTRACT]->(c:Contract)

// ============================================================
// COMPLIANCE AND ESG
// ============================================================

(supplier:Organization)-[:HAS_COMPLIANCE_DOC]->(d:ComplianceDoc)
(u:User)-[:VERIFIED_DOC {verifiedAt: datetime()}]->(d:ComplianceDoc)

// ============================================================
// RATINGS AND PERFORMANCE
// ============================================================

(buyer:Organization)-[:RATED {
    qualityScore: 4.5,
    deliveryScore: 4.0,
    communicationScore: 4.2,
    overallScore: 4.3,
    comments: 'Consistent quality, slight delays on rush orders',
    ratedAt: datetime(),
    poId: 'po-uuid'
}]->(supplier:Organization)

// ============================================================
// BUDGET AND SPEND
// ============================================================

(buyer:Organization)-[:HAS_BUDGET]->(b:Budget)
(po:PurchaseOrder)-[:CHARGED_TO {amount: 45900.00}]->(b:Budget)
```

## Powerful Graph Queries

```cypher
// 1. SUPPLIER DISCOVERY: Find suppliers for a category with quality > 4.0
//    who have delivered to companies in my industry
MATCH (me:Organization {id: $myOrgId})
MATCH (supplier:Organization)-[:CATEGORIZED_AS]->(cat:Category {unspscCode: $targetCode})
MATCH (supplier)-[:SUPPLIES_TO]->(otherBuyer:Organization)
WHERE otherBuyer.industry = me.industry
  AND supplier.status = 'active'
WITH supplier, AVG(otherBuyer.avgQualityScore) AS avgQuality
WHERE avgQuality > 4.0
RETURN supplier.name, supplier.country, avgQuality
ORDER BY avgQuality DESC
LIMIT 20

// 2. SUPPLY CHAIN RISK: Find categories where spend is concentrated
//    in fewer than 3 suppliers
MATCH (buyer:Organization {id: $buyerOrgId})-[:ISSUED_PO]->(po:PurchaseOrder)-[:PLACED_WITH]->(supplier)
MATCH (po)-[:HAS_LINE]->(pl:POLine)-[:IN_CATEGORY]->(cat:Category)
WITH cat, COUNT(DISTINCT supplier) AS supplierCount, SUM(pl.lineTotal) AS totalSpend
WHERE supplierCount < 3 AND totalSpend > 100000
RETURN cat.name, supplierCount, totalSpend
ORDER BY totalSpend DESC

// 3. THREE-WAY MATCH VERIFICATION: Traverse PO -> receipt -> invoice
MATCH (po:PurchaseOrder {poNumber: $poNumber})
MATCH (po)-[:HAS_LINE]->(pl:POLine)
OPTIONAL MATCH (gr:GoodsReceipt)-[rl:RECEIVED_LINE]->(pl)
OPTIONAL MATCH (inv:Invoice)-[il:INVOICE_LINE]->(pl)
RETURN pl.itemDescription,
       pl.quantity AS ordered,
       rl.quantityReceived AS received,
       il.quantity AS invoiced,
       pl.unitPrice AS poPrice,
       il.unitPrice AS invoicePrice,
       CASE WHEN ABS(pl.unitPrice - il.unitPrice) > 0.01 THEN 'PRICE_MISMATCH' ELSE 'OK' END AS priceCheck

// 4. MAVERICK SPEND DETECTION: Find POs not linked to any sourcing event or contract
MATCH (buyer:Organization {id: $buyerOrgId})-[:ISSUED_PO]->(po:PurchaseOrder)
WHERE NOT (po)-[:ORIGINATED_FROM]->(:SourcingEvent)
  AND NOT (po)-[:UNDER_CONTRACT]->(:Contract)
  AND po.totalAmount > 5000
RETURN po.poNumber, po.totalAmount, po.createdAt
ORDER BY po.totalAmount DESC

// 5. SUPPLIER NETWORK ANALYSIS: Find shared suppliers between business units
MATCH (bu1:Organization {name: $bu1Name})-[:ISSUED_PO]->(:PurchaseOrder)-[:PLACED_WITH]->(supplier)
MATCH (bu2:Organization {name: $bu2Name})-[:ISSUED_PO]->(:PurchaseOrder)-[:PLACED_WITH]->(supplier)
WHERE bu1 <> bu2
RETURN supplier.name, supplier.id
ORDER BY supplier.name

// 6. RECOMMENDATION: Suppliers similar to my best-rated ones
MATCH (me:Organization {id: $myOrgId})-[r:RATED]->(topSupplier)
WHERE r.overallScore >= 4.5
MATCH (topSupplier)-[:CATEGORIZED_AS]->(cat:Category)
MATCH (similar:Organization)-[:CATEGORIZED_AS]->(cat)
WHERE NOT (me)-[:SUPPLIES_TO|RATED]-(similar)
  AND similar.status = 'active'
RETURN DISTINCT similar.name, similar.esgScore, COLLECT(DISTINCT cat.name) AS categories
LIMIT 10
```

---

## Indexes for Performance

```cypher
// Full-text search index on organization names and descriptions
CREATE FULLTEXT INDEX orgSearch IF NOT EXISTS
FOR (o:Organization) ON EACH [o.name, o.city, o.country];

// Full-text search on sourcing events
CREATE FULLTEXT INDEX sourcingSearch IF NOT EXISTS
FOR (se:SourcingEvent) ON EACH [se.title, se.description];

// Composite index for PO lookups
CREATE INDEX poStatusIdx IF NOT EXISTS
FOR (po:PurchaseOrder) ON (po.status, po.createdAt);

// Index for contract expiry monitoring
CREATE INDEX contractEndIdx IF NOT EXISTS
FOR (c:Contract) ON (c.endDate, c.status);

// Index for spend date range queries
CREATE INDEX budgetYearIdx IF NOT EXISTS
FOR (b:Budget) ON (b.fiscalYear, b.category);
```

---

## Trade-offs

**Strengths:**
- Relationship-first modeling makes supplier discovery, network analysis, and recommendation queries natural and performant. Multi-hop traversals that require 5+ JOINs in SQL are single Cypher queries.
- The SUPPLIES_TO aggregate relationship captures the evolving buyer-supplier relationship with properties (total orders, average ratings, status) that would require complex views in a relational model.
- Three-way matching is a graph traversal: follow PO -> lines -> receipts and invoices, compare properties at each node.
- Category hierarchy (UNSPSC taxonomy) is a natural tree structure in a graph, supporting queries like "find all suppliers in any subcategory of Industrial Components."
- Supply chain risk analysis (single-source dependency, spend concentration) maps directly to graph analytics (degree centrality, community detection).
- AI features like supplier matching benefit from graph embeddings (Node2Vec, GraphSAGE) that capture structural similarity in the procurement network.

**Weaknesses:**
- Graph databases are not optimized for heavy aggregation queries (SUM, AVG, GROUP BY) that power spend analytics dashboards. These queries are significantly slower than in SQL or columnar stores.
- ACID transaction support in Neo4j is single-database scoped. Complex distributed transactions (e.g., atomic three-way match across PO, receipt, and invoice) require careful design.
- The operational tooling ecosystem (backup, monitoring, migrations) is less mature than PostgreSQL's.
- Fewer engineers have production Neo4j experience compared to relational databases, increasing hiring and training costs.
- Financial data (monetary amounts, tax calculations) lacks the precise NUMERIC types and constraints available in PostgreSQL. Application-layer validation must compensate.
- Licensing: Neo4j Enterprise (clustering, RBAC, online backup) requires a commercial license, while the Community edition has limitations.

## Scalability Considerations

- Neo4j supports sharding via Fabric (Enterprise edition) for distributing data across multiple databases.
- Read replicas handle query scaling for supplier discovery and analytics workloads.
- For high-volume auction bidding, consider a dedicated fast-write store (Redis, Kafka) that feeds results into the graph asynchronously.
- Aggregate spend analytics should be offloaded to a columnar store (ClickHouse, DuckDB) or materialized in a relational database, with the graph used for network analysis and discovery.

## Migration Path

A graph database works best as a complement to a relational system rather than a full replacement. The recommended architecture is to use PostgreSQL (Suggestion 1 or 3) as the transactional system of record for financial data, approval workflows, and document storage, while maintaining a synchronized Neo4j instance for supplier discovery, network analysis, recommendation, and risk assessment. Change Data Capture (CDC) from PostgreSQL to Neo4j keeps the graph current. This polyglot approach plays to each engine's strengths: relational for transactions and aggregations, graph for traversals and recommendations.

An alternative single-engine approach uses PostgreSQL with the Apache AGE extension, which adds Cypher query support to PostgreSQL. This provides graph query capabilities without a separate database, though with less mature tooling and performance than native Neo4j for complex traversals.
