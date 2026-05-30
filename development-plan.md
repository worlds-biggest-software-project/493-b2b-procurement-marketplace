# B2B Procurement Marketplace — Phased Development Plan

> Project: 493-b2b-procurement-marketplace · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased implementation roadmap. It covers technology choices, project structure, and a sequence of additive phases. Each phase produces a working, testable increment.

The data model adopts **Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the system of record. Rationale: procurement is transactional (financial amounts, statuses, three-way matching) and demands referential integrity, but it also ingests highly variable external documents (cXML, EDI, UBL) and tenant-specific custom fields. A hybrid model keeps money and foreign keys in constrained columns while parking variable structure in GIN-indexed JSONB — avoiding both rigid schema churn and the operational cost of polyglot persistence at MVP. Suggestion 4's graph capabilities (supplier discovery, network analysis) are deferred to a later phase as a synchronised read store, exactly as that document recommends. Suggestion 2's event-sourcing concepts are honoured through an immutable `audit_log` and domain-event emission, without the full CQRS complexity.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The product is AI-native: RFQ generation, bid scoring, contract risk extraction, and embedding-based supplier matching are core. Python has first-class LLM SDKs (anthropic, openai), embedding libraries, and mature procurement-document tooling. The reference open-source competitor (ERPNext) is also Python, validating the ecosystem fit. |
| API framework | FastAPI | Generates OpenAPI 3.1 automatically (required by `standards.md`), native Pydantic v2 validation enforces JSON Schema (Draft 2020-12) on request/response bodies, async support for LLM and webhook I/O, and built-in OAuth2/OIDC security scheme declaration (RFC 6749/6750). |
| Data validation | Pydantic v2 | Enforces structure on JSONB payloads at the application layer — the key weakness the hybrid model carries. Emits JSON Schema for OpenAPI. |
| ORM / DB access | SQLAlchemy 2.0 (async) + Alembic | Mature support for PostgreSQL JSONB, GIN indexes, partitioning, and row-level security. Alembic handles versioned migrations. |
| Primary database | PostgreSQL 16 | Required by the chosen data model: NUMERIC for money, JSONB + GIN for flexible fields, table partitioning for `spend_records`/`audit_log`, row-level security for multi-tenant isolation. |
| Cache / queue broker | Redis 7 | Backs the task queue, rate limiting, auction live-state, and session/idempotency keys. |
| Task queue | Celery (Redis broker) | Async workloads: LLM calls (RFQ generation, bid scoring), email distribution, three-way match runs, contract expiry sweeps, EDI/cXML document processing. Long-running and retryable. |
| LLM provider | Anthropic Claude (via `anthropic` SDK), provider-abstracted | Core AI features. Abstracted behind an `LLMClient` interface so a self-hosted model can be swapped in for on-prem deployments. Prompt caching enabled for repeated system prompts. |
| Embeddings + vector search | `pgvector` extension on PostgreSQL | Supplier matching by capability/compliance/performance. Keeping vectors in PostgreSQL avoids a separate vector DB at MVP scale. |
| Auth | Authlib + python-jose | OAuth 2.0 authorization-code + client-credentials flows (RFC 6749), Bearer tokens (RFC 6750), PKCE (RFC 7636) for SPA/mobile, OIDC for enterprise SSO (Okta/Entra ID). |
| Object storage | S3-compatible (boto3; MinIO for self-host) | Compliance documents, contract files, OCR'd invoices, attachments. |
| Frontend | Next.js 14 (App Router) + TypeScript + shadcn/ui + Tailwind | Buyer/supplier portals demand consumer-grade UX (the headline market gap). Server components for dashboards, client components for live auction bidding. Separate deployable from the API. |
| Real-time | WebSockets (FastAPI) + Redis pub/sub | Live auction bidding and notification push. |
| Standards libraries | `lxml` (cXML 1.2.069, UBL 2.1 XML), `pyx12` / `bots`-style EDI mapping, custom PEPPOL BIS 3.0 serialiser | Document interchange per `standards.md`. |
| MCP server | `mcp` Python SDK | Expose `create_rfq`, `search_suppliers`, `get_spend_summary`, `approve_po` as MCP tools for agentic procurement (per `standards.md` notes on agentic commerce). |
| Containerisation | Docker + docker-compose | Self-hosted, cloud, and hybrid deployment per README. compose orchestrates api, worker, postgres, redis, minio, frontend. |
| Testing | pytest + pytest-asyncio + httpx + testcontainers + factory_boy | Unit, mocked-integration, real-integration (ephemeral Postgres/Redis via testcontainers), and E2E. |
| Frontend testing | Vitest + Playwright | Component tests and E2E browser flows. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Consistent style, static type safety on a large codebase. |
| Package manager | uv (Python), pnpm (frontend) | Fast, reproducible installs. |

### Project Structure

```
b2b-procurement-marketplace/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi/                          # exported OpenAPI 3.1 spec (CI-generated)
├── migrations/                       # Alembic versioned migrations
│   └── versions/
├── src/
│   └── procmkt/
│       ├── __init__.py
│       ├── main.py                   # FastAPI app factory, router registration
│       ├── config.py                 # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py            # async engine, session factory
│       │   ├── base.py               # declarative base, mixins (timestamps, JSONB)
│       │   └── rls.py                # row-level security tenant context
│       ├── models/                   # SQLAlchemy ORM models
│       │   ├── organization.py
│       │   ├── user.py
│       │   ├── supplier.py
│       │   ├── sourcing.py
│       │   ├── bid.py
│       │   ├── auction.py
│       │   ├── purchase_order.py
│       │   ├── invoice.py
│       │   ├── contract.py
│       │   ├── budget.py
│       │   ├── spend.py
│       │   ├── approval.py
│       │   ├── integration.py
│       │   └── audit.py
│       ├── schemas/                  # Pydantic request/response + JSONB sub-schemas
│       ├── api/
│       │   ├── deps.py               # auth, tenant, pagination dependencies
│       │   ├── v1/
│       │   │   ├── router.py
│       │   │   ├── auth.py
│       │   │   ├── organizations.py
│       │   │   ├── users.py
│       │   │   ├── suppliers.py
│       │   │   ├── sourcing.py
│       │   │   ├── bids.py
│       │   │   ├── auctions.py
│       │   │   ├── purchase_orders.py
│       │   │   ├── receipts.py
│       │   │   ├── invoices.py
│       │   │   ├── contracts.py
│       │   │   ├── budgets.py
│       │   │   ├── analytics.py
│       │   │   └── integrations.py
│       │   └── ws/                   # WebSocket endpoints (auctions, notifications)
│       ├── services/                 # business logic (no HTTP awareness)
│       │   ├── auth_service.py
│       │   ├── org_service.py
│       │   ├── sourcing_service.py
│       │   ├── bid_service.py
│       │   ├── award_service.py
│       │   ├── po_service.py
│       │   ├── matching_service.py   # three-way matching engine
│       │   ├── approval_service.py
│       │   ├── contract_service.py
│       │   ├── budget_service.py
│       │   ├── analytics_service.py
│       │   └── notification_service.py
│       ├── ai/
│       │   ├── client.py             # LLMClient interface + Anthropic impl
│       │   ├── prompts/              # versioned prompt templates
│       │   ├── rfq_generator.py
│       │   ├── bid_scorer.py
│       │   ├── contract_risk.py
│       │   ├── supplier_matcher.py   # pgvector embedding search
│       │   └── anomaly.py            # maverick spend detection
│       ├── integrations/
│       │   ├── cxml/                 # PunchOut 2.0, OrderRequest, InvoiceDetail
│       │   ├── ubl/                  # UBL 2.1 documents
│       │   ├── peppol/               # PEPPOL BIS Billing 3.0
│       │   ├── edi/                  # X12 850/810, EDIFACT ORDERS/INVOIC
│       │   └── erp/                  # SAP/Oracle/NetSuite/Workday connectors
│       ├── mcp/
│       │   └── server.py             # MCP tool definitions
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/
│       ├── events/
│       │   ├── domain_events.py      # event dataclasses
│       │   └── dispatcher.py         # emit → audit_log + handlers
│       └── core/
│           ├── security.py           # hashing, JWT, PKCE
│           ├── errors.py             # error envelope, exception handlers
│           ├── pagination.py         # RFC 8288 Link headers
│           └── reference.py          # reference-number generation
├── frontend/
│   ├── package.json
│   ├── app/                          # Next.js App Router
│   ├── components/
│   ├── lib/                          # API client, auth
│   └── tests/
└── tests/
    ├── conftest.py                   # fixtures: db, redis, client, factories
    ├── factories/
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure groups by concern (models, schemas, api, services, ai, integrations) so every phase adds files without restructuring.

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the runnable skeleton: configuration, database connectivity, migration tooling, the error/response envelope, containerisation, and CI. After this phase the API boots, exposes a health check, connects to PostgreSQL and Redis, and has a passing test harness. Everything later builds on these primitives.

### Tasks

#### 1.1 — Project bootstrap and tooling

**What**: Initialise the Python project, dependency management, linting, typing, and pre-commit hooks.

**Design**:
- `pyproject.toml` with `uv` managing deps: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pydantic`, `pydantic-settings`, `redis`, `celery`, `authlib`, `python-jose[cryptography]`, `passlib[bcrypt]`, `anthropic`, `pgvector`, `lxml`, `boto3`, `httpx`. Dev group: `pytest`, `pytest-asyncio`, `httpx`, `testcontainers`, `factory_boy`, `ruff`, `mypy`.
- Ruff config: line length 100, target py312, enable `E,F,I,UP,B,SIM`. mypy strict.
- `.pre-commit-config.yaml` running ruff + mypy.

**Testing**:
- `Unit: import procmkt.main → app object exists`
- CI: `ruff check`, `ruff format --check`, `mypy src` all exit 0.

#### 1.2 — Configuration and app factory

**What**: Environment-driven settings and a FastAPI app factory.

**Design**:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="PROCMKT_")
    environment: Literal["dev", "test", "staging", "prod"] = "dev"
    database_url: str                       # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: SecretStr
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 3600
    refresh_token_ttl_seconds: int = 2_592_000
    s3_endpoint_url: str | None = None
    s3_bucket: str = "procmkt"
    anthropic_api_key: SecretStr | None = None
    llm_model: str = "claude-opus-4-8"
    cors_origins: list[str] = []
```
- `create_app() -> FastAPI` registers exception handlers, CORS, the `/healthz` route, and the v1 router. `GET /healthz` returns `{"status": "ok", "db": <bool>, "redis": <bool>}` by pinging both.

**Testing**:
- `Unit: Settings loads from env with PROCMKT_ prefix; missing jwt_secret → ValidationError`
- `Integration (real, testcontainers): GET /healthz → 200, db=true, redis=true`
- `Integration (mocked): redis down → /healthz returns 200 with redis=false` (health is informational, not gating).

#### 1.3 — Database layer, base model, migrations

**What**: Async SQLAlchemy engine/session, declarative base with shared mixins, Alembic wired for autogenerate, and `pgvector`/`uuid-ossp` extension bootstrap migration.

**Design**:
```python
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class Base(DeclarativeBase, TimestampMixin):
    id: Mapped[UUID] = mapped_column(primary_key=True, server_default=text("gen_random_uuid()"))
```
- `get_session()` async dependency yields a session and handles commit/rollback.
- First migration enables `CREATE EXTENSION IF NOT EXISTS vector;` and `pgcrypto`.

**Testing**:
- `Integration (real): apply all migrations to fresh DB → exit 0; extensions present`
- `Integration (real): downgrade base → no procmkt tables remain`
- `Unit: TimestampMixin sets created_at/updated_at on flush`

#### 1.4 — Error envelope, pagination, reference numbers

**What**: Consistent error responses, RFC 8288 pagination, and human-readable document reference generation.

**Design**:
- Error envelope: `{"error": {"code": "string", "message": "string", "details": [...]}}`. Exception handlers map `ValidationError → 422`, `NotFoundError → 404`, `PermissionDenied → 403`, `Conflict → 409`.
- `paginate(query, limit, cursor)` returns items plus a `Link` header (`rel="next"`, `rel="prev"`) per RFC 8288. Default limit 50, max 200.
- `next_reference(doc_type, org_id)` → e.g. `RFQ-2026-000123`, `PO-2026-000456`; monotonic per org+type+year via a `reference_sequences` table with a row lock.

**Testing**:
- `Unit: NotFoundError → 404 with code "not_found"`
- `Unit: paginate returns Link header with correct next cursor`
- `Integration (real): concurrent next_reference calls → no duplicate numbers (row lock holds)`

#### 1.5 — Containerisation and compose

**What**: Dockerfile and docker-compose stack.

**Design**:
- Multi-stage Dockerfile (builder installs deps, runtime slim).
- `docker-compose.yml` services: `api`, `worker`, `postgres:16`, `redis:7`, `minio`, `frontend`. Healthchecks on postgres/redis; api/worker `depends_on` healthy.

**Testing**:
- `E2E (CI): docker compose up → api /healthz returns 200 within 60s`
- `Build: docker build succeeds`

---

## Phase 2: Identity, Organizations & Access Control

### Purpose
Implement multi-tenant identity: organizations (buyer/supplier/both), users, roles, OAuth 2.0 / OIDC authentication, and row-level tenant isolation. This is the security spine; every later endpoint depends on an authenticated principal and an enforced tenant boundary (OWASP API Security A01 — broken object-level authorization).

### Tasks

#### 2.1 — Organization and address models + CRUD

**What**: `organizations` table (per Suggestion 3) with JSONB `profile`/`addresses`/`settings`, plus CRUD endpoints.

**Design**:
```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    org_type VARCHAR(20) NOT NULL CHECK (org_type IN ('buyer','supplier','both')),
    tax_id VARCHAR(50), duns_number VARCHAR(13),
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(20) NOT NULL DEFAULT 'active'
        CHECK (status IN ('pending','active','suspended','deactivated')),
    profile JSONB NOT NULL DEFAULT '{}',
    addresses JSONB NOT NULL DEFAULT '[]',
    settings JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_org_type ON organizations(org_type);
CREATE INDEX idx_org_profile ON organizations USING GIN(profile jsonb_path_ops);
```
- Pydantic `AddressModel` (type, line_1, line_2, city, state_province, postal_code, country_code ISO-3166-alpha-2, is_primary) validates each element of the `addresses` JSONB array.
- Endpoints: `POST /v1/organizations`, `GET /v1/organizations/{id}`, `PATCH /v1/organizations/{id}`.

**Testing**:
- `Unit: AddressModel rejects 3-letter country_code → ValidationError`
- `Integration (real): POST org with org_type='vendor' → 422`
- `Integration (real): create then GET org → addresses round-trip intact`

#### 2.2 — Users, roles, password security

**What**: `users` (JSONB `profile`, `roles` text[]) and a fixed role catalogue.

**Design**:
- Roles seeded: `org_admin`, `procurement_manager`, `buyer`, `approver`, `finance`, `supplier_admin`, `supplier_user`, `viewer`.
- Passwords hashed with bcrypt (passlib). `users.roles` is a `VARCHAR(50)[]` with a GIN index (per Suggestion 3).
- Endpoints: `POST /v1/users` (invite), `GET /v1/users/me`, `PATCH /v1/users/{id}`, `POST /v1/users/{id}/roles`.

**Testing**:
- `Unit: password hash verifies; wrong password fails`
- `Integration (real): assign role 'approver' → roles array contains it`
- `Integration (real): create user under org A cannot be fetched by org B principal → 404` (tenant isolation, see 2.4)

#### 2.3 — OAuth 2.0 / OIDC authentication

**What**: Token issuance and validation per RFC 6749/6750/7636.

**Design**:
- Flows: `password` (first-party portal login → access+refresh JWT), `client_credentials` (ERP/server integrations), `authorization_code + PKCE` (SPA/mobile). OIDC discovery endpoint for enterprise IdP federation.
- `POST /v1/auth/token`, `POST /v1/auth/refresh`, `POST /v1/auth/revoke`. Access tokens are JWTs carrying `sub` (user_id), `org_id`, `roles`, `scope`.
- `get_current_principal()` dependency decodes the Bearer token, loads the user, and constructs a `Principal(user_id, org_id, org_type, roles, scopes)`.
- API clients (`api_clients` table: client_id, hashed secret, org_id, allowed scopes) back client_credentials.

**Testing**:
- `Unit: expired JWT → 401`
- `Integration (real): password grant returns access+refresh; refresh rotates`
- `Integration (real): client_credentials with disallowed scope → 403`
- `Integration (mocked OIDC): authorization_code+PKCE happy path issues token`
- `Security: token signed with wrong key → 401`

#### 2.4 — Row-level security & tenant context

**What**: Enforce tenant isolation at the DB and dependency layer.

**Design**:
- PostgreSQL RLS policies on every tenant-scoped table keyed on a session GUC `app.current_org`, set per-request from the Principal.
- `require_roles(*roles)` and `require_scope(scope)` dependencies for endpoint authorization.
- Buyer/supplier cross-visibility rules: a supplier sees only sourcing events it was invited to and its own bids/POs/invoices; a buyer sees its own documents.

**Testing**:
- `Integration (real): org A principal querying org B's PO → 0 rows (RLS), endpoint returns 404`
- `Integration (real): supplier not invited to RFQ → GET returns 404`
- `Unit: require_roles('approver') with buyer-only principal → 403`

---

## Phase 3: Sourcing — RFQ/RFP/RFI Lifecycle

### Purpose
Deliver the heart of the product: buyers create structured sourcing events with line items, invite suppliers, and publish. This is the first column of MVP value and the entry point for every downstream workflow (bids, awards, POs).

### Tasks

#### 3.1 — Sourcing event & line models

**What**: `sourcing_events` and `sourcing_event_lines` per Suggestion 3, with the full status state machine.

**Design**:
```sql
CREATE TABLE sourcing_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_org_id UUID NOT NULL REFERENCES organizations(id),
    created_by UUID NOT NULL REFERENCES users(id),
    event_type VARCHAR(10) NOT NULL CHECK (event_type IN ('RFQ','RFP','RFI')),
    title VARCHAR(300) NOT NULL,
    reference_number VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(30) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft','published','closed','evaluating','awarded','cancelled')),
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    published_at TIMESTAMPTZ, response_deadline TIMESTAMPTZ,
    details JSONB NOT NULL DEFAULT '{}',
    evaluation_config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- `sourcing_event_lines` with JSONB `specifications` (unspsc_code, technical_specs, tolerances). `details` JSONB validated by `SourcingDetails` Pydantic model: `description`, `terms_and_conditions`, `evaluation_criteria[]`, `required_certifications[]`, `attachments[]`.
- State machine (enforced in `sourcing_service`):
  `draft → published → closed → evaluating → awarded`; `draft|published → cancelled`. Illegal transitions raise `Conflict`.

**Testing**:
- `Unit: transition published→awarded directly → Conflict (must pass through closed/evaluating)`
- `Unit: SourcingDetails rejects unknown evaluation_criteria value`
- `Integration (real): create draft RFQ → reference_number matches RFQ-<year>-NNNNNN`

#### 3.2 — Sourcing CRUD & line management

**What**: Endpoints to author events and lines while in `draft`.

**Design**:
- `POST /v1/sourcing-events`, `GET`, `PATCH`, `POST /v1/sourcing-events/{id}/lines`, `PATCH/DELETE` lines. Mutations to lines allowed only in `draft`.
- `require_roles('procurement_manager','buyer')`.

**Testing**:
- `Integration (real): add line to published event → 409`
- `Integration (real): list lines ordered by line_number`

#### 3.3 — Supplier invitations & publish

**What**: Invite suppliers and publish, opening the response window.

**Design**:
```sql
CREATE TABLE sourcing_invitations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_events(id),
    supplier_org_id UUID NOT NULL REFERENCES organizations(id),
    invited_by UUID NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','viewed','accepted','declined')),
    invited_at TIMESTAMPTZ NOT NULL DEFAULT now(), responded_at TIMESTAMPTZ,
    UNIQUE (sourcing_event_id, supplier_org_id)
);
```
- `POST /v1/sourcing-events/{id}/invitations` (body: supplier_org_ids[]). `POST /{id}/publish` requires ≥1 line and ≥1 invitation, sets `status='published'`, `published_at`, and enqueues invitation emails (Celery, Phase 9 notification service; stubbed earlier).
- Invitee endpoints: `POST /v1/invitations/{id}/accept|decline`.

**Testing**:
- `Integration (real): publish with no lines → 422`
- `Integration (real): duplicate invitation for same supplier → 409`
- `Integration (real): invited supplier accepts → status='accepted', responded_at set`
- `Integration (real): non-invited supplier cannot view event (RLS) → 404`

---

## Phase 4: Bidding, Comparison & Award

### Purpose
Suppliers respond to published events with structured bids; buyers compare bids side-by-side and award. Completing this phase delivers the full manual sourcing-to-award MVP workflow described in features.md "Must-have".

### Tasks

#### 4.1 — Bid & bid-line models with submission rules

**What**: `bids` and `bid_lines` per Suggestion 3 (`bid_details`, `evaluation` JSONB).

**Design**:
- Bid status: `draft → submitted → (withdrawn|awarded|rejected)`. A bid may only be created/edited by a supplier invited to the event, only while the event is `published`, and before `response_deadline`.
- `bid_lines` reference `sourcing_event_lines` and carry `unit_price`, `quantity`, `lead_time_days`, JSONB `line_details` (volume_discounts, alternatives).
- `total_amount` computed server-side from bid lines on submit; never trusted from client (OWASP mass-assignment guard).

**Testing**:
- `Unit: bid total recomputed = sum(unit_price*quantity); client-supplied total ignored`
- `Integration (real): submit bid after deadline → 409`
- `Integration (real): bid by non-invited supplier → 403`
- `Integration (real): submit bid missing a required event line → 422`

#### 4.2 — Bid endpoints

**What**: Supplier bid CRUD + submit.

**Design**:
- `POST /v1/sourcing-events/{id}/bids`, `PATCH /v1/bids/{id}`, `POST /v1/bids/{id}/submit`, `POST /v1/bids/{id}/withdraw`. Suppliers see only their own bids; buyers see all bids on their events only after the event moves to `closed`/`evaluating` (sealed-bid integrity).

**Testing**:
- `Integration (real): buyer cannot read competitor bids while event published → 404`
- `Integration (real): supplier A cannot read supplier B's bid → 404`

#### 4.3 — Close, comparison matrix & award

**What**: Close events, produce a side-by-side comparison, and award a winning bid generating a PO stub.

**Design**:
- `POST /{id}/close` sets `closed`; auto-close via Celery beat when `response_deadline` passes.
- `GET /v1/sourcing-events/{id}/comparison` returns a matrix: per event-line, every supplier's `unit_price`, `quantity`, `lead_time_days`, line total, plus per-bid totals and lowest-price highlighting. Response is the `proj_bid_comparison` shape from Suggestion 2 (used here as a computed view, not a persisted projection).
- `POST /{id}/award` (body: winning_bid_id, optional split awards by line) → sets event `awarded`, bid `awarded`, others `rejected`, and emits `EventAwarded` domain event that Phase 5 consumes to create a PO.

**Testing**:
- `Unit: comparison matrix lowest unit_price flagged per line`
- `Integration (real): award sets winning bid awarded, others rejected`
- `Integration (real): award before close → 409`
- `Integration (real): EventAwarded emitted with winning_bid_id`

---

## Phase 5: Purchase Orders & Approval Workflows

### Purpose
Convert awards into purchase orders, route them through configurable approval chains, and track status to delivery. PO creation from an awarded event completes the MVP's transactional core and is prerequisite for matching, invoicing, and spend.

### Tasks

#### 5.1 — Purchase order & line models

**What**: `purchase_orders` and `purchase_order_lines` per Suggestion 3 (`terms`, `approval_state` JSONB).

**Design**:
- Status state machine: `draft → pending_approval → approved → sent → acknowledged → partially_received → received → closed`; `cancelled` reachable from any pre-`received` state.
- `terms` JSONB: payment_terms, incoterms, ship_to_address, bill_to_address, expected_delivery, cxml/edi references.
- PO generated from `EventAwarded`: lines copied from the winning bid lines; `sourcing_event_id` and `bid_id` set.

**Testing**:
- `Unit: PO line_total = quantity*unit_price; subtotal/total recomputed`
- `Integration (real): EventAwarded → PO created in draft with copied lines`
- `Unit: transition received→pending_approval → Conflict`

#### 5.2 — Approval chains & requests

**What**: `approval_chains` (JSONB `chain_config` steps) and `approval_requests` per Suggestion 3, with an amount-threshold routing engine.

**Design**:
```python
class ApprovalStep(BaseModel):
    step_order: int
    approver_role: str | None = None
    approver_user_id: UUID | None = None
    min_amount: Decimal | None = None
    max_amount: Decimal | None = None
    auto_approve_below: Decimal | None = None
```
- `approval_service.route(document)`: selects the active chain for `(org, document_type)`, filters steps whose `[min,max]` bracket contains the PO total, creates `approval_requests` for the first applicable step. Approving advances `current_step`; final approval → document `approved`.
- Endpoints: `POST /v1/purchase-orders/{id}/submit`, `POST /v1/approval-requests/{id}/decide` (approve/reject + comments).

**Testing**:
- `Unit: PO of 4,000 with auto_approve_below=5,000 → auto-approved, no request created`
- `Unit: PO of 50,000 routes through both managerial and finance steps in order`
- `Integration (real): approver from wrong role decides → 403`
- `Integration (real): reject at step 2 → PO status pending_approval→rejected, history recorded`

#### 5.3 — PO transmission & supplier acknowledgement

**What**: Send approved POs to suppliers and capture acknowledgement.

**Design**:
- `POST /{id}/send` (status→`sent`, enqueues notification + optional cXML/EDI transmission via Phase 8). `POST /{id}/acknowledge` (supplier; →`acknowledged`).

**Testing**:
- `Integration (real): send unapproved PO → 409`
- `Integration (real): supplier acknowledges → acknowledged, ack timestamp in approval_state`

---

## Phase 6: Receiving, Invoicing & Three-Way Matching

### Purpose
Record goods receipts, capture supplier invoices, and run automated three-way matching (PO ↔ receipt ↔ invoice). This closes the procure-to-pay loop and prevents duplicate/incorrect payments — a core finance-team value driver.

### Tasks

#### 6.1 — Goods receipts

**What**: `goods_receipts` and `goods_receipt_lines` per Suggestion 3.

**Design**:
- `POST /v1/purchase-orders/{id}/receipts` records `quantity_received`/`quantity_rejected` per PO line. Updates PO status to `partially_received` or `received` based on cumulative received vs ordered quantities.

**Testing**:
- `Unit: cumulative received < ordered → PO partially_received; == ordered → received`
- `Integration (real): receipt against cancelled PO → 409`
- `Unit: quantity_received > outstanding ordered → 422 (over-receipt guard, configurable tolerance)`

#### 6.2 — Invoices

**What**: `invoices` and `invoice_lines` per Suggestion 3 (`invoice_metadata` JSONB for OCR/EDI/UBL provenance).

**Design**:
- `POST /v1/invoices` (supplier-submitted or ingested). `match_status` enum: `unmatched|two_way|three_way|exception`. Unique `(supplier_org_id, invoice_number)` prevents duplicate submission.

**Testing**:
- `Integration (real): duplicate invoice_number for supplier → 409`
- `Integration (real): invoice references PO from different buyer → 422`

#### 6.3 — Three-way matching engine

**What**: A matching service producing clean matches or itemised discrepancies.

**Design**:
```python
@dataclass
class Discrepancy:
    line_id: UUID
    kind: Literal["price","quantity","missing_receipt","over_invoice","tax"]
    po_value: Decimal; invoice_value: Decimal; receipt_value: Decimal | None
    within_tolerance: bool

@dataclass
class MatchResult:
    match_type: Literal["two_way","three_way"]
    is_clean: bool
    discrepancies: list[Discrepancy]
```
- Algorithm: for each invoice line, locate PO line and aggregate received qty. Compare unit price (tolerance default ±2% or configurable per org in `settings`), invoiced qty ≤ received qty (3-way) or ≤ ordered qty (2-way fallback when no receipt required), tax. Clean → emit `InvoiceThreeWayMatched`, set invoice `matched`; otherwise `InvoiceMatchException`, status `disputed`, discrepancies stored in `invoice_metadata`.
- `POST /v1/invoices/{id}/match` runs synchronously; bulk nightly run via Celery.

**Testing**:
- `Unit: price within 2% tolerance → no discrepancy; 5% over → price discrepancy`
- `Unit: invoiced qty > received qty → over_invoice discrepancy`
- `Unit: no receipt + 3-way required → missing_receipt discrepancy`
- `Integration (real): clean invoice → matched + InvoiceThreeWayMatched event`
- `Integration (real): exception invoice → disputed + discrepancies persisted`

---

## Phase 7: Contracts, Budgets & Spend Analytics

### Purpose
Add the spend-management layer: contract repository with expiry alerts, budget guardrails at request time, partitioned spend records, and analytics dashboards by category/vendor/business-unit. Delivers the finance and CPO visibility that features.md flags as table-stakes.

### Tasks

#### 7.1 — Contracts with expiry alerts

**What**: `contracts` per Suggestion 3 (`contract_details` JSONB) and a Celery beat expiry sweep.

**Design**:
- `contracts` with `start_date`/`end_date`/`status`. Nightly task scans `end_date - alert_days_before_expiry <= today` and emits `ContractExpiryAlertSent` + notification. Status auto-transitions to `expired` past `end_date`.
- POs may reference a contract; PO under an active contract is non-maverick by definition.

**Testing**:
- `Integration (real): contract ending in 20 days with 30-day alert → alert emitted once (idempotent)`
- `Unit: end_date in past → status expired on sweep`

#### 7.2 — Budgets & point-of-request guardrails

**What**: `budgets` per Suggestion 3 with available-budget calculation.

**Design**:
- `available = allocated_amount - spent_amount - pending_commitments`. `budget_service.check(org, category, fiscal_year, amount)` returns `{available, would_exceed: bool}` surfaced when creating a PO/request. Exceeding raises a soft warning (configurable to hard-block in org `settings`).

**Testing**:
- `Unit: PO exceeding remaining budget → would_exceed=true`
- `Integration (real): hard-block setting → 409 on over-budget PO submit`

#### 7.3 — Spend records & maverick detection

**What**: Partitioned `spend_records` per Suggestion 3, populated from approved POs/matched invoices via domain-event handlers.

**Design**:
- Partition by `spend_date` (annual partitions). `is_maverick=true` when spend has no linked sourcing event AND no contract AND amount > org threshold (mirrors Suggestion 4's maverick query, implemented in SQL). Multi-currency: store native `amount` + `amount_usd` (FX at record time).

**Testing**:
- `Integration (real): PO with no event/contract over threshold → spend_record.is_maverick=true`
- `Unit: amount_usd computed from FX rate at spend_date`

#### 7.4 — Analytics endpoints

**What**: Aggregation endpoints powering dashboards.

**Design**:
- `GET /v1/analytics/spend?group_by=category|vendor|business_unit&from=&to=` → grouped sums, transaction counts, maverick amount. `GET /v1/analytics/spend/trend?interval=month`. Backed by SQL `GROUP BY` over partitioned `spend_records`; a materialized view `mv_spend_by_category` (the `proj_spend_by_category` shape from Suggestion 2) refreshed by Celery for heavy dashboards.

**Testing**:
- `Integration (real): spend grouped by category sums correctly across partitions`
- `Integration (real): date-range filter excludes out-of-range records`
- `Integration (real): tenant isolation — analytics never include other orgs' spend`

---

## Phase 8: Standards Integration — cXML, UBL, PEPPOL, EDI, ERP

### Purpose
Make the platform interoperable with enterprise procurement and ERP ecosystems via open standards. This is a baseline market expectation (features.md) and, for EU buyers, a near-term mandate (PEPPOL e-invoicing). Can be developed in parallel with Phase 9 once Phase 6 exists.

### Tasks

#### 8.1 — cXML PunchOut 2.0 (supplier catalog exposure)

**What**: PunchOut SetupRequest/Response and cart-return per cXML 1.2.069.

**Design**:
- `POST /v1/cxml/punchout/setup` parses a `PunchOutSetupRequest` (lxml), authenticates the buyer via `<Credential>` against `integration_configs`, returns a `PunchOutSetupResponse` with a session start URL. On checkout the platform POSTs a `PunchOutOrderMessage` back to the buyer's `BrowserFormPost` URL.
- cXML serialisation/parsing isolated in `integrations/cxml/`.

**Testing**:
- `Fixture-based: parse sample PunchOutSetupRequest fixture → extracts credentials, buyer cookie`
- `Unit: invalid credential → cXML 401 Status response`
- `Fixture-based: generated PunchOutOrderMessage validates against cXML DTD`

#### 8.2 — UBL 2.1 & PEPPOL BIS Billing 3.0

**What**: Serialise/parse Order and Invoice as UBL 2.1; PEPPOL BIS 3.0 invoice export.

**Design**:
- `ubl.order_from_po(po) -> bytes` and `ubl.invoice_from_invoice(inv) -> bytes` emit EN 16931-conformant UBL. `peppol.export_invoice` wraps in BIS Billing 3.0 with required `cbc:CustomizationID`/`ProfileID`. Endpoint `GET /v1/invoices/{id}/ubl` and `/peppol`.

**Testing**:
- `Fixture-based: UBL invoice validates against UBL 2.1 XSD`
- `Unit: PEPPOL export sets correct CustomizationID for BIS Billing 3.0`
- `Fixture-based: round-trip parse of sample UBL Order → SourcingEvent/PO fields`

#### 8.3 — EDI X12 850/810 & EDIFACT ORDERS/INVOIC

**What**: Map POs to X12 850 / EDIFACT ORDERS and invoices to 810 / INVOIC.

**Design**:
- Mapping layer in `integrations/edi/` with segment builders. Inbound parsing tolerant of trading-partner variation (stored in `integration_configs.config.field_mappings`).

**Testing**:
- `Fixture-based: PO → X12 850 with correct ST/BEG/PO1 segments`
- `Fixture-based: parse sample 810 → Invoice with matching totals`

#### 8.4 — ERP connector framework

**What**: Pluggable connector interface with SAP/Oracle/NetSuite/Workday adapters (NetSuite first, others stubbed).

**Design**:
```python
class ERPConnector(Protocol):
    async def push_po(self, po: PurchaseOrder) -> ERPRef: ...
    async def pull_invoices(self, since: datetime) -> list[InvoicePayload]: ...
```
- Config and credentials referenced (never stored raw) via `integration_configs.config.credentials_ref` (vault path). Celery tasks sync on schedule.

**Testing**:
- `Integration (mocked NetSuite API): push_po posts mapped payload, stores ERPRef`
- `Unit: connector registry resolves integration_type → adapter`

---

## Phase 9: AI-Native Features

### Purpose
Deliver the differentiating AI layer: natural-language RFQ generation, AI bid scoring/shortlisting, contract risk extraction, embedding-based supplier matching, and maverick-spend anomaly detection. These are the headline advantages over incumbents and depend on the transactional core (Phases 3–7). Parallelisable with Phase 8.

### Tasks

#### 9.1 — LLM client abstraction & prompt management

**What**: Provider-agnostic `LLMClient` with prompt caching, structured-output parsing, and cost/usage logging.

**Design**:
```python
class LLMClient(Protocol):
    async def complete(self, *, system: str, user: str, schema: type[BaseModel] | None,
                       cache_system: bool = True) -> LLMResult: ...
```
- Anthropic implementation uses prompt caching on stable system prompts and tool-use for structured output. Versioned prompts in `ai/prompts/`. All calls logged (tokens, latency, model) to an `ai_invocations` table for auditability and cost tracking. Configurable model; self-hosted endpoint supported for on-prem.

**Testing**:
- `Unit (mocked SDK): complete returns parsed schema; malformed output → retry then error`
- `Unit: usage logged to ai_invocations`

#### 9.2 — RFQ generation from natural language

**What**: Turn a plain-language buyer brief into a structured draft sourcing event + lines.

**Design**:
- `POST /v1/sourcing-events/generate` (body: `prompt`, optional `category`, `currency`). System prompt instructs the model to output a `GeneratedRFQ` schema (title, description, evaluation_criteria, lines[{description, quantity, uom, target_price?, specifications}]). Result persisted as a `draft` event with `details.ai_generated_spec` set for transparency/editability. Human always reviews before publish.

**Testing**:
- `Unit (mocked LLM): brief → GeneratedRFQ with ≥1 line, persisted as draft`
- `Unit: model output failing schema validation → 422 with guidance, no event created`
- `Integration (real, optional/marked): live LLM produces valid schema for a sample brief`

#### 9.3 — Bid scoring & shortlist

**What**: AI scores and summarises competing bids with confidence and explainability.

**Design**:
- On event `closed`, a Celery task scores each bid against `evaluation_config` weights (price, quality, lead_time, esg). Writes `bids.evaluation = {ai_score, ai_confidence, scoring_breakdown, rationale}`. Comparison endpoint (4.3) surfaces scores. Reasoning text stored for the explainability gap noted in features.md.

**Testing**:
- `Unit (mocked LLM): three bids scored, ranked desc by ai_score; breakdown sums to weights`
- `Integration (real): scores written to bid.evaluation JSONB and appear in comparison`

#### 9.4 — Embedding-based supplier matching

**What**: Recommend best-fit suppliers for an event/line using pgvector similarity over capability/category/performance embeddings.

**Design**:
- `supplier_embeddings(supplier_id, embedding vector(1536), updated_at)`. Embeddings computed from supplier `capabilities`/`categories`/ratings. `GET /v1/suppliers/match?event_id=` embeds the event requirement and runs `ORDER BY embedding <=> :query LIMIT k`, re-ranked by compliance status and avg rating.

**Testing**:
- `Integration (real, pgvector): seeded suppliers → match returns nearest by cosine distance`
- `Unit: suppliers missing required certification filtered from results`

#### 9.5 — Contract risk extraction

**What**: NLP flags non-standard terms, missing clauses, and unfavourable conditions in uploaded contracts.

**Design**:
- `POST /v1/contracts/{id}/analyze` extracts text (uploaded file), prompts the model against a clause checklist, writes `contract_details.ai_risk_flags = [{clause, severity, excerpt, recommendation}]`. Surfaced in the contract view.

**Testing**:
- `Unit (mocked LLM): contract missing liability clause → flag severity 'high'`
- `Integration (real): risk flags persisted and queryable via GIN index`

#### 9.6 — Maverick spend anomaly detection

**What**: Real-time detection of off-contract/duplicate spend against policy.

**Design**:
- A domain-event handler on `SpendRecorded` evaluates rules (off-contract, duplicate invoice signature, out-of-policy category) plus a statistical outlier check; flags emit `MaverickSpendDetected` and surface on the analytics dashboard. Rule-based core (deterministic, testable); LLM used only for natural-language policy interpretation.

**Testing**:
- `Unit: duplicate invoice signature within window → flagged`
- `Unit: spend in disallowed category for org → flagged`
- `Integration (real): MaverickSpendDetected increments dashboard maverick_amount`

---

## Phase 10: Real-Time eAuctions

### Purpose
Add the competitive-bidding module (reverse/forward/Dutch/Japanese) with live WebSocket updates and auto-extension. A backlog/differentiator feature (features.md), built after the core sourcing engine and notification infra exist.

### Tasks

#### 10.1 — Auction & auction-bid models

**What**: `auctions` and `auction_bids` per Suggestion 3 (`rules` JSONB).

**Design**:
- Linked to a `sourcing_event`. Status: `scheduled → active → (extended) → closed`. `rules` JSONB: auto_extend, extend_minutes, min_decrement, reserve_price, lot_structure. Current best and ranks maintained in Redis for low-latency reads.

**Testing**:
- `Unit: reverse auction accepts bid only if amount ≤ current_best - min_decrement`
- `Unit: bid within extend_minutes of end → end_time extended, status 'extended'`

#### 10.2 — Live bidding WebSocket

**What**: WebSocket channel broadcasting bids/rank/time updates.

**Design**:
- `WS /v1/auctions/{id}/live`. Bids placed via `POST /v1/auctions/{id}/bids` validated against rules, persisted, pushed to Redis pub/sub, fanned out to subscribers. On close, best bid maps to award → PO (reuses Phase 5).

**Testing**:
- `Integration (real, ws): two suppliers bidding → both receive rank updates`
- `Integration (real): bid after close → 409`
- `Unit: concurrent equal bids resolved by timestamp ordering`

---

## Phase 11: Frontend Portals

### Purpose
Deliver the consumer-grade buyer and supplier web experiences — the headline market gap. Built against the now-stable API. Buyer intake, sourcing, comparison, PO/approval, dashboards; supplier onboarding, invitations, bidding, invoices.

### Tasks

#### 11.1 — App shell, auth & API client

**What**: Next.js shell with OIDC/OAuth login, role-aware navigation, typed API client.

**Design**:
- Typed client generated from the exported OpenAPI 3.1 spec. Auth via authorization-code+PKCE; tokens stored per Next.js best practice (httpOnly cookies via route handlers). shadcn/ui + Tailwind. Buyer vs supplier navigation driven by `org_type`/roles.

**Testing**:
- `Component (Vitest): nav renders supplier-only items for supplier principal`
- `E2E (Playwright): login → dashboard redirect`

#### 11.2 — Buyer workflows

**What**: RFQ wizard (incl. AI-generate), invitation management, bid comparison matrix, PO/approval inbox, spend dashboards.

**Design**:
- Sourcing wizard with progressive disclosure (Zip-style). Comparison view consumes `GET .../comparison` with AI score cards. Approval inbox lists pending `approval_requests` for the principal.

**Testing**:
- `E2E (Playwright): create RFQ via wizard → publish → appears in list`
- `E2E: AI-generate RFQ from prompt → editable draft`
- `E2E: approver approves PO → status updates`

#### 11.3 — Supplier workflows

**What**: Onboarding (profile, compliance docs, categories), invitation response, bid submission, invoice submission.

**Design**:
- Compliance document upload to S3 with type/expiry; bid form mirrors event lines; invoice form prefilled from PO.

**Testing**:
- `E2E (Playwright): supplier accepts invitation → submits bid → buyer sees it after close`
- `E2E: upload compliance doc → appears with expiry`

---

## Phase 12: Agentic Interface (MCP), Hardening & Observability

### Purpose
Expose core actions to AI agents via MCP (positioning for the agentic-commerce wave per standards.md), and harden the platform: OWASP API Top 10 review, rate limiting, audit completeness, observability, and deployment readiness.

### Tasks

#### 12.1 — MCP server

**What**: Expose `search_suppliers`, `create_rfq`, `get_spend_summary`, `approve_po`, `match_suppliers` as MCP tools.

**Design**:
- `mcp/server.py` registers tools that call existing services under an authenticated principal scoped by OAuth client. Approvals/POs above a threshold require human confirmation (tool returns a confirmation token).

**Testing**:
- `Integration (mocked MCP client): create_rfq tool → draft event created under correct org`
- `Unit: approve_po above auto threshold → returns confirmation-required, no state change`

#### 12.2 — Rate limiting, idempotency & security hardening

**What**: Redis-backed rate limits, idempotency keys on POST, and an OWASP API Top 10 pass.

**Design**:
- Per-client/IP rate limits (token bucket in Redis). `Idempotency-Key` header on mutating endpoints stored 24h. Audit: confirm BOLA coverage (every object fetch checks org ownership), mass-assignment guards (server-computed monetary fields), and that no secrets are logged.

**Testing**:
- `Integration (real): exceeding rate limit → 429 with Retry-After`
- `Integration (real): replayed Idempotency-Key → original response, no duplicate PO`
- `Security: enumerate other org's PO IDs → all 404`

#### 12.3 — Observability & deployment

**What**: Structured logging, metrics, tracing, and production deploy artifacts.

**Design**:
- JSON structured logs with correlation IDs (propagated from domain-event metadata). Prometheus metrics (request latency, queue depth, LLM cost). OpenTelemetry tracing. Helm chart / production compose with secrets via env/vault.

**Testing**:
- `Integration: request emits log with correlation_id matching audit_log entry`
- `E2E (CI): production compose boots; /metrics scraped`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding         ─── required by everything
    │
Phase 2: Identity, Orgs & Access Control  ─── requires 1
    │
Phase 3: Sourcing (RFQ/RFP/RFI)           ─── requires 2
    │
Phase 4: Bidding, Comparison & Award      ─── requires 3
    │
Phase 5: Purchase Orders & Approvals      ─── requires 4
    │
Phase 6: Receiving, Invoicing & 3-Way Match ── requires 5
    │
    ├── Phase 7: Contracts, Budgets, Spend Analytics ── requires 6
    │
    ├── Phase 8: Standards Integration (cXML/UBL/PEPPOL/EDI/ERP) ── requires 6 · parallel with 9
    │
    └── Phase 9: AI-Native Features ── requires 3–7 · parallel with 8
             │
Phase 10: Real-Time eAuctions  ── requires 4 + 5 (award→PO) + notification infra
Phase 11: Frontend Portals     ── requires a stable API (best after 3–9)
Phase 12: MCP, Hardening, Observability ── requires core actions (2–7); 12.1 benefits from 9
```

**Parallelism opportunities**
- Phases 8 and 9 can be developed concurrently once Phase 6 is complete (independent surfaces over the same transactional core).
- Phase 11 (frontend) can begin against each API surface as it stabilises (incremental, per-domain) rather than waiting for all backend phases.
- Phase 10 can proceed in parallel with 7/8 once 4–5 are done.

**Estimated scope: large** (12 phases, 38 tasks; full-stack platform with multiple standards integrations and an AI layer).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and mocked-integration tests pass; real-integration tests pass under testcontainers in CI.
3. `ruff check`, `ruff format --check`, and `mypy src` (strict) pass with no errors.
4. New endpoints appear in the auto-generated OpenAPI 3.1 spec, and the exported spec in `openapi/` is regenerated.
5. New/changed tables have an Alembic migration that applies and downgrades cleanly against a fresh database.
6. Tenant isolation (RLS) and authorization (`require_roles`/`require_scope`) are enforced and tested for every new tenant-scoped endpoint (OWASP API A01).
7. Monetary and status fields are server-computed, never accepted from the client (OWASP API mass-assignment).
8. New config options are added to `config.py` and documented in `.env.example`.
9. `docker compose up` boots the affected services and `/healthz` returns 200.
10. New standards-conformant outputs (cXML, UBL, PEPPOL, EDI) validate against their schemas/DTDs via fixture-based tests.
11. Frontend phases additionally: component tests (Vitest) and Playwright E2E for the new flow pass.
12. Domain events emitted by the phase are written to `audit_log` with correlation IDs.
```
