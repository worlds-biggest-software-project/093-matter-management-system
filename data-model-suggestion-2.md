# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Matter Management System · Created: 2026-05-12

## Philosophy

This model treats every change to the system as an immutable domain event stored in an append-only event store. The current state of any entity — a matter, a time entry, an invoice, a trust account — is derived by replaying its event stream. Read-optimised projections (materialised views) serve the application's query needs, while the event store remains the single source of truth. This is the CQRS (Command Query Responsibility Segregation) pattern: commands produce events, queries read from projections.

This architecture is inspired by financial ledger systems, audit-grade compliance platforms, and event-sourced fintech systems. In legal practice management, the pattern is particularly compelling because every action on a matter — opening it, assigning an attorney, recording time, generating an invoice, making a trust deposit — is a discrete event with legal significance. The legal profession's ethical obligations (ABA Model Rules, IOLTA trust accounting, conflict-of-interest documentation) all demand the ability to answer "what was the state of this matter on date X?" — a question that event sourcing answers natively.

The event-sourced model is best for deployments where a complete, tamper-evident audit trail is a primary requirement — corporate in-house legal departments, government legal offices, and firms subject to regulatory oversight.

**Best for:** Environments requiring tamper-evident audit trails, temporal queries ("state as of date X"), and AI-powered analytics on change patterns.

**Trade-offs:**
- Pro: Complete, immutable audit history is built into the architecture — not bolted on as an afterthought
- Pro: Temporal queries are natural — reconstruct any entity's state at any point in time by replaying events up to that timestamp
- Pro: AI analytics on event streams (pattern detection, anomaly alerting) are straightforward since every change is a structured event
- Pro: Trust accounting three-way reconciliation benefits from an immutable transaction ledger
- Pro: Enables "undo" and "what-if" scenarios by replaying events with modifications
- Con: Higher implementation complexity — developers must think in events rather than CRUD
- Con: Projections must be maintained and can become stale or require rebuilding
- Con: Storage requirements are higher — events accumulate indefinitely
- Con: Simple queries require reading from projections rather than querying the source tables directly
- Con: Schema evolution of events requires careful versioning (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SALI LMSS v2 | Matter classification events include SALI IRI identifiers; projection tables store resolved SALI tags for querying |
| LEDES XML 2.2 | Invoice projections mirror LEDES segment hierarchy; LEDES export reads from the invoice projection rather than reconstructing from events |
| UTBMS Codes | Time entry events include UTBMS task/activity codes; projections enable aggregate billing analytics |
| ABA Model Rule 1.15 | Trust accounting events form an immutable ledger; three-way reconciliation reads from the event stream with cryptographic verification |
| ISO 3166 | Jurisdiction codes embedded in matter events and trust account events |
| OCSF (Open Cybersecurity Schema Framework) | Event schema borrows structural patterns from OCSF for consistent event metadata (actor, action, target, timestamp) |
| OAuth 2.0 / OIDC | Authentication events recorded in the event store; token management via separate operational tables |

---

## Event Store (Source of Truth)

```sql
-- The immutable event store — append-only, never updated or deleted
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    -- Stream identification
    stream_type     VARCHAR(50) NOT NULL,             -- matter, contact, time_entry, invoice, trust_account, user
    stream_id       UUID NOT NULL,                    -- the aggregate/entity ID
    sequence_num    BIGINT NOT NULL,                  -- monotonically increasing per stream
    -- Event metadata
    event_type      VARCHAR(100) NOT NULL,            -- e.g., "matter.opened", "time_entry.recorded", "invoice.approved"
    event_version   INTEGER NOT NULL DEFAULT 1,       -- schema version for this event type
    -- Event payload
    data            JSONB NOT NULL,                   -- the event-specific payload
    metadata        JSONB NOT NULL DEFAULT '{}',      -- actor, ip, user_agent, correlation_id
    -- Causation chain
    correlation_id  UUID,                             -- ties related events together (e.g., all events from one API request)
    causation_id    UUID,                             -- the event that caused this event
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Integrity
    checksum        VARCHAR(64),                      -- SHA-256 of (previous_checksum + data) for tamper detection
    UNIQUE(stream_type, stream_id, sequence_num)
);

-- Partition by month for storage management and query performance
CREATE TABLE events_2026_05 PARTITION OF events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Primary query patterns
CREATE INDEX idx_events_stream ON events(stream_type, stream_id, sequence_num);
CREATE INDEX idx_events_type ON events(event_type, occurred_at);
CREATE INDEX idx_events_tenant_time ON events(tenant_id, occurred_at);
CREATE INDEX idx_events_correlation ON events(correlation_id);

-- Subscriptions for projection rebuilding
CREATE TABLE event_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_name VARCHAR(100) NOT NULL UNIQUE,   -- e.g., "matters_projection", "billing_projection"
    last_processed_id UUID,                           -- last event ID processed by this subscription
    last_processed_at TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Type Catalogue

```sql
-- Registry of all known event types with their JSON schemas
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    event_version   INTEGER NOT NULL DEFAULT 1,
    description     TEXT NOT NULL,
    json_schema     JSONB NOT NULL,                   -- JSON Schema for the event data payload
    stream_type     VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event type registrations:
-- matter.opened, matter.classified, matter.assigned, matter.stage_changed,
-- matter.closed, matter.reopened
-- contact.created, contact.updated, contact.role_assigned, contact.relationship_added
-- time_entry.recorded, time_entry.approved, time_entry.adjusted, time_entry.written_off
-- invoice.created, invoice.line_item_added, invoice.approved, invoice.sent,
-- invoice.payment_received, invoice.voided
-- trust.deposit_received, trust.disbursement_made, trust.transfer_executed,
-- trust.reconciliation_completed
-- conflict.check_performed, conflict.result_reviewed, conflict.waiver_obtained
```

### Example Event Payloads

```sql
-- Example: matter.opened event
-- data: {
--   "matter_number": "2026-00142",
--   "display_name": "Smith v. Jones",
--   "sali_matter_type_iri": "https://lmss.sali.org/R3B5C7D9E1",
--   "sali_area_of_law_iri": "https://lmss.sali.org/A2B4C6D8E0",
--   "utbms_code_set": "litigation",
--   "jurisdiction": {"country": "US", "subdivision": "US-CA"},
--   "billing_method": "hourly",
--   "responsible_attorney_id": "uuid-here",
--   "originating_attorney_id": "uuid-here",
--   "client_contact_id": "uuid-here",
--   "template_id": "uuid-here"
-- }
-- metadata: {
--   "actor_id": "uuid-of-user",
--   "actor_email": "attorney@firm.com",
--   "ip_address": "192.168.1.1",
--   "user_agent": "Mozilla/5.0...",
--   "correlation_id": "uuid-for-request"
-- }

-- Example: time_entry.recorded event
-- data: {
--   "matter_id": "uuid-of-matter",
--   "date": "2026-05-12",
--   "duration_hours": 1.5,
--   "duration_raw": 1.42,
--   "description": "Drafted motion to dismiss",
--   "utbms_task_code": "L120",
--   "utbms_activity_code": "A104",
--   "rate": 450.00,
--   "total": 675.00,
--   "is_billable": true,
--   "source": "timer"
-- }

-- Example: trust.deposit_received event
-- data: {
--   "trust_account_id": "uuid",
--   "client_contact_id": "uuid",
--   "matter_id": "uuid",
--   "amount": 5000.00,
--   "deposit_method": "wire",
--   "reference": "Wire-20260512-001",
--   "running_balance": 12500.00
-- }

-- Example: trust.reconciliation_completed event
-- data: {
--   "trust_account_id": "uuid",
--   "reconciliation_date": "2026-04-30",
--   "bank_statement_balance": 125000.00,
--   "book_balance": 125000.00,
--   "client_ledger_total": 125000.00,
--   "outstanding_deposits": 0.00,
--   "outstanding_checks": 0.00,
--   "is_balanced": true
-- }
```

---

## Read Model Projections

### Operational Tables (for multi-tenancy and auth — not event-sourced)

```sql
-- These tables are NOT derived from events — they are operational infrastructure
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'user',
    bar_number      VARCHAR(50),
    hourly_rate     NUMERIC(10,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
```

### Matter Projection

```sql
-- Materialised from matter.* events
CREATE TABLE proj_matters (
    id              UUID PRIMARY KEY,                 -- same as stream_id in events
    tenant_id       UUID NOT NULL,
    matter_number   VARCHAR(50) NOT NULL,
    display_name    VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL,
    -- SALI
    sali_matter_type_iri  VARCHAR(500),
    sali_matter_type_label VARCHAR(255),
    sali_area_of_law_iri  VARCHAR(500),
    sali_area_of_law_label VARCHAR(255),
    sali_industry_iri     VARCHAR(500),
    sali_industry_label   VARCHAR(255),
    -- UTBMS
    utbms_code_set  VARCHAR(30),
    -- Jurisdiction
    jurisdiction_country  CHAR(2),
    jurisdiction_subdivision VARCHAR(6),
    court_name      VARCHAR(255),
    case_number     VARCHAR(100),
    -- Dates
    date_opened     DATE NOT NULL,
    date_closed     DATE,
    statute_of_limitations DATE,
    -- Billing
    billing_method  VARCHAR(30) NOT NULL,
    budget_amount   NUMERIC(12,2),
    -- Assignment
    responsible_attorney_id UUID,
    originating_attorney_id UUID,
    practice_group  VARCHAR(100),
    pipeline_stage  VARCHAR(100),
    -- Projection metadata
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    version         BIGINT NOT NULL,                  -- sequence_num of last applied event
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_matters_tenant ON proj_matters(tenant_id);
CREATE INDEX idx_proj_matters_status ON proj_matters(tenant_id, status);
CREATE INDEX idx_proj_matters_pipeline ON proj_matters(tenant_id, pipeline_stage);
CREATE INDEX idx_proj_matters_attorney ON proj_matters(responsible_attorney_id);
```

### Contact Projection

```sql
CREATE TABLE proj_contacts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    contact_type    VARCHAR(20) NOT NULL,
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    company_name    VARCHAR(500),
    lei             VARCHAR(20),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_event_id   UUID NOT NULL,
    version         BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_contacts_tenant ON proj_contacts(tenant_id);
CREATE INDEX idx_proj_contacts_name ON proj_contacts(tenant_id, last_name, first_name);

-- Matter-contact role projection
CREATE TABLE proj_matter_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL,
    contact_id      UUID NOT NULL,
    role            VARCHAR(50) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    UNIQUE(matter_id, contact_id, role)
);

CREATE INDEX idx_proj_mc_matter ON proj_matter_contacts(matter_id);
CREATE INDEX idx_proj_mc_contact ON proj_matter_contacts(contact_id);

-- Contact relationship projection (for conflict checking)
CREATE TABLE proj_contact_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    from_contact_id UUID NOT NULL,
    to_contact_id   UUID NOT NULL,
    relationship    VARCHAR(50) NOT NULL,
    start_date      DATE,
    end_date        DATE
);

CREATE INDEX idx_proj_crel_from ON proj_contact_relationships(from_contact_id);
CREATE INDEX idx_proj_crel_to ON proj_contact_relationships(to_contact_id);
```

### Time Entry Projection

```sql
CREATE TABLE proj_time_entries (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    matter_id       UUID NOT NULL,
    user_id         UUID NOT NULL,
    date            DATE NOT NULL,
    duration_hours  NUMERIC(6,2) NOT NULL,
    description     TEXT NOT NULL,
    utbms_task_code VARCHAR(10),
    utbms_activity_code VARCHAR(10),
    rate            NUMERIC(10,2) NOT NULL,
    total           NUMERIC(12,2) NOT NULL,
    is_billable     BOOLEAN NOT NULL,
    is_billed       BOOLEAN NOT NULL DEFAULT false,
    invoice_id      UUID,
    source          VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    last_event_id   UUID NOT NULL,
    version         BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_te_matter ON proj_time_entries(matter_id);
CREATE INDEX idx_proj_te_user ON proj_time_entries(user_id);
CREATE INDEX idx_proj_te_date ON proj_time_entries(tenant_id, date);
CREATE INDEX idx_proj_te_unbilled ON proj_time_entries(matter_id) WHERE is_billable AND NOT is_billed;
```

### Invoice Projection

```sql
CREATE TABLE proj_invoices (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    invoice_number  VARCHAR(50) NOT NULL,
    matter_id       UUID NOT NULL,
    client_id       UUID NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    total_fees      NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_expenses  NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_tax       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(12,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(12,2) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL,
    last_event_id   UUID NOT NULL,
    version         BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_inv_matter ON proj_invoices(matter_id);
CREATE INDEX idx_proj_inv_status ON proj_invoices(tenant_id, status);

CREATE TABLE proj_invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL,
    line_type       VARCHAR(10) NOT NULL,
    time_entry_id   UUID,
    description     TEXT NOT NULL,
    date            DATE NOT NULL,
    utbms_task_code VARCHAR(10),
    utbms_activity_code VARCHAR(10),
    utbms_expense_code VARCHAR(10),
    quantity        NUMERIC(8,2) NOT NULL,
    rate            NUMERIC(10,2) NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    timekeeper_id   UUID,
    timekeeper_classification VARCHAR(50)
);

CREATE INDEX idx_proj_ili_invoice ON proj_invoice_line_items(invoice_id);
```

### Trust Accounting Projection

```sql
CREATE TABLE proj_trust_accounts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    account_name    VARCHAR(255) NOT NULL,
    bank_name       VARCHAR(255) NOT NULL,
    account_number_last4 CHAR(4) NOT NULL,
    jurisdiction_country CHAR(2) NOT NULL,
    jurisdiction_subdivision VARCHAR(6),
    is_iolta        BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    current_balance NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    version         BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_trust_client_ledgers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL,
    contact_id      UUID NOT NULL,
    matter_id       UUID,
    balance         NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    version         BIGINT NOT NULL
);

-- Trust transaction log projection (derived from trust.* events)
CREATE TABLE proj_trust_transactions (
    id              UUID PRIMARY KEY,
    trust_account_id UUID NOT NULL,
    client_ledger_id UUID NOT NULL,
    transaction_type VARCHAR(20) NOT NULL,
    amount          NUMERIC(14,2) NOT NULL,
    running_balance NUMERIC(14,2) NOT NULL,
    description     TEXT NOT NULL,
    reference       VARCHAR(255),
    transaction_date DATE NOT NULL,
    -- Source event for full traceability
    source_event_id UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_ttx_account ON proj_trust_transactions(trust_account_id);
CREATE INDEX idx_proj_ttx_ledger ON proj_trust_transactions(client_ledger_id);
CREATE INDEX idx_proj_ttx_date ON proj_trust_transactions(transaction_date);

-- Trust reconciliation projection
CREATE TABLE proj_trust_reconciliations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL,
    reconciliation_date DATE NOT NULL,
    bank_statement_balance NUMERIC(14,2) NOT NULL,
    book_balance    NUMERIC(14,2) NOT NULL,
    client_ledger_total NUMERIC(14,2) NOT NULL,
    is_balanced     BOOLEAN NOT NULL,
    discrepancy     NUMERIC(14,2) NOT NULL DEFAULT 0,
    reconciled_by   UUID NOT NULL,
    source_event_id UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL
);
```

### Document & Task Projections

```sql
CREATE TABLE proj_documents (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    matter_id       UUID,
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    title           VARCHAR(500),
    category        VARCHAR(100),
    version         INTEGER NOT NULL DEFAULT 1,
    is_current_version BOOLEAN NOT NULL DEFAULT true,
    uploaded_by     UUID NOT NULL,
    tags            TEXT[],                           -- denormalised for faster search
    last_event_id   UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_docs_matter ON proj_documents(matter_id);
CREATE INDEX idx_proj_docs_tags ON proj_documents USING GIN(tags);

CREATE TABLE proj_tasks (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    matter_id       UUID,
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    priority        VARCHAR(10) NOT NULL,
    due_date        DATE,
    assigned_to     UUID,
    last_event_id   UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_tasks_matter ON proj_tasks(matter_id);
CREATE INDEX idx_proj_tasks_assigned ON proj_tasks(assigned_to, status);
CREATE INDEX idx_proj_tasks_due ON proj_tasks(tenant_id, due_date) WHERE status NOT IN ('completed', 'cancelled');
```

---

## Temporal Query Examples

```sql
-- Reconstruct a matter's state as of a specific date
-- "What was the status and assigned attorney of matter X on March 15, 2026?"
SELECT data, occurred_at
FROM events
WHERE stream_type = 'matter'
  AND stream_id = 'uuid-of-matter'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence_num ASC;
-- Application replays these events to reconstruct state at that point in time

-- Find all changes to a specific matter in the last 30 days
SELECT event_type, data, metadata, occurred_at
FROM events
WHERE stream_type = 'matter'
  AND stream_id = 'uuid-of-matter'
  AND occurred_at >= now() - INTERVAL '30 days'
ORDER BY sequence_num ASC;

-- Trust account balance as of any historical date
-- Replay trust.deposit_received and trust.disbursement_made events up to the target date
SELECT
    stream_id AS trust_account_id,
    SUM(
        CASE
            WHEN event_type IN ('trust.deposit_received', 'trust.transfer_in')
                THEN (data->>'amount')::NUMERIC
            WHEN event_type IN ('trust.disbursement_made', 'trust.transfer_out')
                THEN -(data->>'amount')::NUMERIC
            ELSE 0
        END
    ) AS balance_as_of_date
FROM events
WHERE stream_type = 'trust_account'
  AND stream_id = 'uuid-of-trust-account'
  AND occurred_at <= '2026-03-31T23:59:59Z'
GROUP BY stream_id;

-- AI analytics: find matters with unusual billing patterns
-- (matters where time entries were adjusted more than 3 times)
SELECT stream_id AS time_entry_id,
       COUNT(*) AS adjustment_count
FROM events
WHERE event_type = 'time_entry.adjusted'
  AND tenant_id = 'uuid-of-tenant'
  AND occurred_at >= now() - INTERVAL '90 days'
GROUP BY stream_id
HAVING COUNT(*) > 3
ORDER BY adjustment_count DESC;
```

---

## Projection Rebuild Process

```sql
-- Projection rebuild function (pseudocode — implemented in application code)
-- 1. Truncate the projection table
-- 2. Read all events for the relevant stream type, ordered by sequence_num
-- 3. Apply each event to rebuild the projection row
-- 4. Update the event_subscriptions table with the last processed event ID

-- The application maintains a projection builder per projection:
-- MatterProjectionBuilder: listens for matter.* events, updates proj_matters
-- TimeEntryProjectionBuilder: listens for time_entry.* events, updates proj_time_entries
-- InvoiceProjectionBuilder: listens for invoice.* events, updates proj_invoices + proj_invoice_line_items
-- TrustProjectionBuilder: listens for trust.* events, updates proj_trust_* tables
-- ContactProjectionBuilder: listens for contact.* events, updates proj_contacts + proj_matter_contacts

-- Event processing is idempotent — replaying the same event produces the same result
-- Subscriptions track the last processed event ID to enable incremental catch-up
```

---

## Tamper Detection

```sql
-- Verify event chain integrity for a given stream
-- Each event's checksum = SHA-256(previous_checksum || event_data)
-- Application code validates the chain on-demand or on a scheduled basis

-- Example verification query (chain integrity check)
SELECT
    e1.id,
    e1.sequence_num,
    e1.checksum,
    e1.data,
    LAG(e1.checksum) OVER (ORDER BY e1.sequence_num) AS previous_checksum,
    CASE
        WHEN e1.sequence_num = 1 THEN true  -- first event has no predecessor
        ELSE e1.checksum = encode(
            digest(
                COALESCE(LAG(e1.checksum) OVER (ORDER BY e1.sequence_num), '') ||
                e1.data::TEXT,
                'sha256'
            ),
            'hex'
        )
    END AS is_valid
FROM events e1
WHERE stream_type = 'trust_account'
  AND stream_id = 'uuid-of-trust-account'
ORDER BY sequence_num;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | events (partitioned), event_subscriptions, event_type_registry |
| Operational (non-event-sourced) | 2 | tenants, users |
| Matter Projections | 2 | proj_matters, proj_matter_contacts |
| Contact Projections | 2 | proj_contacts, proj_contact_relationships |
| Time Entry Projections | 1 | proj_time_entries |
| Invoice Projections | 2 | proj_invoices, proj_invoice_line_items |
| Trust Projections | 4 | proj_trust_accounts, proj_trust_client_ledgers, proj_trust_transactions, proj_trust_reconciliations |
| Document & Task Projections | 2 | proj_documents, proj_tasks |
| **Total** | **18** | Plus event table partitions |

---

## Key Design Decisions

1. **Single event store table rather than per-aggregate event tables** — all events go into one `events` table (partitioned by month), identified by `stream_type` and `stream_id`. This simplifies cross-aggregate queries (e.g., "all events for tenant X in the last hour") and makes event subscription management uniform. The trade-off is that the table grows large, but monthly partitioning and the append-only nature make this manageable.

2. **Checksum chain for tamper detection** — each event stores a SHA-256 checksum computed from the previous event's checksum plus the current event's data. This creates a hash chain similar to a blockchain, enabling cryptographic verification that no events have been modified or deleted. This is critical for trust accounting and regulatory compliance.

3. **Projections are disposable and rebuildable** — every projection table can be dropped and rebuilt from the event store. This means projection schema changes (adding columns, changing indexes) do not require data migrations — just rebuild the projection. The `event_subscriptions` table tracks the last processed event per projection to enable incremental updates.

4. **SALI IRIs stored directly in events** — rather than foreign-keying to SALI reference tables (which would create a dependency between the event store and reference data), SALI IRIs are stored as strings in event payloads. The projection builder resolves them to labels when building the projection.

5. **Trust accounting is the strongest fit for event sourcing** — the trust account event stream (`trust.deposit_received`, `trust.disbursement_made`, `trust.transfer_executed`) is a natural financial ledger. Three-way reconciliation reads directly from the event stream, and the immutable nature of events means trust transactions cannot be silently altered — exactly what ABA Model Rule 1.15 requires.

6. **UTBMS codes stored as strings in events, not as foreign keys** — event payloads contain the UTBMS code string (e.g., "L120") rather than a UUID reference to a UTBMS reference table. This makes events self-contained and prevents coupling between the event store and reference data tables. The projection builder can validate codes during projection building.

7. **Correlation IDs for request tracing** — every event includes a `correlation_id` that ties together all events produced by a single user action or API request. For example, creating a matter might produce `matter.opened`, `contact.role_assigned`, and `task.created` events — all sharing the same correlation ID. This enables debugging and audit trail reconstruction.

8. **Event versioning for schema evolution** — each event has an `event_version` field and a corresponding JSON Schema in the `event_type_registry`. When the event schema changes (e.g., adding a field to `matter.opened`), the version is incremented and an upcaster function transforms old events to the new shape during projection rebuilding.

9. **Operational tables for auth and tenancy are not event-sourced** — user accounts, tenant configuration, and authentication are managed as traditional CRUD tables. Event sourcing adds complexity that does not benefit these operational concerns, and user management has different lifecycle patterns (password resets, session management) that do not fit the event model.

10. **Projections include `last_event_id` and `version` for optimistic concurrency** — when updating a projection, the application checks that the current version matches the expected version, preventing race conditions between concurrent event processors. This is the standard optimistic concurrency control pattern for CQRS systems.
