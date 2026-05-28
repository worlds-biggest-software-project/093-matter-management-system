# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Matter Management System · Created: 2026-05-12

## Philosophy

This model keeps a relational backbone for core entities and relationships that are queried frequently and must maintain referential integrity (matters, contacts, invoices, trust accounts), but uses PostgreSQL JSONB columns extensively for variable, jurisdiction-specific, and tenant-customisable data. Instead of the EAV (Entity-Attribute-Value) anti-pattern or dozens of nullable columns, domain-specific fields live in structured JSONB columns with GIN indexes for fast containment queries.

This approach is inspired by platforms like Filevine, which offer "highly customisable workflows" with per-practice-area field configurations — a use case that maps naturally to JSONB. It is also well-suited to legal practice management because matter structures vary significantly by practice area (a personal injury matter has different fields than a trademark filing), by jurisdiction (Australian conveyancing vs. US federal litigation), and by firm preference (custom intake fields, firm-specific billing categories). Rather than anticipating every possible field in the relational schema, this model provides a stable relational core with JSONB extension points.

The hybrid model is best for rapid MVP development, multi-jurisdiction deployments where field requirements vary by region, and firms that need practice-area-specific matter configurations without database migrations.

**Best for:** Rapid development, multi-jurisdiction flexibility, and practice-area-specific customisation without schema migrations.

**Trade-offs:**
- Pro: No schema migrations for custom fields — add new fields by updating a JSONB schema definition, not altering tables
- Pro: Practice-area-specific fields (PI case: injury type, medical provider; IP case: patent number, filing date) live naturally in JSONB without polluting the core schema
- Pro: Jurisdiction-specific trust accounting rules, tax configurations, and compliance fields can vary per tenant without separate tables
- Pro: Faster MVP development — fewer tables to create and maintain
- Pro: GIN-indexed JSONB supports efficient containment queries (`@>` operator)
- Con: JSONB fields lack referential integrity — the database cannot enforce foreign keys within JSONB
- Con: Reporting on JSONB fields requires JSON path extraction, which is slower than querying indexed relational columns
- Con: Schema validation must be handled at the application level (JSON Schema) rather than by the database
- Con: Developers less familiar with JSONB query syntax may find it harder to write efficient queries
- Con: JSONB columns can become "junk drawers" without disciplined schema governance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SALI LMSS v2 | Matter classification stored in `matters.sali_classification` JSONB column containing multiple SALI IRIs (matter type, area of law, industry, jurisdiction) — enables multi-tag classification without junction tables |
| LEDES XML 2.2 | Invoice relational structure mirrors LEDES segments; LEDES-specific extension fields (regulatory statements, tiered taxes) stored in JSONB |
| UTBMS Codes | UTBMS codes stored as strings in time entries and line items; code set metadata in a lightweight reference table |
| ISO 3166 | Jurisdiction codes in `matters.jurisdiction` JSONB; trust accounting rules in `trust_accounts.jurisdiction_rules` JSONB |
| ISO 17442 (LEI) | Contact `details` JSONB includes `lei` field for organisations |
| ABA Model Rule 1.15 | Trust accounting uses relational tables for transactions with JSONB for jurisdiction-specific compliance metadata |
| JSON Schema Draft 2020-12 | Each JSONB column has a registered JSON Schema that the application validates against before writes |

---

## Schema Definition Registry

```sql
-- JSON Schema registry for validating JSONB columns
CREATE TABLE jsonb_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,             -- matter, contact, time_entry, invoice
    schema_name     VARCHAR(100) NOT NULL,            -- e.g., "matter_details_pi", "matter_details_ip"
    schema_version  INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,                   -- JSON Schema Draft 2020-12
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entity_type, schema_name, schema_version)
);

-- Example: JSON Schema for a Personal Injury matter's details
-- {
--   "$schema": "https://json-schema.org/draft/2020-12/schema",
--   "type": "object",
--   "properties": {
--     "injury_type": {"type": "string", "enum": ["auto", "slip_fall", "medical_malpractice", "product_liability", "workplace"]},
--     "injury_date": {"type": "string", "format": "date"},
--     "medical_providers": {"type": "array", "items": {"type": "object", "properties": {"name": {"type": "string"}, "specialty": {"type": "string"}, "total_bills": {"type": "number"}}}},
--     "insurance_company": {"type": "string"},
--     "policy_number": {"type": "string"},
--     "policy_limit": {"type": "number"},
--     "demand_amount": {"type": "number"},
--     "settlement_amount": {"type": "number"}
--   }
-- }
```

---

## Core Infrastructure

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Tenant-specific configuration
    billing_config  JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "default_currency": "USD",
    --   "rounding_increment": 0.1,
    --   "rounding_method": "up",
    --   "default_payment_terms_days": 30,
    --   "trust_accounting_enabled": true,
    --   "ledes_formats": ["xml_2_2", "1998b"]
    -- }
    branding        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"logo_url": "...", "primary_color": "#1a1a2e", "firm_name": "Smith & Associates LLP"}
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
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "bar_number": "CA-12345",
    --   "bar_state": "US-CA",
    --   "hourly_rate": 450.00,
    --   "timekeeper_classification": "partner",
    --   "practice_areas": ["litigation", "employment"],
    --   "phone": "+1-555-0100",
    --   "title": "Senior Partner"
    -- }
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_profile ON users USING GIN(profile);
```

---

## Contact Management

```sql
CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    contact_type    VARCHAR(20) NOT NULL,             -- individual, organisation
    display_name    VARCHAR(500) NOT NULL,            -- computed: "Last, First" or company_name
    -- Core searchable fields (relational for indexing)
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    company_name    VARCHAR(500),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    -- Everything else in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Individual example: {
    --   "prefix": "Dr.",
    --   "suffix": "Esq.",
    --   "middle_name": "Michael",
    --   "addresses": [
    --     {"type": "home", "line1": "123 Main St", "city": "San Francisco", "state": "CA", "postal": "94102", "country": "US"},
    --     {"type": "office", "line1": "456 Market St", "city": "San Francisco", "state": "CA", "postal": "94105", "country": "US"}
    --   ],
    --   "phones": [{"type": "mobile", "number": "+1-555-0100"}, {"type": "office", "number": "+1-555-0200"}],
    --   "social": {"linkedin": "https://linkedin.com/in/..."},
    --   "notes": "Prefers email communication"
    -- }
    -- Organisation example: {
    --   "lei": "549300EXAMPLE000001",
    --   "tax_id": "12-3456789",
    --   "industry": "Technology",
    --   "website": "https://acme.com",
    --   "addresses": [...],
    --   "officers": [{"name": "Jane Smith", "title": "CEO", "contact_id": "uuid-if-known"}],
    --   "subsidiaries": ["uuid-of-subsidiary-contact"]
    -- }
    relationships   JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"related_contact_id": "uuid", "relationship": "officer_of", "title": "CEO", "start_date": "2020-01-15"},
    --   {"related_contact_id": "uuid", "relationship": "subsidiary_of"}
    -- ]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_name ON contacts(tenant_id, last_name, first_name);
CREATE INDEX idx_contacts_company ON contacts(tenant_id, company_name);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_contacts_details ON contacts USING GIN(details);
CREATE INDEX idx_contacts_relationships ON contacts USING GIN(relationships);

-- Matter-contact roles (relational — frequently joined)
CREATE TABLE matter_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matter_id       UUID NOT NULL REFERENCES matters(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contacts(id),
    role            VARCHAR(50) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example for opposing counsel: {"firm_name": "Jones & Partners", "bar_number": "NY-67890"}
    -- Example for expert witness: {"specialty": "Forensic Accounting", "hourly_rate": 500}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(matter_id, contact_id, role)
);

CREATE INDEX idx_mc_matter ON matter_contacts(matter_id);
CREATE INDEX idx_mc_contact ON matter_contacts(contact_id);
```

---

## Matter Management

```sql
CREATE TABLE matters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_number   VARCHAR(50) NOT NULL,
    display_name    VARCHAR(500) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    pipeline_stage  VARCHAR(100),
    -- SALI Classification (JSONB — supports multi-tag classification)
    sali_classification JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "matter_type": {"iri": "https://lmss.sali.org/R3B5C7D9E1", "label": "Breach of Contract"},
    --   "area_of_law": {"iri": "https://lmss.sali.org/A2B4C6D8E0", "label": "Commercial Litigation"},
    --   "industry": {"iri": "https://lmss.sali.org/I1A2B3C4D5", "label": "Technology"},
    --   "sub_areas": [
    --     {"iri": "https://lmss.sali.org/S9A8B7C6D5", "label": "Contract Disputes"}
    --   ]
    -- }
    -- Jurisdiction (JSONB — multi-jurisdiction matters)
    jurisdiction    JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "primary": {"country": "US", "subdivision": "US-CA", "court": "Superior Court of California, County of Los Angeles"},
    --   "secondary": [{"country": "US", "subdivision": "US-NY"}],
    --   "case_number": "BC-2026-123456",
    --   "judge": "Hon. Maria Garcia"
    -- }
    -- Billing configuration
    billing         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "method": "hourly",
    --   "budget_amount": 50000.00,
    --   "rate_overrides": [
    --     {"user_id": "uuid", "rate": 500.00},
    --     {"timekeeper_classification": "associate", "rate": 350.00}
    --   ],
    --   "utbms_code_set": "litigation",
    --   "outside_counsel_guidelines_ref": "doc-uuid"
    -- }
    -- Dates
    dates           JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "opened": "2026-01-15",
    --   "closed": null,
    --   "statute_of_limitations": "2028-01-15",
    --   "trial_date": "2027-03-01",
    --   "discovery_cutoff": "2026-11-01",
    --   "mediation_date": "2026-09-15"
    -- }
    -- Practice-area-specific details (validated against jsonb_schemas)
    details         JSONB NOT NULL DEFAULT '{}',
    -- PI example: {
    --   "injury_type": "auto",
    --   "injury_date": "2025-08-20",
    --   "medical_providers": [{"name": "Dr. Smith", "specialty": "Orthopedics", "total_bills": 45000}],
    --   "insurance_company": "State Farm",
    --   "policy_number": "SF-2025-99887",
    --   "policy_limit": 100000,
    --   "demand_amount": 95000
    -- }
    -- IP example: {
    --   "patent_number": "US10,123,456",
    --   "filing_date": "2024-06-15",
    --   "patent_office": "USPTO",
    --   "inventors": ["Jane Doe", "John Smith"],
    --   "claims_count": 24,
    --   "prior_art_refs": ["US9,876,543", "US9,765,432"]
    -- }
    -- Assignment
    responsible_attorney_id UUID REFERENCES users(id),
    originating_attorney_id UUID REFERENCES users(id),
    practice_group  VARCHAR(100),
    -- Schema reference (which JSON Schema governs `details`)
    details_schema_id UUID REFERENCES jsonb_schemas(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
);

CREATE INDEX idx_matters_tenant ON matters(tenant_id);
CREATE INDEX idx_matters_status ON matters(tenant_id, status);
CREATE INDEX idx_matters_pipeline ON matters(tenant_id, pipeline_stage);
CREATE INDEX idx_matters_attorney ON matters(responsible_attorney_id);
CREATE INDEX idx_matters_sali ON matters USING GIN(sali_classification);
CREATE INDEX idx_matters_jurisdiction ON matters USING GIN(jurisdiction);
CREATE INDEX idx_matters_billing ON matters USING GIN(billing);
CREATE INDEX idx_matters_details ON matters USING GIN(details);
CREATE INDEX idx_matters_dates ON matters USING GIN(dates);

-- Matter templates with JSONB defaults
CREATE TABLE matter_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- Template defaults
    sali_classification JSONB NOT NULL DEFAULT '{}',
    billing_defaults    JSONB NOT NULL DEFAULT '{}',
    details_schema_id   UUID REFERENCES jsonb_schemas(id),
    default_tasks       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"title": "Draft Complaint", "due_offset_days": 7, "assignee_role": "associate"},
    --   {"title": "File with Court", "due_offset_days": 14, "assignee_role": "paralegal"},
    --   {"title": "Serve Defendant", "due_offset_days": 30, "assignee_role": "paralegal"}
    -- ]
    default_documents   JSONB NOT NULL DEFAULT '[]',
    pipeline_stages     JSONB NOT NULL DEFAULT '[]',
    -- Example: ["intake", "pleadings", "discovery", "depositions", "motions", "trial_prep", "trial", "post_trial", "closed"]
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
    -- Core relational fields (heavily queried)
    date            DATE NOT NULL,
    duration_hours  NUMERIC(6,2) NOT NULL,
    rate            NUMERIC(10,2) NOT NULL,
    total           NUMERIC(12,2) NOT NULL,
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    is_billed       BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- UTBMS codes as strings (no FK — code sets are stable reference data)
    utbms_task_code     VARCHAR(10),
    utbms_activity_code VARCHAR(10),
    -- Description
    description     TEXT NOT NULL,
    -- Flexible metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "duration_raw": 1.42,
    --   "rounding_method": "up_6min",
    --   "source": "timer",
    --   "source_ref": "calendar-event-id-123",
    --   "ai_confidence": 0.87,
    --   "approved_by": "uuid",
    --   "approved_at": "2026-05-12T10:30:00Z",
    --   "invoice_id": "uuid",
    --   "invoice_line_item_id": "uuid"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_te_matter ON time_entries(matter_id);
CREATE INDEX idx_te_user ON time_entries(user_id);
CREATE INDEX idx_te_date ON time_entries(tenant_id, date);
CREATE INDEX idx_te_status ON time_entries(tenant_id, status);
CREATE INDEX idx_te_unbilled ON time_entries(matter_id) WHERE is_billable AND NOT is_billed;
CREATE INDEX idx_te_metadata ON time_entries USING GIN(metadata);

-- Active timers
CREATE TABLE timers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    matter_id       UUID REFERENCES matters(id),
    description     TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    paused_at       TIMESTAMPTZ,
    elapsed_seconds INTEGER NOT NULL DEFAULT 0,
    is_running      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Billing & Invoicing

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_number  VARCHAR(50) NOT NULL,
    matter_id       UUID NOT NULL REFERENCES matters(id),
    client_id       UUID NOT NULL REFERENCES contacts(id),
    -- Core financial fields (relational)
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    total_fees      NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_expenses  NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_tax       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_discounts NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(12,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(12,2) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- LEDES and tax details in JSONB (varies by jurisdiction)
    ledes_details   JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "format": "xml_2_2",
    --   "exported_at": "2026-05-12T14:00:00Z",
    --   "law_firm_id": "LEDES-FIRM-001",
    --   "client_matter_id": "CLIENT-MATTER-2026-001",
    --   "regulatory_statements": [
    --     {"jurisdiction": "JP", "statement_type": "withholding_tax", "content": "..."}
    --   ],
    --   "tax_summary": {
    --     "taxes": [
    --       {"type": "VAT", "rate": 0.20, "taxable_amount": 10000.00, "tax_amount": 2000.00},
    --       {"type": "Withholding", "rate": 0.1021, "taxable_amount": 10000.00, "tax_amount": 1021.00}
    --     ]
    --   },
    --   "discounts": [
    --     {"type": "early_payment", "percentage": 0.02, "amount": 240.00}
    --   ]
    -- }
    -- Workflow metadata
    workflow        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "approved_by": "uuid",
    --   "approved_at": "2026-05-12T10:00:00Z",
    --   "sent_at": "2026-05-12T14:30:00Z",
    --   "sent_via": "email",
    --   "payment_reminders": [
    --     {"sent_at": "2026-06-12T09:00:00Z", "method": "email"}
    --   ]
    -- }
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_inv_matter ON invoices(matter_id);
CREATE INDEX idx_inv_client ON invoices(client_id);
CREATE INDEX idx_inv_status ON invoices(tenant_id, status);
CREATE INDEX idx_inv_date ON invoices(tenant_id, invoice_date);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    line_type       VARCHAR(10) NOT NULL,             -- fee, expense
    time_entry_id   UUID REFERENCES time_entries(id),
    description     TEXT NOT NULL,
    date            DATE NOT NULL,
    utbms_task_code VARCHAR(10),
    utbms_activity_code VARCHAR(10),
    utbms_expense_code VARCHAR(10),
    quantity        NUMERIC(8,2) NOT NULL DEFAULT 1,
    rate            NUMERIC(10,2) NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    timekeeper_id   UUID REFERENCES users(id),
    -- Line-item-level tax and adjustments in JSONB
    tax_items       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"type": "VAT", "rate": 0.20, "amount": 90.00, "jurisdiction": "GB"},
    --   {"type": "Withholding", "rate": 0.1021, "amount": 45.95, "jurisdiction": "JP"}
    -- ]
    adjustments     JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type": "discount", "description": "Volume discount", "amount": -50.00}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ili_invoice ON invoice_line_items(invoice_id);

-- Payments
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(12,2) NOT NULL,
    payment_method  VARCHAR(30) NOT NULL,
    payment_date    DATE NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "reference": "CHK-10042",
    --   "processor_ref": "pi_3ABC123DEF456",
    --   "processor": "stripe",
    --   "notes": "Final payment on invoice"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
```

---

## Trust Accounting

```sql
CREATE TABLE trust_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    account_name    VARCHAR(255) NOT NULL,
    bank_name       VARCHAR(255) NOT NULL,
    account_number_last4 CHAR(4) NOT NULL,
    is_iolta        BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    current_balance NUMERIC(14,2) NOT NULL DEFAULT 0,
    -- Jurisdiction-specific rules in JSONB
    jurisdiction_rules JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "country": "US",
    --   "state": "US-CA",
    --   "iolta_mandatory": true,
    --   "reconciliation_frequency": "monthly",
    --   "interest_reporting_required": true,
    --   "bar_association": "State Bar of California",
    --   "record_retention_years": 5,
    --   "commingling_rules": "No firm funds except to cover bank fees per Rule 1.15"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trust_client_ledgers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    contact_id      UUID NOT NULL REFERENCES contacts(id),
    matter_id       UUID REFERENCES matters(id),
    balance         NUMERIC(14,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(trust_account_id, contact_id, matter_id)
);

CREATE TABLE trust_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    client_ledger_id UUID NOT NULL REFERENCES trust_client_ledgers(id),
    transaction_type VARCHAR(20) NOT NULL,
    amount          NUMERIC(14,2) NOT NULL,
    running_balance NUMERIC(14,2) NOT NULL,
    description     TEXT NOT NULL,
    transaction_date DATE NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "reference": "Wire-20260512-001",
    --   "payment_id": "uuid-if-linked",
    --   "created_by": "uuid-of-user",
    --   "check_number": "10042",
    --   "cleared_date": "2026-05-14"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ttx_account ON trust_transactions(trust_account_id);
CREATE INDEX idx_ttx_ledger ON trust_transactions(client_ledger_id);
CREATE INDEX idx_ttx_date ON trust_transactions(transaction_date);

CREATE TABLE trust_reconciliations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    reconciliation_date DATE NOT NULL,
    bank_statement_balance NUMERIC(14,2) NOT NULL,
    book_balance    NUMERIC(14,2) NOT NULL,
    client_ledger_total NUMERIC(14,2) NOT NULL,
    outstanding_deposits NUMERIC(14,2) NOT NULL DEFAULT 0,
    outstanding_checks NUMERIC(14,2) NOT NULL DEFAULT 0,
    adjusted_bank_balance NUMERIC(14,2) NOT NULL,
    is_balanced     BOOLEAN NOT NULL,
    discrepancy     NUMERIC(14,2) NOT NULL DEFAULT 0,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "reconciled_by": "uuid",
    --   "notes": "All items reconciled; minor timing difference on check #10041",
    --   "outstanding_items": [
    --     {"type": "check", "number": "10041", "amount": 250.00, "date": "2026-04-29"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Documents, Tasks, Calendar, Communications

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    storage_provider VARCHAR(30) NOT NULL DEFAULT 'internal',
    title           VARCHAR(500),
    category        VARCHAR(100),
    version         INTEGER NOT NULL DEFAULT 1,
    parent_document_id UUID REFERENCES documents(id),
    is_current_version BOOLEAN NOT NULL DEFAULT true,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    -- Flexible metadata and tags in JSONB
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "tags": ["pleading", "motion", "filed"],
    --   "description": "Motion to Dismiss - Filed 2026-05-10",
    --   "court_filing_date": "2026-05-10",
    --   "docket_entry_number": 42,
    --   "ai_summary": "Motion arguing lack of personal jurisdiction under...",
    --   "ocr_status": "completed",
    --   "full_text_indexed": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_matter ON documents(matter_id);
CREATE INDEX idx_docs_tenant ON documents(tenant_id);
CREATE INDEX idx_docs_metadata ON documents USING GIN(metadata);

CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    priority        VARCHAR(10) NOT NULL DEFAULT 'medium',
    due_date        DATE,
    completed_at    TIMESTAMPTZ,
    assigned_to     UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"checklist": [{"item": "Draft motion", "done": true}, {"item": "Review with partner", "done": false}]}
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
    event_type      VARCHAR(30) NOT NULL,
    start_at        TIMESTAMPTZ NOT NULL,
    end_at          TIMESTAMPTZ,
    is_all_day      BOOLEAN NOT NULL DEFAULT false,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "location": "Courtroom 4B, Los Angeles Superior Court",
    --   "description": "Motion hearing before Hon. Garcia",
    --   "court_rule_ref": "CRC 3.1300",
    --   "external_calendar_id": "google-event-abc123",
    --   "external_provider": "google",
    --   "recurrence_rule": "FREQ=WEEKLY;UNTIL=20260601",
    --   "attendees": [{"contact_id": "uuid", "name": "Jane Smith", "role": "opposing_counsel"}]
    -- }
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_matter ON calendar_events(matter_id);
CREATE INDEX idx_cal_date ON calendar_events(tenant_id, start_at);

CREATE TABLE communications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    comm_type       VARCHAR(20) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    subject         VARCHAR(500),
    body            TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL,
    logged_by       UUID NOT NULL REFERENCES users(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "participants": [
    --     {"contact_id": "uuid", "name": "John Client", "role": "client"},
    --     {"contact_id": "uuid", "name": "Jane Attorney", "role": "attorney"}
    --   ],
    --   "external_ref": "msg-id-abc123@email.com",
    --   "attachments": [{"document_id": "uuid", "filename": "exhibit-a.pdf"}],
    --   "ai_summary": "Discussed settlement offer; client wants to counter at $85K"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comms_matter ON communications(matter_id);
CREATE INDEX idx_comms_date ON communications(tenant_id, occurred_at);
CREATE INDEX idx_comms_details ON communications USING GIN(details);
```

---

## Conflict Checking & Audit

```sql
CREATE TABLE conflict_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    search_query    TEXT NOT NULL,
    matter_id       UUID REFERENCES matters(id),
    results_count   INTEGER NOT NULL DEFAULT 0,
    outcome         VARCHAR(20),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example: {
    --   "reviewed_by": "uuid",
    --   "reviewed_at": "2026-05-12T10:00:00Z",
    --   "notes": "Cleared — no conflicts found",
    --   "results": [
    --     {"contact_id": "uuid", "match_type": "exact_name", "score": 95.0, "matter_id": "uuid", "role": "opposing_party", "is_conflict": false, "notes": "Different entity — same name"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(30) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    request_metadata JSONB,
    -- Example: {"ip": "192.168.1.1", "user_agent": "Mozilla/5.0...", "correlation_id": "uuid"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## JSONB Query Examples

```sql
-- Find all Personal Injury matters with policy limits above $50,000
SELECT id, display_name, details->>'injury_type' AS injury_type,
       (details->>'policy_limit')::NUMERIC AS policy_limit
FROM matters
WHERE tenant_id = 'uuid'
  AND details @> '{"injury_type": "auto"}'
  AND (details->>'policy_limit')::NUMERIC > 50000;

-- Find all matters classified under "Commercial Litigation" in SALI
SELECT id, display_name
FROM matters
WHERE tenant_id = 'uuid'
  AND sali_classification @> '{"area_of_law": {"label": "Commercial Litigation"}}';

-- Find contacts who are officers of a specific organisation
SELECT id, display_name
FROM contacts
WHERE tenant_id = 'uuid'
  AND relationships @> '[{"related_contact_id": "uuid-of-org", "relationship": "officer_of"}]';

-- Find all time entries generated by AI with confidence below 0.7
SELECT id, description, (metadata->>'ai_confidence')::NUMERIC AS confidence
FROM time_entries
WHERE tenant_id = 'uuid'
  AND metadata @> '{"source": "ai_suggestion"}'
  AND (metadata->>'ai_confidence')::NUMERIC < 0.7
ORDER BY date DESC;

-- Trust accounts in California with mandatory IOLTA
SELECT id, account_name
FROM trust_accounts
WHERE tenant_id = 'uuid'
  AND jurisdiction_rules @> '{"state": "US-CA", "iolta_mandatory": true}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 3 | tenants, users, jsonb_schemas |
| Contact Management | 2 | contacts, matter_contacts |
| Matter Management | 2 | matters, matter_templates |
| Time Tracking | 2 | time_entries, timers |
| Billing & Invoicing | 3 | invoices, invoice_line_items, payments |
| Trust Accounting | 4 | trust_accounts, trust_client_ledgers, trust_transactions, trust_reconciliations |
| Documents & Tasks | 2 | documents, tasks |
| Calendar & Communications | 2 | calendar_events, communications |
| Conflict & Audit | 2 | conflict_checks, audit_log |
| **Total** | **22** | Significantly fewer than normalized (39 tables) |

---

## Key Design Decisions

1. **JSONB for practice-area-specific fields rather than EAV** — the `matters.details` JSONB column replaces the entire `custom_field_definitions` / `custom_field_values` EAV pattern from Model 1. A PI matter stores `injury_type`, `medical_providers`, `policy_limit` in JSONB; an IP matter stores `patent_number`, `filing_date`, `claims_count`. The `jsonb_schemas` table provides application-level validation via JSON Schema, replacing the database-level constraints lost by using JSONB.

2. **SALI classification as JSONB rather than foreign keys** — the `matters.sali_classification` JSONB column stores SALI IRIs and labels inline rather than requiring three separate foreign-key columns pointing to SALI reference tables. This eliminates three joins from every matter query and supports multi-tag classification (a matter can have multiple sub-areas of law) without junction tables.

3. **Contact details and relationships in JSONB** — multiple addresses, phone numbers, social links, and organisation relationships are stored in the `contacts.details` and `contacts.relationships` JSONB columns. The most-queried fields (`first_name`, `last_name`, `company_name`, `email`, `phone`) remain as indexed relational columns for fast search, while variable data stays in JSONB.

4. **Invoice tax and adjustment items in line-item JSONB** — rather than separate `invoice_tax_items` and `invoice_adjustments` tables, each line item stores its taxes and adjustments in JSONB arrays. This mirrors the LEDES XML structure (where tax items are nested within fee/expense segments) and eliminates two tables.

5. **Trust accounting jurisdiction rules in JSONB** — IOLTA rules vary by US state and by country. Rather than a separate `trust_accounting_rules` table with per-jurisdiction rows, each trust account carries its applicable rules in the `jurisdiction_rules` JSONB column. This keeps the trust accounting queries simple (no joins to a rules table) and allows each tenant to customise their compliance metadata.

6. **22 tables vs. 39 in the normalised model** — the JSONB hybrid eliminates 17 tables by absorbing reference data, custom fields, tax items, and variable metadata into JSONB columns. This reduces join complexity and simplifies the ORM layer, at the cost of moving some validation to the application layer.

7. **GIN indexes on all JSONB columns** — every JSONB column has a GIN index to support efficient containment queries (`@>` operator). This is critical for performance: without GIN indexes, JSONB queries degrade to full table scans.

8. **Application-level JSON Schema validation** — every write to a JSONB column is validated against the registered JSON Schema before persisting. The `jsonb_schemas` table with versioning ensures that schema evolution is tracked and old data can be migrated progressively.

9. **Audit log with JSONB changes** — like Model 1, the audit log records changes as JSONB diffs. However, for JSONB columns, the diff can be a JSON Patch (RFC 6902) showing the structural changes within the JSONB column, providing finer granularity than a simple old/new comparison.

10. **Conflict check results embedded in JSONB** — rather than a separate `conflict_check_results` table, results are stored in the `conflict_checks.details` JSONB column. Conflict checks are infrequent write-once operations where the results are always read together with the check record, making a single-table design more efficient.
