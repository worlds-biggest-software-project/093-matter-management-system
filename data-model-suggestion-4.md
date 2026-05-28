# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Matter Management System · Created: 2026-05-12

## Philosophy

This model combines a conventional relational backbone for operational CRUD (matters, invoices, time entries, trust accounting) with a property graph layer for relationship-heavy queries: conflict-of-interest checking, corporate hierarchy traversal, matter-contact network analysis, and party relationship mapping. The graph layer is implemented using PostgreSQL tables (`graph_nodes` and `graph_edges`) with recursive CTEs, avoiding the need for a separate graph database while enabling graph-style traversal queries.

Legal practice management has an inherently graph-shaped relationship problem. A single conflict-of-interest check requires traversing: the client's corporate parent, its subsidiaries, its officers and directors, their other directorships, those companies' law firms, those firms' other matters, and the adverse parties in those matters. This is a multi-hop graph traversal that is awkward to express with traditional relational joins but natural with a graph model. Similarly, ABA Model Rule 1.7 (Conflict of Interest: Current Clients) requires identifying "directly adverse" and "materially limited" relationships across a network of entities — a graph problem by definition.

This approach is inspired by platforms like Litify (built on Salesforce, which has graph-like relationship objects) and by knowledge graph architectures used in compliance and anti-money-laundering systems where entity relationship mapping is the core value proposition.

**Best for:** Firms prioritising conflict-of-interest analysis, corporate client relationship mapping, and complex party networks across large matter portfolios.

**Trade-offs:**
- Pro: Conflict-of-interest checks become graph traversals rather than complex multi-table joins — faster development and more comprehensive results
- Pro: Corporate hierarchy navigation (parent companies, subsidiaries, officers, directors) is a first-class operation
- Pro: "Six degrees of separation" queries between any two entities are trivial — useful for detecting non-obvious conflicts
- Pro: The graph layer can be extended for AI-powered relationship discovery and risk analysis
- Pro: Matter network visualization (who is connected to whom across all matters) is a natural output
- Con: The graph layer adds conceptual complexity — developers must understand both relational and graph patterns
- Con: Recursive CTEs in PostgreSQL can be slow for deep traversals (> 10 hops) on large graphs
- Con: Graph data must be kept in sync with the relational tables — dual-write consistency management
- Con: More complex than a pure relational model for simple CRUD operations
- Con: Fewer developers are experienced with graph patterns than with traditional relational design

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SALI LMSS v2 | SALI ontology tags modelled as graph nodes with hierarchical edges, enabling ontology traversal and multi-tag classification |
| LEDES XML 2.2 | Invoice tables follow standard relational design mirroring LEDES segment hierarchy |
| UTBMS Codes | UTBMS codes stored relationally; graph edges link matters to their applicable code sets |
| ABA Model Rule 1.7 | The graph layer directly implements the conflict-of-interest analysis required by Rule 1.7 — graph traversal identifies "directly adverse" and "materially limited" relationships |
| ISO 17442 (LEI) | LEI stored as a graph node property for organisations; enables entity resolution across matters |
| ISO 3166 | Jurisdictions modelled as graph nodes; edges connect matters and trust accounts to their jurisdictions |
| ABA Model Rule 1.15 | Trust accounting uses relational tables; graph edges connect trust accounts to clients and matters for relationship visualization |

---

## Graph Layer

```sql
-- Universal graph node table — every entity that participates in relationships
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: person, organisation, matter, court, jurisdiction,
    --             sali_tag, utbms_code, document, user
    -- Link to relational entity
    entity_id       UUID,                             -- FK to contacts.id, matters.id, users.id, etc.
    entity_table    VARCHAR(50),                      -- 'contacts', 'matters', 'users', etc.
    -- Display
    label           VARCHAR(500) NOT NULL,            -- display name for graph visualization
    -- Properties (flexible key-value)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Person example:  {"first_name": "John", "last_name": "Smith", "email": "john@example.com", "bar_number": "CA-12345"}
    -- Org example:     {"company_name": "Acme Corp", "lei": "549300EXAMPLE01", "industry": "Technology"}
    -- Matter example:  {"matter_number": "2026-00142", "status": "open", "billing_method": "hourly"}
    -- Court example:   {"name": "Superior Court of California", "jurisdiction": "US-CA"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_gn_type ON graph_nodes(tenant_id, node_type);
CREATE INDEX idx_gn_entity ON graph_nodes(entity_table, entity_id);
CREATE INDEX idx_gn_label ON graph_nodes(tenant_id, label);
CREATE INDEX idx_gn_properties ON graph_nodes USING GIN(properties);

-- Directed edges between graph nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    from_node_id    UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    to_node_id      UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types for contacts/parties:
    --   officer_of, director_of, employee_of, subsidiary_of, parent_of,
    --   shareholder_of, counsel_for, adverse_to, related_to, affiliated_with
    -- Edge types for matter-contact roles:
    --   client_of, opposing_party_in, opposing_counsel_in, judge_in,
    --   witness_in, expert_in, co_counsel_in
    -- Edge types for hierarchies:
    --   child_of (SALI ontology, org hierarchy)
    -- Edge types for documents:
    --   attached_to, version_of
    -- Properties (flexible metadata per edge)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example: {"start_date": "2020-01-15", "title": "CEO", "is_primary": true}
    -- Example: {"ownership_percentage": 51.0}
    -- Temporal validity
    valid_from      DATE,
    valid_to        DATE,                             -- null = currently active
    weight          NUMERIC(5,2) DEFAULT 1.0,         -- for weighted traversal (e.g., ownership percentage)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_from ON graph_edges(from_node_id);
CREATE INDEX idx_ge_to ON graph_edges(to_node_id);
CREATE INDEX idx_ge_type ON graph_edges(edge_type);
CREATE INDEX idx_ge_tenant ON graph_edges(tenant_id);
CREATE INDEX idx_ge_from_type ON graph_edges(from_node_id, edge_type);
CREATE INDEX idx_ge_to_type ON graph_edges(to_node_id, edge_type);
CREATE INDEX idx_ge_temporal ON graph_edges(valid_from, valid_to) WHERE valid_to IS NULL;
```

---

## Relational Core (Operational CRUD)

### Tenancy & Users

```sql
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
    -- Corresponding graph node
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

### Contacts

```sql
CREATE TABLE contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    contact_type    VARCHAR(20) NOT NULL,
    first_name      VARCHAR(255),
    last_name       VARCHAR(255),
    company_name    VARCHAR(500),
    lei             VARCHAR(20),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address         JSONB NOT NULL DEFAULT '{}',
    details         JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Corresponding graph node (bidirectional link)
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contacts_tenant ON contacts(tenant_id);
CREATE INDEX idx_contacts_name ON contacts(tenant_id, last_name, first_name);
CREATE INDEX idx_contacts_company ON contacts(tenant_id, company_name);
CREATE INDEX idx_contacts_email ON contacts(tenant_id, email);
CREATE INDEX idx_contacts_graph ON contacts(graph_node_id);
```

### Matters

```sql
CREATE TABLE matters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_number   VARCHAR(50) NOT NULL,
    display_name    VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    pipeline_stage  VARCHAR(100),
    -- SALI (stored relationally + graph edges for ontology traversal)
    sali_matter_type    VARCHAR(500),                 -- SALI IRI
    sali_area_of_law    VARCHAR(500),                 -- SALI IRI
    -- Jurisdiction
    jurisdiction_country CHAR(2),
    jurisdiction_subdivision VARCHAR(6),
    court_name      VARCHAR(255),
    case_number     VARCHAR(100),
    -- Dates
    date_opened     DATE NOT NULL DEFAULT CURRENT_DATE,
    date_closed     DATE,
    statute_of_limitations DATE,
    -- Billing
    billing_method  VARCHAR(30) NOT NULL DEFAULT 'hourly',
    budget_amount   NUMERIC(12,2),
    billing_config  JSONB NOT NULL DEFAULT '{}',
    -- Assignment
    responsible_attorney_id UUID REFERENCES users(id),
    originating_attorney_id UUID REFERENCES users(id),
    practice_group  VARCHAR(100),
    -- Practice-area-specific details
    details         JSONB NOT NULL DEFAULT '{}',
    -- Corresponding graph node
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, matter_number)
);

CREATE INDEX idx_matters_tenant ON matters(tenant_id);
CREATE INDEX idx_matters_status ON matters(tenant_id, status);
CREATE INDEX idx_matters_attorney ON matters(responsible_attorney_id);
CREATE INDEX idx_matters_graph ON matters(graph_node_id);
```

### Time Tracking, Billing, Trust (same relational structure as Model 1/3)

```sql
CREATE TABLE time_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID NOT NULL REFERENCES matters(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    date            DATE NOT NULL,
    duration_hours  NUMERIC(6,2) NOT NULL,
    description     TEXT NOT NULL,
    utbms_task_code VARCHAR(10),
    utbms_activity_code VARCHAR(10),
    rate            NUMERIC(10,2) NOT NULL,
    total           NUMERIC(12,2) NOT NULL,
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    is_billed       BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    source          VARCHAR(30) NOT NULL DEFAULT 'manual',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_te_matter ON time_entries(matter_id);
CREATE INDEX idx_te_user ON time_entries(user_id);
CREATE INDEX idx_te_date ON time_entries(tenant_id, date);
CREATE INDEX idx_te_unbilled ON time_entries(matter_id) WHERE is_billable AND NOT is_billed;

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_number  VARCHAR(50) NOT NULL,
    matter_id       UUID NOT NULL REFERENCES matters(id),
    client_id       UUID NOT NULL REFERENCES contacts(id),
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    total_fees      NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_expenses  NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_tax       NUMERIC(12,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(12,2) NOT NULL DEFAULT 0,
    amount_paid     NUMERIC(12,2) NOT NULL DEFAULT 0,
    balance_due     NUMERIC(12,2) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    ledes_details   JSONB NOT NULL DEFAULT '{}',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, invoice_number)
);

CREATE INDEX idx_inv_matter ON invoices(matter_id);
CREATE INDEX idx_inv_status ON invoices(tenant_id, status);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    line_type       VARCHAR(10) NOT NULL,
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
    tax_items       JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ili_invoice ON invoice_line_items(invoice_id);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    amount          NUMERIC(12,2) NOT NULL,
    payment_method  VARCHAR(30) NOT NULL,
    payment_date    DATE NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE trust_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    account_name    VARCHAR(255) NOT NULL,
    bank_name       VARCHAR(255) NOT NULL,
    account_number_last4 CHAR(4) NOT NULL,
    jurisdiction_country CHAR(2) NOT NULL,
    jurisdiction_subdivision VARCHAR(6),
    is_iolta        BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    current_balance NUMERIC(14,2) NOT NULL DEFAULT 0,
    jurisdiction_rules JSONB NOT NULL DEFAULT '{}',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ttx_account ON trust_transactions(trust_account_id);
CREATE INDEX idx_ttx_date ON trust_transactions(transaction_date);

CREATE TABLE trust_reconciliations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trust_account_id UUID NOT NULL REFERENCES trust_accounts(id),
    reconciliation_date DATE NOT NULL,
    bank_statement_balance NUMERIC(14,2) NOT NULL,
    book_balance    NUMERIC(14,2) NOT NULL,
    client_ledger_total NUMERIC(14,2) NOT NULL,
    is_balanced     BOOLEAN NOT NULL,
    discrepancy     NUMERIC(14,2) NOT NULL DEFAULT 0,
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Documents, Tasks, Calendar

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100),
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    title           VARCHAR(500),
    category        VARCHAR(100),
    version         INTEGER NOT NULL DEFAULT 1,
    is_current_version BOOLEAN NOT NULL DEFAULT true,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Corresponding graph node (optional — for documents that participate in relationships)
    graph_node_id   UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_matter ON documents(matter_id);

CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    matter_id       UUID REFERENCES matters(id),
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    priority        VARCHAR(10) NOT NULL DEFAULT 'medium',
    due_date        DATE,
    assigned_to     UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_matter ON tasks(matter_id);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to, status);

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
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_matter ON calendar_events(matter_id);
CREATE INDEX idx_cal_date ON calendar_events(tenant_id, start_at);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(30) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    request_metadata JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_date ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Graph Query Examples

### Conflict of Interest Check

```sql
-- Full conflict check: find all entities within 3 hops of a potential new client
-- and check if any of them appear as adverse parties in existing matters
WITH RECURSIVE reachable AS (
    -- Start from the potential new client's graph node
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        0 AS depth,
        ARRAY[gn.id] AS path,
        ARRAY[gn.label] AS path_labels,
        'origin'::VARCHAR(50) AS relationship
    FROM graph_nodes gn
    WHERE gn.entity_id = 'uuid-of-potential-client'
      AND gn.entity_table = 'contacts'
      AND gn.tenant_id = 'uuid-of-tenant'

    UNION ALL

    -- Traverse edges in both directions
    SELECT
        CASE WHEN ge.from_node_id = r.node_id THEN ge.to_node_id ELSE ge.from_node_id END,
        gn2.label,
        gn2.node_type,
        r.depth + 1,
        r.path || gn2.id,
        r.path_labels || gn2.label,
        ge.edge_type
    FROM reachable r
    JOIN graph_edges ge ON (ge.from_node_id = r.node_id OR ge.to_node_id = r.node_id)
    JOIN graph_nodes gn2 ON gn2.id = CASE
        WHEN ge.from_node_id = r.node_id THEN ge.to_node_id
        ELSE ge.from_node_id
    END
    WHERE r.depth < 3                                 -- limit traversal depth
      AND NOT (gn2.id = ANY(r.path))                 -- prevent cycles
      AND (ge.valid_to IS NULL OR ge.valid_to > CURRENT_DATE)  -- only active relationships
      AND ge.edge_type IN (
          'officer_of', 'director_of', 'employee_of',
          'subsidiary_of', 'parent_of', 'shareholder_of',
          'affiliated_with', 'related_to'
      )
)
-- Now find if any reachable entity is an adverse party in an existing matter
SELECT DISTINCT
    r.node_id,
    r.label AS entity_name,
    r.node_type,
    r.depth,
    r.path_labels AS relationship_chain,
    m.display_name AS conflicting_matter,
    m.matter_number,
    adverse_edge.edge_type AS adverse_role
FROM reachable r
JOIN graph_edges adverse_edge ON adverse_edge.from_node_id = r.node_id
    AND adverse_edge.edge_type IN ('opposing_party_in', 'adverse_to')
JOIN graph_nodes matter_node ON matter_node.id = adverse_edge.to_node_id
    AND matter_node.node_type = 'matter'
JOIN matters m ON m.graph_node_id = matter_node.id
    AND m.status != 'closed'
ORDER BY r.depth, r.label;
```

### Corporate Hierarchy Traversal

```sql
-- Find the complete corporate family tree for an organisation
WITH RECURSIVE org_tree AS (
    -- Start from the target organisation
    SELECT
        gn.id AS node_id,
        gn.label AS company_name,
        gn.properties->>'lei' AS lei,
        0 AS depth,
        'root'::VARCHAR(50) AS relationship,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_id = 'uuid-of-organisation'
      AND gn.entity_table = 'contacts'

    UNION ALL

    -- Traverse parent/subsidiary relationships
    SELECT
        gn2.id,
        gn2.label,
        gn2.properties->>'lei',
        ot.depth + 1,
        ge.edge_type,
        ot.path || gn2.id
    FROM org_tree ot
    JOIN graph_edges ge ON ge.from_node_id = ot.node_id
        AND ge.edge_type IN ('subsidiary_of', 'parent_of', 'shareholder_of')
        AND (ge.valid_to IS NULL OR ge.valid_to > CURRENT_DATE)
    JOIN graph_nodes gn2 ON gn2.id = ge.to_node_id
    WHERE NOT (gn2.id = ANY(ot.path))
      AND ot.depth < 10
)
SELECT node_id, company_name, lei, depth, relationship
FROM org_tree
ORDER BY depth, company_name;
```

### Matter Network: Who Is Connected?

```sql
-- Find all contacts connected to a specific matter and their relationships
SELECT
    gn_contact.label AS contact_name,
    gn_contact.node_type,
    ge.edge_type AS role_in_matter,
    ge.properties AS role_details,
    -- Also find their connections to other matters
    (
        SELECT json_agg(json_build_object(
            'matter', gn_other_matter.label,
            'role', ge2.edge_type
        ))
        FROM graph_edges ge2
        JOIN graph_nodes gn_other_matter ON gn_other_matter.id = ge2.to_node_id
            AND gn_other_matter.node_type = 'matter'
            AND gn_other_matter.id != gn_matter.id
        WHERE ge2.from_node_id = gn_contact.id
            AND ge2.edge_type LIKE '%_in'
    ) AS other_matter_connections
FROM graph_nodes gn_matter
JOIN graph_edges ge ON ge.to_node_id = gn_matter.id
    AND ge.edge_type IN ('client_of', 'opposing_party_in', 'opposing_counsel_in',
                         'judge_in', 'witness_in', 'expert_in', 'co_counsel_in')
JOIN graph_nodes gn_contact ON gn_contact.id = ge.from_node_id
WHERE gn_matter.entity_id = 'uuid-of-matter'
  AND gn_matter.entity_table = 'matters'
ORDER BY ge.edge_type, gn_contact.label;
```

### Shortest Path Between Two Entities

```sql
-- Find the shortest relationship path between two entities
-- (e.g., "How is John Smith connected to Acme Corp?")
WITH RECURSIVE shortest_path AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        0 AS depth,
        ARRAY[gn.id] AS path,
        ARRAY[gn.label] AS path_labels,
        ARRAY[]::VARCHAR[] AS edge_types
    FROM graph_nodes gn
    WHERE gn.entity_id = 'uuid-of-john-smith'
      AND gn.entity_table = 'contacts'
      AND gn.tenant_id = 'uuid-of-tenant'

    UNION ALL

    SELECT
        gn2.id,
        gn2.label,
        sp.depth + 1,
        sp.path || gn2.id,
        sp.path_labels || gn2.label,
        sp.edge_types || ge.edge_type
    FROM shortest_path sp
    JOIN graph_edges ge ON (ge.from_node_id = sp.node_id OR ge.to_node_id = sp.node_id)
    JOIN graph_nodes gn2 ON gn2.id = CASE
        WHEN ge.from_node_id = sp.node_id THEN ge.to_node_id
        ELSE ge.from_node_id
    END
    WHERE NOT (gn2.id = ANY(sp.path))
      AND sp.depth < 6
      AND (ge.valid_to IS NULL OR ge.valid_to > CURRENT_DATE)
)
SELECT path_labels, edge_types, depth
FROM shortest_path
WHERE node_id = (
    SELECT gn.id FROM graph_nodes gn
    WHERE gn.entity_id = 'uuid-of-acme-corp'
      AND gn.entity_table = 'contacts'
)
ORDER BY depth
LIMIT 1;
```

---

## Graph Synchronisation

```sql
-- Trigger function to keep graph nodes in sync with relational tables
-- (executed on INSERT/UPDATE to contacts, matters, users)

-- Example: when a contact is created, auto-create the corresponding graph node
CREATE OR REPLACE FUNCTION sync_contact_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_nodes (tenant_id, node_type, entity_id, entity_table, label, properties)
        VALUES (
            NEW.tenant_id,
            NEW.contact_type,  -- 'individual' or 'organisation'
            NEW.id,
            'contacts',
            COALESCE(NEW.company_name, NEW.last_name || ', ' || NEW.first_name),
            jsonb_build_object(
                'first_name', NEW.first_name,
                'last_name', NEW.last_name,
                'company_name', NEW.company_name,
                'lei', NEW.lei,
                'email', NEW.email
            )
        )
        RETURNING id INTO NEW.graph_node_id;
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE graph_nodes
        SET label = COALESCE(NEW.company_name, NEW.last_name || ', ' || NEW.first_name),
            properties = jsonb_build_object(
                'first_name', NEW.first_name,
                'last_name', NEW.last_name,
                'company_name', NEW.company_name,
                'lei', NEW.lei,
                'email', NEW.email
            ),
            updated_at = now()
        WHERE id = NEW.graph_node_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_contact_graph_sync
    BEFORE INSERT OR UPDATE ON contacts
    FOR EACH ROW EXECUTE FUNCTION sync_contact_to_graph();

-- Similar triggers for matters and users
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Core Infrastructure | 2 | tenants, users |
| Contact Management | 1 | contacts (relationships in graph_edges) |
| Matter Management | 1 | matters (party roles in graph_edges) |
| Time Tracking | 1 | time_entries |
| Billing & Invoicing | 3 | invoices, invoice_line_items, payments |
| Trust Accounting | 4 | trust_accounts, trust_client_ledgers, trust_transactions, trust_reconciliations |
| Documents, Tasks, Calendar | 3 | documents, tasks, calendar_events |
| Audit | 1 | audit_log |
| **Total** | **18** | Graph replaces contact_relationships, matter_contacts, and conflict_check tables |

---

## Key Design Decisions

1. **PostgreSQL graph tables rather than a separate graph database (Neo4j)** — implementing the graph layer as `graph_nodes` / `graph_edges` tables in PostgreSQL avoids the operational complexity of running a separate database, keeps all data in one transactional context, and leverages PostgreSQL's RLS for tenant isolation. Recursive CTEs handle traversals up to ~10 hops efficiently. For very large graphs (millions of nodes), a migration path to Neo4j or Apache AGE (PostgreSQL graph extension) exists.

2. **Bidirectional entity-graph linking** — relational entities (contacts, matters, users) have a `graph_node_id` column, and graph nodes have `entity_id` / `entity_table` columns. This bidirectional link enables seamless transitions between relational CRUD operations and graph traversal queries.

3. **Graph edges replace junction tables** — the `graph_edges` table replaces three separate tables from Model 1: `matter_contacts` (party roles), `contact_relationships` (corporate hierarchies), and `conflict_check_results` (match relationships). All relationships are unified in a single edge table with typed edges, enabling cross-type traversals.

4. **Temporal validity on edges** — the `valid_from` / `valid_to` columns on `graph_edges` enable temporal relationship queries ("who were the officers of Acme Corp on January 1, 2025?"). This is critical for conflict checking, which must consider both current and historical relationships.

5. **Trigger-based graph synchronisation** — PostgreSQL triggers on the contacts, matters, and users tables automatically create and update corresponding graph nodes. This dual-write approach ensures consistency without requiring application code to manage both layers. The trade-off is that trigger-based sync adds latency to write operations.

6. **Weight column on edges for ownership analysis** — the `weight` column enables weighted graph traversals. For example, ownership edges can carry the ownership percentage (`weight: 51.0`), enabling beneficial ownership analysis ("who ultimately controls >50% of this entity?").

7. **SALI ontology as graph nodes** — SALI matter types, areas of law, and industries can be loaded as graph nodes with `child_of` edges forming the ontology hierarchy. This enables SALI-aware queries like "find all matters classified under any child of 'Commercial Litigation'" using a single recursive CTE, rather than maintaining a separate hierarchy table.

8. **Conflict checking is a graph operation, not a table scan** — instead of the text-matching approach in Model 1 (scanning contact names for fuzzy matches), conflict checking in this model starts from the potential new client's graph node and traverses outward through corporate hierarchies, officer/director relationships, and matter affiliations. This captures structural conflicts (indirect adversity through corporate parents) that text-based matching would miss.

9. **Documents optionally participate in the graph** — most documents are simple matter-linked files (relational FK). However, documents that form part of a relationship chain (e.g., a contract between two parties, an engagement letter for a specific matter-client pair) can have graph nodes, enabling queries like "show all documents connecting Client X to Matter Y."

10. **Graph enables AI-powered relationship discovery** — the graph structure is ideal for AI features like: "alert me when a new matter creates a potential conflict I haven't noticed," "suggest related matters based on shared parties," and "identify unusual relationship patterns across the firm's matter portfolio." These features require the kind of multi-hop traversal that the graph layer makes efficient.
