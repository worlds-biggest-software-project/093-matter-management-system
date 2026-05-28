# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Matter Management System · Created: 2026-05-12

## Philosophy

This model follows classic third-normal-form (3NF) relational design: every domain concept gets its own table with well-defined foreign-key relationships, no data is duplicated, and referential integrity is enforced at the database level. Each table maps directly to a concept a lawyer or office manager would recognise — a matter, a contact, a time entry, an invoice, a trust ledger transaction.

This is the approach used by the majority of established legal practice management platforms (Clio, MyCase, Centerbase). It mirrors how the Clio API v4 exposes its data: distinct REST resources for matters, contacts, activities, bills, documents, calendar entries, and tasks, each linked by foreign keys. The LEDES billing standard and UTBMS code system both assume a flat, relational structure of invoice header -> line items -> task/activity codes, which maps naturally to normalized tables.

The normalised model is best for teams that value data integrity, need complex cross-entity reporting (e.g., "total fees by UTBMS phase code across all matters for a given client this quarter"), and want a schema that maps directly to the LEDES and SALI standards without any impedance mismatch.

**Best for:** Teams prioritising data integrity, standards compliance, and familiar SQL query patterns over schema flexibility.

**Trade-offs:**
- Pro: Maximum referential integrity — the database prevents orphaned records, broken relationships, and inconsistent states
- Pro: Maps directly to LEDES invoice structure, UTBMS code sets, and SALI LMSS ontology without translation
- Pro: Well-understood by developers; excellent tooling support (ORMs, migration tools, reporting engines)
- Pro: Complex cross-entity queries (joins across matters, contacts, time entries, invoices) are straightforward
- Con: Schema changes require migrations — adding a jurisdiction-specific field means altering a table or adding a new one
- Con: High table count increases join complexity for some queries
- Con: Custom fields require either an EAV pattern (which is slow) or frequent schema migrations
- Con: Audit history requires a separate mechanism (trigger-based audit tables or application-level logging)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SALI LMSS v2 | `sali_matter_types`, `sali_areas_of_law`, `sali_industries` reference tables hold the SALI ontology tags; matters link via foreign keys to classify by area of law, matter type, and industry |
| LEDES XML 2.2 | Invoice tables (`invoices`, `invoice_line_items`) mirror the LEDES XML segment hierarchy: @INVOICE → @MATTER → @FEE / @EXPENSE → @TAX_ITEM |
| UTBMS Codes | `utbms_phases`, `utbms_tasks`, `utbms_activities`, `utbms_expenses` reference tables hold the five code sets (Litigation, IP, Counseling, Project, Bankruptcy) |
| ISO 3166 | `jurisdictions` table uses ISO 3166-1 (country) and 3166-2 (subdivision) codes for court jurisdictions and trust accounting rule sets |
| ISO 17442 (LEI) | `organisations.lei` column stores Legal Entity Identifiers for corporate clients |
| ABA Model Rule 1.15 | Trust accounting tables (`trust_accounts`, `trust_transactions`, `trust_reconciliations`) implement three-way reconciliation |
| OAuth 2.0 / OIDC | `oauth_clients`, `oauth_tokens` tables support API authentication per RFC 6749 and OpenID Connect |
| PCI DSS v4.0.1 | Payment tables store only tokenised references; actual card data is never stored (delegated to Stripe/LawPay) |

---

## Core Infrastructure Tables

### Tenancy & Authentication

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,       -- subdomain identifier
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free', -- free, professional, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',         -- tenant-level config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),                        -- null if SSO-only
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'user', -- admin, partner, associate, paralegal, user
    bar_number      VARCHAR(50),                         -- attorney bar admission number
    hourly_rate     NUMERIC(10,2),                       -- default billing rate
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

### Row-Level Security

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant')::UUID);

-- Apply similar policies to all tenant-scoped tables below
```

---

## SALI / UTBMS Reference Data Tables

```sql
-- SALI LMSS Ontology Reference Tables (loaded from SALI OWL export)
CREATE TABLE sali_matter_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sali_iri        VARCHAR(500) NOT NULL UNIQUE,     -- SALI IRI identifier
    code            VARCHAR(50) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES sali_matter_types(id),
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE sali_areas_of_law (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sali_iri        VARCHAR(500) NOT NULL UNIQUE,
    code            VARCHAR(50) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES sali_areas_of_law(id),
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE sali_industries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sali_iri        VARCHAR(500) NOT NULL UNIQUE,
    code            VARCHAR(50) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES sali_industries(id),
    is_active       BOOLEAN NOT NULL DEFAULT true
);

-- UTBMS Code Sets
CREATE TABLE utbms_code_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,     -- Litigation, IP, Counseling, Project, Bankruptcy
    letter_prefix   CHAR(1) NOT NULL UNIQUE           -- L, P, C, T, B
);

CREATE TABLE utbms_phases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_set_id     UUID NOT NULL REFERENCES utbms_code_sets(id),
    code            VARCHAR(10) NOT NULL UNIQUE,      -- e.g., "L100"
    description     VARCHAR(255) NOT NULL
);

CREATE TABLE utbms_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phase_id        UUID NOT NULL REFERENCES utbms_phases(id),
    code            VARCHAR(10) NOT NULL UNIQUE,      -- e.g., "L110"
    description     VARCHAR(255) NOT NULL
);

CREATE TABLE utbms_activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,      -- e.g., "A101"
    description     VARCHAR(255) NOT NULL
);

CREATE TABLE utbms_expenses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,      -- e.g., "E101" or "X101"
    description     VARCHAR(255) NOT NULL,
    is_revised      BOOLEAN NOT NULL DEFAULT false     -- true for LOC 2013 revised (X-prefix)
);

-- Jurisdictions (ISO 3166)
CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    CHAR(2) NOT NULL,                 -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                      -- ISO 3166-2 (e.g., "US-CA")
    name            VARCHAR(255) NOT NULL,
    court_system    VARCHAR(100),                     -- e.g., "US Federal", "US State", "England & Wales"
    trust_accounting_rules JSONB,                     -- jurisdiction-specific IOLTA rules
    UNIQUE(country_code, subdivision_code)
);

CREATE INDEX idx_jurisdictions_country ON jurisdictions(country_code);
```

---

## Contact Management

```sql
CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    contact_type    VARCHAR(20) NOT NULL,             -- individual, organisation
    -- Individual fields
    first_name      VARCHAR(255),
    middle_name     VARCHAR(255),
    last_name       VARCHAR(255),
    prefix          VARCHAR(20),                      -- Mr., Ms., Dr., Hon.
    suffix          VARCHAR(20),                      -- Jr., III, Esq.
    -- Organisation fields
    company_name    VARCHAR(500),
    lei             VARCHAR(20),                      -- ISO 17442 Legal Entity Identifier
    tax_id          VARCHAR(50),                      -- EIN / tax registration
    -- Common fields
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),                          -- ISO 3166-1
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_name ON contacts(tenant_id, last_name, first_name);
CREATE INDEX idx_contacts_company ON contacts(tenant_id, company_name);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);

-- Contact roles in a matter (client, opposing party, opposing counsel, judge, expert witness, etc.)
CREATE TABLE matter_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contacts(id),
    role            VARCHAR(50) NOT NULL,             -- client, opposing_party, opposing_counsel, judge, witness, expert_witness, co_counsel
    is_primary      BOOLEAN NOT NULL DEFAULT false,   -- primary client contact
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, contact_id, role)
);

CREATE INDEX idx_matter_contacts_matter ON matter_contacts(matter_id);
CREATE INDEX idx_matter_contacts_contact ON matter_contacts(contact_id);

-- Organisation-individual relationships (officer, director, employee, etc.)
CREATE TABLE contact_relationships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    from_contact_id UUID NOT NULL REFERENCES contacts(id),
    to_contact_id   UUID NOT NULL REFERENCES contacts(id),
    relationship    VARCHAR(50) NOT NULL,             -- officer_of, director_of, employee_of, subsidiary_of, parent_of
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_rels_from ON contact_relationships(from_contact_id);
CREATE INDEX idx_contact_rels_to ON contact_relationships(to_contact_id);
```

---

## Matter Management

```sql
CREATE TABLE matters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_number   VARCHAR(50) NOT NULL,             -- firm-generated matter number
    display_name    VARCHAR(500) NOT NULL,            -- e.g., "Smith v. Jones" or "Acme Corp Trademark Filing"
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open', -- open, pending, closed, archived
    -- SALI Classification
    sali_matter_type_id   UUID REFERENCES sali_matter_types(id),
    sali_area_of_law_id   UUID REFERENCES sali_areas_of_law(id),
    sali_industry_id      UUID REFERENCES sali_industries(id),
    -- UTBMS
    utbms_code_set_id     UUID REFERENCES utbms_code_sets(id),  -- which UTBMS code set applies
    -- Jurisdiction
    jurisdiction_id       UUID REFERENCES jurisdictions(id),
    court_name            VARCHAR(255),
    case_number           VARCHAR(100),               -- court-assigned case number
    -- Dates
    date_opened           DATE NOT NULL DEFAULT CURRENT_DATE,
    date_closed           DATE,
    statute_of_limitations DATE,
    -- Billing
    billing_method        VARCHAR(30) NOT NULL DEFAULT 'hourly', -- hourly, flat_fee, contingency, blended, pro_bono
    billing_rate_override NUMERIC(10,2),              -- override default attorney rates
    budget_amount         NUMERIC(12,2),
    -- Assignment
    responsible_attorney_id UUID REFERENCES users(id),
    originating_attorney_id UUID REFERENCES users(id),
    practice_group        VARCHAR(100),
    -- Pipeline stage
    pipeline_stage        VARCHAR(100),               -- for kanban view: intake, active, discovery, trial, settlement, closed
    -- Template reference
    template_id           UUID REFERENCES matter_templates(id),
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
);

CREATE INDEX idx_matters_tenant ON matters(tenant_id);
CREATE INDEX idx_matters_status ON matters(tenant_id, status);
CREATE INDEX idx_matters_responsible ON matters(responsible_attorney_id);
CREATE INDEX idx_matters_sali_type ON matters(sali_matter_type_id);
CREATE INDEX idx_matters_pipeline ON matters(tenant_id, pipeline_stage);

-- Matter custom fields (EAV pattern for tenant-specific fields)
CREATE TABLE custom_field_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    entity_type     VARCHAR(50) NOT NULL,             -- matter, contact
    field_name      VARCHAR(100) NOT NULL,
    field_type      VARCHAR(30) NOT NULL,             -- text, number, date, boolean, select, multi_select
    options         JSONB,                            -- for select/multi_select types
    is_required     BOOLEAN NOT NULL DEFAULT false,
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, entity_type, field_name)
);

CREATE TABLE custom_field_values (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_definition_id UUID NOT NULL REFERENCES custom_field_definitions(id),
    entity_id       UUID NOT NULL,                    -- matter or contact ID
    value_text      TEXT,
    value_number    NUMERIC(18,4),
    value_date      DATE,
    value_boolean   BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(field_definition_id, entity_id)
);

CREATE INDEX idx_custom_values_entity ON custom_field_values(entity_id);

-- Matter templates
CREATE TABLE matter_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    sali_matter_type_id UUID REFERENCES sali_matter_types(id),
    utbms_code_set_id   UUID REFERENCES utbms_code_sets(id),
    default_tasks   JSONB NOT NULL DEFAULT '[]',      -- [{name, due_offset_days, assignee_role}]
    default_documents JSONB NOT NULL DEFAULT '[]',    -- [{name, template_url}]
    default_pipeline_stages JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Time Tracking

```sql
CREATE TABLE time_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    -- Time
    date            DATE NOT NULL,
    duration_hours  NUMERIC(6,2) NOT NULL,            -- after rounding
    duration_raw    NUMERIC(6,2),                     -- before rounding (for audit)
    -- Description
    description     TEXT NOT NULL,
    -- UTBMS Codes
    utbms_task_id   UUID REFERENCES utbms_tasks(id),
    utbms_activity_id UUID REFERENCES utbms_activities(id),
    -- Billing
    rate            NUMERIC(10,2) NOT NULL,           -- hourly rate applied
    total           NUMERIC(12,2) NOT NULL,           -- rate * duration_hours
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    is_billed       BOOLEAN NOT NULL DEFAULT false,
    invoice_line_item_id UUID,                        -- set when billed
    -- Source
    source          VARCHAR(30) NOT NULL DEFAULT 'manual', -- manual, timer, ai_suggestion, calendar, email
    source_ref      VARCHAR(255),                     -- reference to source event (calendar ID, email ID)
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, approved, billed, written_off
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_time_entries_matter ON time_entries(matter_id);
CREATE INDEX idx_time_entries_user ON time_entries(user_id);
CREATE INDEX idx_time_entries_date ON time_entries(tenant_id, date);
CREATE INDEX idx_time_entries_status ON time_entries(tenant_id, status);
CREATE INDEX idx_time_entries_unbilled ON time_entries(matter_id) WHERE is_billable = true AND is_billed = false;

-- Active timers (in-progress time capture)
CREATE TABLE timers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    matter_id       UUID REFERENCES matters(id),
    description     TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    paused_at       TIMESTAMPTZ,
    elapsed_seconds INTEGER NOT NULL DEFAULT 0,       -- accumulated before current start
    is_running      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timers_user ON timers(user_id, is_running);
```

---

## Billing & Invoicing (LEDES-Aligned)

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_number  VARCHAR(50) NOT NULL,
    matter_id       UUID NOT NULL REFERENCES matters(id),
    client_id       UUID NOT NULL REFERENCES contacts(id),
    -- LEDES Invoice Header fields
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',   -- ISO 4217
    -- Amounts
    total_fees      NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_expenses  NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_tax       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_discounts NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL DEFAULT 0, -- fees + expenses + tax - discounts
    amount_paid     NUMERIC(12,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(12,2) NOT NULL DEFAULT 0,
    -- Status
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, pending_review, approved, sent, paid, partial, overdue, void
    -- LEDES export
    ledes_format    VARCHAR(20),                      -- xml_2_2, xml_2_1, 1998b
    ledes_exported_at TIMESTAMPTZ,
    -- Approval
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_invoices_matter ON invoices(matter_id);
CREATE INDEX idx_invoices_client ON invoices(client_id);
CREATE INDEX idx_invoices_status ON invoices(tenant_id, status);
CREATE INDEX idx_invoices_date ON invoices(tenant_id, invoice_date);

-- LEDES @FEE and @EXPENSE segments
CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    line_type       VARCHAR(10) NOT NULL,             -- fee, expense
    -- For fees: link to time entry
    time_entry_id   UUID REFERENCES time_entries(id),
    -- Description
    description     TEXT NOT NULL,
    date            DATE NOT NULL,
    -- UTBMS codes
    utbms_task_id   UUID REFERENCES utbms_tasks(id),
    utbms_activity_id UUID REFERENCES utbms_activities(id),
    utbms_expense_id UUID REFERENCES utbms_expenses(id),
    -- Amounts
    quantity        NUMERIC(8,2) NOT NULL DEFAULT 1,  -- hours for fees, units for expenses
    rate            NUMERIC(10,2) NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    -- Timekeeper (LEDES requirement)
    timekeeper_id   UUID REFERENCES users(id),
    timekeeper_classification VARCHAR(50),            -- partner, associate, paralegal, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

-- Tax items per line (LEDES @TAX_ITEM_FEE / @TAX_ITEM_EXPENSE)
CREATE TABLE invoice_tax_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    line_item_id    UUID NOT NULL REFERENCES invoice_line_items(id) ON DELETE CASCADE,
    tax_type        VARCHAR(100) NOT NULL,            -- e.g., "VAT", "GST", "Withholding"
    tax_rate        NUMERIC(6,4) NOT NULL,            -- e.g., 0.2000 for 20%
    tax_amount      NUMERIC(12,2) NOT NULL,
    jurisdiction_id UUID REFERENCES jurisdictions(id)
);

-- Payments received
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(12,2) NOT NULL,
    payment_method  VARCHAR(30) NOT NULL,             -- check, wire, ach, credit_card, trust_transfer
    payment_date    DATE NOT NULL,
    reference       VARCHAR(255),                     -- check number, transaction ID
    -- For credit card: tokenised reference only (PCI DSS compliance)
    payment_processor_ref VARCHAR(255),               -- Stripe/LawPay payment intent ID
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

---

## Trust Accounting (IOLTA / ABA Rule 1.15)

```sql
CREATE TABLE trust_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    account_name    VARCHAR(255) NOT NULL,
    bank_name       VARCHAR(255) NOT NULL,
    account_number_last4 CHAR(4) NOT NULL,            -- last 4 digits only (security)
    routing_number  VARCHAR(20),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    is_iolta        BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    current_balance NUMERIC(14,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Individual client trust ledger (one per client per trust account)
CREATE TABLE trust_client_ledgers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    contact_id      UUID NOT NULL REFERENCES contacts(id),   -- the client
    matter_id       UUID REFERENCES matters(id),             -- optional matter-level sub-ledger
    balance         NUMERIC(14,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(trust_account_id, contact_id, matter_id)
);

-- Trust transactions (deposits, disbursements, transfers)
CREATE TABLE trust_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    client_ledger_id UUID NOT NULL REFERENCES trust_client_ledgers(id),
    transaction_type VARCHAR(20) NOT NULL,            -- deposit, disbursement, transfer_in, transfer_out, interest, bank_fee
    amount          NUMERIC(14,2) NOT NULL,           -- positive for deposits, negative for disbursements
    running_balance NUMERIC(14,2) NOT NULL,
    description     TEXT NOT NULL,
    reference       VARCHAR(255),                     -- check number, wire reference
    transaction_date DATE NOT NULL,
    -- Link to invoice payment if disbursed to pay firm invoice
    payment_id      UUID REFERENCES payments(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trust_tx_account ON trust_transactions(trust_account_id);
CREATE INDEX idx_trust_tx_ledger ON trust_transactions(client_ledger_id);
CREATE INDEX idx_trust_tx_date ON trust_transactions(transaction_date);

-- Three-way reconciliation records
CREATE TABLE trust_reconciliations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    reconciliation_date DATE NOT NULL,
    -- Three-way balances
    bank_statement_balance  NUMERIC(14,2) NOT NULL,
    book_balance            NUMERIC(14,2) NOT NULL,   -- trust account ledger balance
    client_ledger_total     NUMERIC(14,2) NOT NULL,   -- sum of all client ledger balances
    -- Reconciling items
    outstanding_deposits    NUMERIC(14,2) NOT NULL DEFAULT 0,
    outstanding_checks      NUMERIC(14,2) NOT NULL DEFAULT 0,
    adjusted_bank_balance   NUMERIC(14,2) NOT NULL,
    -- Result
    is_balanced     BOOLEAN NOT NULL,                 -- true if all three match
    discrepancy     NUMERIC(14,2) NOT NULL DEFAULT 0,
    notes           TEXT,
    reconciled_by   UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_trust_recon_account ON trust_reconciliations(trust_account_id, reconciliation_date);
```

---

## Document Management

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    -- File info
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100),                     -- MIME type
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,           -- S3/GCS path or local path
    storage_provider VARCHAR(30) NOT NULL DEFAULT 'internal', -- internal, s3, gcs, imanage
    -- Metadata
    title           VARCHAR(500),
    description     TEXT,
    category        VARCHAR(100),                     -- pleading, motion, correspondence, contract, evidence, engagement_letter
    -- Version control
    version         INTEGER NOT NULL DEFAULT 1,
    parent_document_id UUID REFERENCES documents(id), -- previous version
    is_current_version BOOLEAN NOT NULL DEFAULT true,
    -- Upload info
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_matter ON documents(matter_id);
CREATE INDEX idx_documents_tenant ON documents(tenant_id);
CREATE INDEX idx_documents_category ON documents(tenant_id, category);

-- Document tags for search and classification
CREATE TABLE document_tags (
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    tag             VARCHAR(100) NOT NULL,
    PRIMARY KEY (document_id, tag)
);

CREATE INDEX idx_document_tags_tag ON document_tags(tag);
```

---

## Tasks & Calendar

```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, in_progress, completed, cancelled
    priority        VARCHAR(10) NOT NULL DEFAULT 'medium',  -- low, medium, high, urgent
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    assigned_to     UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_matter ON tasks(matter_id);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to, status);
CREATE INDEX idx_tasks_due ON tasks(tenant_id, due_date) WHERE status NOT IN ('completed', 'cancelled');

CREATE TABLE calendar_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    event_type      VARCHAR(30) NOT NULL,             -- hearing, deposition, meeting, deadline, filing_deadline, statute_expiry
    location        VARCHAR(500),
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ,
    is_all_day      BOOLEAN NOT NULL DEFAULT false,
    -- Court deadline rule reference
    court_rule_ref  VARCHAR(255),                     -- reference to jurisdiction rule set
    -- External sync
    external_calendar_id VARCHAR(255),                -- Google Calendar / Outlook event ID
    external_provider    VARCHAR(30),                 -- google, microsoft
    -- Recurrence
    recurrence_rule VARCHAR(255),                     -- iCal RRULE format
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calendar_matter ON calendar_events(matter_id);
CREATE INDEX idx_calendar_date ON calendar_events(tenant_id, start_at);
CREATE INDEX idx_calendar_type ON calendar_events(tenant_id, event_type);
```

---

## Client Communication Portal

```sql
CREATE TABLE portal_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    sender_user_id  UUID REFERENCES users(id),        -- null if sent by client
    sender_contact_id UUID REFERENCES contacts(id),   -- null if sent by firm user
    subject         VARCHAR(500),
    body            TEXT NOT NULL,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_portal_messages_matter ON portal_messages(matter_id);

-- Communication log (emails, calls, meetings linked to matters)
CREATE TABLE communications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    comm_type       VARCHAR(20) NOT NULL,             -- email, phone, meeting, letter, portal_message
    direction       VARCHAR(10) NOT NULL,             -- inbound, outbound
    subject         VARCHAR(500),
    body            TEXT,
    participants    JSONB NOT NULL DEFAULT '[]',      -- [{contact_id, role}]
    occurred_at     TIMESTAMPTZ NOT NULL,
    external_ref    VARCHAR(255),                     -- email message ID, calendar event ID
    logged_by       UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_communications_matter ON communications(matter_id);
CREATE INDEX idx_communications_date ON communications(tenant_id, occurred_at);
```

---

## Conflict Checking

```sql
-- Conflict check searches (log of every check performed)
CREATE TABLE conflict_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    search_query    TEXT NOT NULL,                    -- names searched
    matter_id       UUID REFERENCES matters(id),     -- matter being opened (if applicable)
    results_count   INTEGER NOT NULL DEFAULT 0,
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    outcome         VARCHAR(20),                     -- clear, potential_conflict, conflict_found, waiver_obtained
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conflict_check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conflict_check_id UUID NOT NULL REFERENCES conflict_checks(id) ON DELETE CASCADE,
    matched_contact_id UUID NOT NULL REFERENCES contacts(id),
    matched_matter_id  UUID REFERENCES matters(id),
    match_type      VARCHAR(30) NOT NULL,            -- exact_name, alias, related_entity, adverse_party
    match_score     NUMERIC(5,2),                    -- similarity score (0-100)
    role_in_match   VARCHAR(50),                     -- what role this contact plays in the matched matter
    is_conflict     BOOLEAN,                         -- attorney determination
    notes           TEXT
);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(30) NOT NULL,             -- create, update, delete, view, export, login, logout
    entity_type     VARCHAR(50) NOT NULL,             -- matter, contact, time_entry, invoice, document, trust_transaction
    entity_id       UUID NOT NULL,
    changes         JSONB,                            -- {field: {old: x, new: y}}
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);

-- Partition by month for performance
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 5 | tenants, users, roles, user_roles, audit_log |
| Reference Data (SALI/UTBMS/Jurisdictions) | 9 | SALI matter types, areas of law, industries; UTBMS code sets, phases, tasks, activities, expenses; jurisdictions |
| Contact Management | 3 | contacts, matter_contacts, contact_relationships |
| Matter Management | 4 | matters, matter_templates, custom_field_definitions, custom_field_values |
| Time Tracking | 2 | time_entries, timers |
| Billing & Invoicing | 4 | invoices, invoice_line_items, invoice_tax_items, payments |
| Trust Accounting | 4 | trust_accounts, trust_client_ledgers, trust_transactions, trust_reconciliations |
| Document Management | 2 | documents, document_tags |
| Tasks & Calendar | 2 | tasks, calendar_events |
| Communication | 2 | portal_messages, communications |
| Conflict Checking | 2 | conflict_checks, conflict_check_results |
| **Total** | **39** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation, avoids sequence contention in multi-tenant deployments, and supports offline-first mobile clients that can generate IDs before syncing.

2. **Separate SALI reference tables rather than inline enums** — the SALI LMSS ontology contains 18,000+ tags that update independently of the application schema. Storing them as reference tables loaded from the SALI OWL export allows updates without schema migrations.

3. **UTBMS codes as relational reference tables** — each time entry and invoice line item links to a specific UTBMS task, activity, or expense code via foreign key, matching the LEDES invoice structure and enabling aggregate billing analytics by phase and task code.

4. **EAV pattern for custom fields** — the `custom_field_definitions` / `custom_field_values` tables allow tenants to add matter-specific and contact-specific fields without schema changes. This adds query complexity but avoids the need for per-tenant schema migrations. The trade-off is that complex filtering on custom fields requires joins that scale linearly with the number of custom fields.

5. **Explicit trust accounting tables with three-way reconciliation** — rather than treating trust as part of general billing, trust accounts, client ledgers, and transactions are modelled as separate entities with a dedicated reconciliation record, directly implementing ABA Model Rule 1.15 requirements.

6. **LEDES-aligned invoice structure** — the `invoices` → `invoice_line_items` → `invoice_tax_items` hierarchy mirrors the LEDES XML segment structure (@INVOICE → @MATTER → @FEE/@EXPENSE → @TAX_ITEM), making LEDES export a straightforward serialisation rather than a complex transformation.

7. **Row-level security for tenant isolation** — PostgreSQL RLS policies on every tenant-scoped table enforce data isolation at the database level, providing defence-in-depth beyond application-level filtering. This is the standard pattern for shared-table multi-tenant SaaS (used by Supabase, Nile, and similar platforms).

8. **Trigger-based audit log rather than event sourcing** — an `audit_log` table captures changes as JSONB diffs. This is simpler than full event sourcing but means the audit trail is a secondary record, not the source of truth. Historical state reconstruction requires replaying diffs rather than querying the event store directly.

9. **Contact relationship table for conflict checking** — the `contact_relationships` table models corporate hierarchies, officer/director relationships, and affiliations that are essential for conflict-of-interest analysis. Combined with `matter_contacts` (which tracks party roles per matter), this enables graph-style traversal for conflict checks using recursive CTEs.

10. **Time entry source tracking** — the `source` column on `time_entries` distinguishes manual entries from timer-generated, AI-suggested, and calendar/email-derived entries. This is essential for the AI passive time capture feature and for attorney review workflows where AI-generated entries require explicit approval.
