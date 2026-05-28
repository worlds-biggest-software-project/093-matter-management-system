# Matter Management System — Development Plan

> Project: Matter Management System (Candidate #93)
> Created: 2026-05-25
> Status: Phased development plan — ready for implementation

---

## Technology Decisions & Rationale

### Data Model: Hybrid Relational + JSONB (Model 3) with Graph Extension (Model 4)

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the primary schema, augmented with the graph layer from Data Model Suggestion 4 for conflict checking and relationship traversal — introduced in a later phase.

**Rationale:**
- Model 3 provides 22 tables (vs. 39 in the fully normalised Model 1), accelerating MVP delivery without sacrificing relational integrity on core entities (matters, invoices, trust accounts)
- JSONB columns eliminate the EAV anti-pattern for practice-area-specific fields and absorb jurisdiction-specific metadata (trust rules, LEDES extensions, tax configurations) without schema migrations
- Model 2 (event-sourced CQRS) is architecturally elegant for audit but imposes significant implementation complexity on the team; the trigger-based audit log from Model 3 meets compliance needs for the MVP, with an event-sourcing migration path available later if trust accounting auditors demand it
- Model 4's graph layer is deferred to Phase 7 because conflict checking (while important) is not blocking for initial matter/billing workflows. The graph tables (`graph_nodes`, `graph_edges`) can be added incrementally alongside the existing relational tables

### Backend: TypeScript + Node.js + Fastify

**Rationale:**
- TypeScript provides compile-time type safety essential for financial calculations (billing, trust accounting) without the runtime overhead of Java/Spring
- Fastify outperforms Express in benchmarks and provides built-in schema validation via JSON Schema (aligns with Model 3's JSONB schema validation requirement)
- The legal tech developer community skews toward JavaScript/TypeScript; maximises contributor pool for an OSS project
- Prisma ORM handles PostgreSQL migrations, JSONB typing, and row-level security integration cleanly

### Frontend: Next.js 15 (App Router) + React + shadcn/ui

**Rationale:**
- Next.js App Router provides server-side rendering for the matter dashboard (SEO irrelevant, but SSR reduces client-side JavaScript for attorneys on slower connections in courthouses)
- shadcn/ui provides accessible, customisable components without the bundle size of Material UI
- React ecosystem has the richest set of components for the UX patterns identified in competitor analysis: kanban boards (react-beautiful-dnd), data tables (TanStack Table), rich text editors (Tiptap), and calendar views (FullCalendar)
- Server components enable streaming large matter lists and document search results

### Database: PostgreSQL 16

**Rationale:**
- Only database engine that provides all of: JSONB with GIN indexes, row-level security policies for multi-tenant isolation, recursive CTEs for graph traversal, and full-text search — all required by the chosen data model
- RLS is the standard multi-tenancy pattern used by Supabase and Nile, providing defence-in-depth for tenant data isolation
- pgvector extension available for future AI-powered semantic search across documents and matter descriptions

### File Storage: S3-Compatible Object Storage (MinIO for self-hosted, AWS S3 for SaaS)

**Rationale:**
- Legal documents can be large (multi-GB depositions, medical records) — object storage scales without impacting database performance
- MinIO provides an S3-compatible self-hosted option critical for the open-source deployment target
- Presigned URLs enable direct client-to-storage uploads/downloads, reducing backend load

### Authentication: Lucia Auth + OIDC Provider Support

**Rationale:**
- Lucia is a lightweight, framework-agnostic auth library that avoids the vendor lock-in of Auth0/Clerk while being more robust than hand-rolled JWT
- OIDC support is mandatory for enterprise law firms using Microsoft Entra ID or Okta SSO (identified as a requirement in the standards research)
- Session-based auth with CSRF protection aligns better with legal compliance than stateless JWTs for a security-sensitive application

### Billing & Payments: Stripe Connect (delegated PCI compliance)

**Rationale:**
- Delegating card processing to Stripe limits PCI DSS scope to SAQ-A (the simplest self-assessment), avoiding the multi-month certification process
- Stripe Connect supports the trust account deposit workflow (client pays into trust, firm draws from trust to pay invoices)
- LawPay (the legal industry standard) is built on Stripe Connect, validating this approach

### AI Layer: Claude API via Anthropic SDK with MCP Server

**Rationale:**
- Claude demonstrates strong performance on legal document analysis and structured data extraction tasks
- MCP server exposure aligns with the standards research recommendation: AI agents can access matter data via standardised MCP protocol
- Prompt caching reduces costs for repetitive operations (intake classification, time entry suggestions)
- The AI layer is explicitly modular — the system must function fully without AI features for self-hosted deployments that cannot use cloud AI

### Testing: Vitest (unit/integration) + Playwright (E2E) + pgTAP (database)

**Rationale:**
- Vitest is significantly faster than Jest for TypeScript projects and supports concurrent test execution
- Playwright provides cross-browser E2E testing essential for the client portal (attorneys and clients use different browsers)
- pgTAP enables testing database-level logic (RLS policies, trust accounting triggers) directly in PostgreSQL

---

## Project Structure

```
matter-management-system/
├── apps/
│   ├── api/                          # Fastify backend
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/             # Authentication, OIDC, sessions
│   │   │   │   ├── tenants/          # Multi-tenancy management
│   │   │   │   ├── contacts/         # Contact CRUD, search
│   │   │   │   ├── matters/          # Matter lifecycle, templates, pipeline
│   │   │   │   ├── time-tracking/    # Timers, time entries, rounding
│   │   │   │   ├── billing/          # Invoices, LEDES export, payments
│   │   │   │   ├── trust/            # Trust accounting, reconciliation
│   │   │   │   ├── documents/        # Upload, versioning, search
│   │   │   │   ├── tasks/            # Task management, deadlines
│   │   │   │   ├── calendar/         # Calendar events, court rules
│   │   │   │   ├── communications/   # Portal messages, comm log
│   │   │   │   ├── conflicts/        # Conflict checking (graph-based)
│   │   │   │   ├── reporting/        # Matter, billing, realisation reports
│   │   │   │   └── ai/              # AI intake, time capture, budgeting
│   │   │   ├── shared/
│   │   │   │   ├── db/               # Prisma client, RLS middleware
│   │   │   │   ├── auth/             # Auth middleware, RBAC
│   │   │   │   ├── validation/       # JSON Schema validators
│   │   │   │   ├── ledes/            # LEDES XML/1998B serialisers
│   │   │   │   ├── utbms/            # UTBMS code loader and validator
│   │   │   │   ├── sali/             # SALI LMSS ontology loader
│   │   │   │   └── storage/          # S3/MinIO file operations
│   │   │   ├── mcp/                  # MCP server for AI agent access
│   │   │   └── app.ts               # Fastify app bootstrap
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── tests/
│   ├── web/                          # Next.js frontend
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── (auth)/           # Login, SSO callback
│   │   │   │   ├── (dashboard)/      # Main attorney-facing UI
│   │   │   │   │   ├── matters/
│   │   │   │   │   ├── contacts/
│   │   │   │   │   ├── time/
│   │   │   │   │   ├── billing/
│   │   │   │   │   ├── documents/
│   │   │   │   │   ├── calendar/
│   │   │   │   │   ├── trust/
│   │   │   │   │   └── reports/
│   │   │   │   └── portal/           # Client-facing portal
│   │   │   ├── components/
│   │   │   │   ├── ui/               # shadcn/ui components
│   │   │   │   ├── matters/
│   │   │   │   ├── billing/
│   │   │   │   ├── time/
│   │   │   │   └── shared/
│   │   │   └── lib/
│   │   │       ├── api-client.ts     # Type-safe API client
│   │   │       └── hooks/
│   │   └── tests/
│   └── mobile/                       # React Native (Phase 9)
├── packages/
│   ├── shared-types/                 # Shared TypeScript types
│   ├── ledes/                        # LEDES format library (publishable)
│   ├── utbms/                        # UTBMS code reference (publishable)
│   ├── sali/                         # SALI LMSS loader (publishable)
│   └── trust-accounting/             # Trust accounting logic (publishable)
├── docker/
│   ├── docker-compose.yml            # Local dev environment
│   ├── docker-compose.prod.yml       # Self-hosted production
│   └── Dockerfile
├── docs/
│   ├── api/                          # OpenAPI 3.1 spec (auto-generated)
│   ├── deployment/
│   └── security/                     # SOC 2 control documentation
└── turbo.json                        # Turborepo config
```

---

## Phase Dependency Graph

```
Phase 1 ──────────────────────────────────┐
(Foundation)                              │
    │                                     │
    ├──► Phase 2 ──► Phase 3 ──► Phase 4  │
    │   (Matters)   (Time)     (Billing)  │
    │       │                     │       │
    │       │                     ▼       │
    │       │              Phase 5        │
    │       │          (Trust Acctg)      │
    │       │                             │
    │       ├──► Phase 6                  │
    │       │   (Client Portal)           │
    │       │                             │
    │       └──► Phase 7                  │
    │           (Conflicts/Graph)         │
    │                                     │
    ├──► Phase 8 ◄────────────────────────┘
    │   (AI Layer)   requires Phases 2-4
    │
    ├──► Phase 9
    │   (Mobile)     requires Phases 2,3,6
    │
    └──► Phase 10
        (Integrations) requires Phase 4
```

**Critical path:** Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5

All phases after Phase 5 can proceed in parallel once their dependencies are met.

---

## Phase 1: Foundation & Infrastructure

**Duration:** 4 weeks
**Dependencies:** None
**Goal:** Establish the monorepo, database, authentication, multi-tenancy, and CI/CD pipeline. At phase end, a developer can sign up, create a tenant, and authenticate via email/password or OIDC.

### Task 1.1: Monorepo & Tooling Setup

**What:** Initialise the Turborepo monorepo with `apps/api` (Fastify), `apps/web` (Next.js), and `packages/shared-types`. Configure ESLint, Prettier, TypeScript strict mode, and Husky pre-commit hooks.

**Design:**
```
turbo.json — pipeline config for build, dev, test, lint
apps/api/package.json — fastify, @fastify/cors, @fastify/cookie, prisma, vitest
apps/web/package.json — next@15, react@19, tailwindcss, shadcn/ui, vitest
packages/shared-types/src/index.ts — shared enums (MatterStatus, ContactType, etc.)
```
```typescript
// packages/shared-types/src/index.ts
export enum MatterStatus { Open = 'open', Pending = 'pending', Closed = 'closed', Archived = 'archived' }
export enum ContactType { Individual = 'individual', Organisation = 'organisation' }
export enum BillingMethod { Hourly = 'hourly', FlatFee = 'flat_fee', Contingency = 'contingency', Blended = 'blended', ProBono = 'pro_bono' }
export enum TimeEntryStatus { Draft = 'draft', Approved = 'approved', Billed = 'billed', WrittenOff = 'written_off' }
export enum InvoiceStatus { Draft = 'draft', PendingReview = 'pending_review', Approved = 'approved', Sent = 'sent', Paid = 'paid', Partial = 'partial', Overdue = 'overdue', Void = 'void' }
```

**Testing:**
- `turbo run build` succeeds across all packages and apps with zero errors
- `turbo run lint` passes with zero warnings (strict config)
- `turbo run test` runs empty test suites in all apps/packages and reports success
- TypeScript strict mode enabled: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitReturns`
- Hot reload works: changing `shared-types` triggers rebuild in `api` and `web`

### Task 1.2: Database Schema & Migrations

**What:** Create the PostgreSQL 16 schema covering tenants, users, roles, JSONB schema registry, and the audit log. Configure Prisma with custom SQL migrations for RLS policies. Set up Docker Compose for local PostgreSQL.

**Design:**
```prisma
// apps/api/prisma/schema.prisma (core tables — Phases 1-2 only)
model Tenant {
  id        String   @id @default(uuid()) @db.Uuid
  name      String   @db.VarChar(255)
  slug      String   @unique @db.VarChar(100)
  planTier  String   @default("free") @db.VarChar(50)
  settings  Json     @default("{}")
  billingConfig Json @default("{}")
  branding  Json     @default("{}")
  createdAt DateTime @default(now()) @db.Timestamptz
  updatedAt DateTime @updatedAt @db.Timestamptz
  users     User[]
}

model User {
  id           String   @id @default(uuid()) @db.Uuid
  tenantId     String   @db.Uuid
  email        String   @db.VarChar(255)
  passwordHash String?  @db.VarChar(255)
  fullName     String   @db.VarChar(255)
  role         String   @default("user") @db.VarChar(50)
  profile      Json     @default("{}")
  permissions  Json     @default("[]")
  isActive     Boolean  @default(true)
  lastLoginAt  DateTime? @db.Timestamptz
  createdAt    DateTime @default(now()) @db.Timestamptz
  updatedAt    DateTime @updatedAt @db.Timestamptz
  tenant       Tenant   @relation(fields: [tenantId], references: [id])
  @@unique([tenantId, email])
  @@index([tenantId])
}
```
```sql
-- Custom migration: RLS policies
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_users ON users
    USING (tenant_id = current_setting('app.current_tenant')::UUID);
```

**Testing:**
- `prisma migrate dev` creates all tables, indexes, and RLS policies without errors
- RLS test: inserting a row with tenant_id A, then setting `app.current_tenant` to tenant B, and querying returns zero rows
- `prisma db seed` loads a test tenant with an admin user
- Docker Compose `docker compose up db` starts PostgreSQL 16 with the pgTAP extension
- pgTAP tests: `SELECT plan(3); SELECT has_table('tenants'); SELECT has_table('users'); SELECT col_is_pk('tenants', 'id'); SELECT * FROM finish();`

### Task 1.3: Authentication & Session Management

**What:** Implement email/password authentication with Lucia, session cookie management, and CSRF protection. Add OIDC provider support (Microsoft Entra ID, Google Workspace) for SSO.

**Design:**
```typescript
// apps/api/src/modules/auth/routes.ts
import { Fastify } from 'fastify';
import { lucia } from './lucia';
import { Argon2id } from 'oslo/password';

export async function authRoutes(app: Fastify) {
  app.post('/api/auth/signup', {
    schema: {
      body: { type: 'object', required: ['email', 'password', 'fullName', 'tenantSlug'],
        properties: {
          email: { type: 'string', format: 'email' },
          password: { type: 'string', minLength: 12 },  // PCI DSS v4.0.1
          fullName: { type: 'string', minLength: 1 },
          tenantSlug: { type: 'string', pattern: '^[a-z0-9-]+$' }
        }
      }
    }
  }, async (req, reply) => {
    const hashedPassword = await new Argon2id().hash(req.body.password);
    // Create user, create session, set cookie
  });

  app.post('/api/auth/login', { /* ... */ });
  app.post('/api/auth/logout', { /* ... */ });
  app.get('/api/auth/oidc/authorize/:provider', { /* OIDC redirect */ });
  app.get('/api/auth/oidc/callback/:provider', { /* OIDC callback */ });
}
```

**Testing:**
- Signup with valid credentials creates user and returns session cookie
- Signup with password < 12 characters returns 400 (PCI DSS v4.0.1 requirement)
- Signup with duplicate email returns 409
- Login with valid credentials returns session cookie with `Secure`, `HttpOnly`, `SameSite=Lax`
- Login with invalid credentials returns 401 (no information leakage — same message for wrong email vs. wrong password)
- Authenticated request with valid session cookie succeeds
- Request with expired session cookie returns 401
- CSRF token validation: POST without CSRF token returns 403
- OIDC flow: mock provider returns valid ID token → user created/linked → session established
- Rate limiting: 10 failed login attempts in 5 minutes triggers 429

### Task 1.4: Multi-Tenancy Middleware

**What:** Implement Fastify middleware that extracts tenant context from the authenticated session, sets the PostgreSQL `app.current_tenant` variable for RLS, and provides tenant-scoped Prisma operations.

**Design:**
```typescript
// apps/api/src/shared/db/tenant-middleware.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { prisma } from './prisma';

export async function tenantMiddleware(req: FastifyRequest, reply: FastifyReply) {
  const session = req.session; // set by auth middleware
  if (!session?.tenantId) {
    return reply.status(401).send({ error: 'No tenant context' });
  }
  // Set RLS context for this request
  await prisma.$executeRawUnsafe(
    `SET LOCAL app.current_tenant = '${session.tenantId}'`
  );
  req.tenantId = session.tenantId;
}
```

**Testing:**
- Request without session returns 401
- Request with valid session sets `app.current_tenant` correctly (verified via `SHOW app.current_tenant`)
- Tenant A user cannot access Tenant B data (RLS enforced at DB level)
- Concurrent requests from different tenants each have isolated RLS context (no cross-contamination)
- `req.tenantId` is available in all downstream route handlers

### Task 1.5: Audit Log Infrastructure

**What:** Create the audit log table and a Fastify plugin that automatically logs all create/update/delete operations with JSONB diffs, user identity, and request metadata.

**Design:**
```typescript
// apps/api/src/shared/db/audit-plugin.ts
export function auditLog(entityType: string, action: string, entityId: string, changes: object, req: FastifyRequest) {
  return prisma.auditLog.create({
    data: {
      tenantId: req.tenantId,
      userId: req.session.userId,
      action,
      entityType,
      entityId,
      changes: changes as Prisma.InputJsonValue,
      requestMetadata: {
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        correlationId: req.id
      }
    }
  });
}
```

**Testing:**
- Creating a user generates an audit log entry with action `create`, entity_type `user`, and the full created entity as `changes`
- Updating a user generates a `changes` object containing `{ field: { old: x, new: y } }` for every changed field
- Audit log entries include correct `ip`, `user_agent`, and `correlation_id`
- Audit log entries are tenant-scoped (RLS applies)
- Audit log is append-only: UPDATE and DELETE on audit_log are rejected by database policy

### Task 1.6: CI/CD Pipeline

**What:** Configure GitHub Actions for lint, type-check, unit tests, integration tests (with PostgreSQL service container), and Docker image builds. Add Dependabot for dependency updates.

**Design:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma migrate deploy
      - run: pnpm turbo test
```

**Testing:**
- CI pipeline completes green on a clean commit
- CI fails fast if lint errors exist (exits before running tests)
- PostgreSQL service container starts and migrations apply successfully in CI
- Test coverage report is generated and uploaded as an artefact
- Docker image builds successfully and starts without errors

### Definition of Done — Phase 1
- [ ] Monorepo builds, lints, and type-checks with zero errors
- [ ] PostgreSQL 16 schema with RLS policies deployed via Prisma migrations
- [ ] Email/password authentication working with session cookies
- [ ] OIDC SSO working with at least one provider (Microsoft Entra ID)
- [ ] Multi-tenant RLS isolation verified by cross-tenant access test
- [ ] Audit log capturing all CUD operations with JSONB diffs
- [ ] CI/CD pipeline green on main branch
- [ ] Docker Compose local development environment documented and working
- [ ] OpenAPI 3.1 spec auto-generated from Fastify route schemas

---

## Phase 2: Matter & Contact Management

**Duration:** 4 weeks
**Dependencies:** Phase 1
**Goal:** Users can create contacts, open matters with SALI classification, manage matter templates, view matters in list and kanban views, and track tasks/deadlines.

### Task 2.1: Contact CRUD & Search

**What:** Implement contact creation (individual and organisation), editing, search (by name, email, company), and the hybrid relational+JSONB details field with JSON Schema validation.

**Design:**
```typescript
// apps/api/src/modules/contacts/routes.ts
app.post('/api/contacts', {
  schema: {
    body: {
      type: 'object',
      required: ['contactType', 'displayName'],
      properties: {
        contactType: { enum: ['individual', 'organisation'] },
        firstName: { type: 'string' },
        lastName: { type: 'string' },
        companyName: { type: 'string' },
        email: { type: 'string', format: 'email' },
        phone: { type: 'string' },
        details: { type: 'object' }  // validated against jsonb_schemas
      }
    }
  }
}, async (req, reply) => {
  // Validate details against registered JSON Schema if provided
  // Create contact with audit log
});

app.get('/api/contacts', {
  schema: {
    querystring: {
      q: { type: 'string' },        // full-text search
      type: { enum: ['individual', 'organisation'] },
      page: { type: 'integer', minimum: 1, default: 1 },
      limit: { type: 'integer', minimum: 1, maximum: 100, default: 25 }
    }
  }
});
```

**Testing:**
- Create individual contact with first_name, last_name, email → 201 with UUID returned
- Create organisation contact with company_name, LEI → 201
- Create contact with invalid email → 400
- Search by partial last_name returns matching contacts (case-insensitive)
- Search by email returns exact match
- JSONB details validated against registered schema: invalid field type → 400
- Contacts are tenant-isolated: Tenant A contacts not visible to Tenant B
- Pagination: page=2, limit=10 returns contacts 11-20
- Audit log entry created for every contact CUD operation

### Task 2.2: SALI LMSS Ontology Loader

**What:** Build a package (`packages/sali`) that parses the SALI LMSS OWL ontology export and loads matter types, areas of law, and industries into the `jsonb_schemas` registry. Provide a search/autocomplete endpoint for matter classification.

**Design:**
```typescript
// packages/sali/src/loader.ts
export interface SaliTag {
  iri: string;
  code: string;
  label: string;
  parentIri?: string;
  children: SaliTag[];
}

export async function loadSaliOntology(owlFilePath: string): Promise<{
  matterTypes: SaliTag[];
  areasOfLaw: SaliTag[];
  industries: SaliTag[];
}> {
  // Parse OWL/RDF XML, extract SALI class hierarchy
  // Return structured tag trees
}

// apps/api/src/modules/matters/sali-routes.ts
app.get('/api/sali/search', {
  schema: { querystring: { q: { type: 'string', minLength: 2 }, category: { enum: ['matter_type', 'area_of_law', 'industry'] } } }
}, async (req, reply) => {
  // Search SALI tags by label prefix match
  // Return top 20 matches with IRI, label, parent chain
});
```

**Testing:**
- `loadSaliOntology()` parses the official SALI OWL export without errors
- Search for "breach" returns "Breach of Contract" among results
- Search returns parent chain for hierarchical display (e.g., "Commercial Litigation > Contract Disputes > Breach of Contract")
- Search is case-insensitive
- Empty search returns 400

### Task 2.3: Matter CRUD & Templates

**What:** Implement matter creation with SALI classification, configurable JSONB details, matter templates, and matter lifecycle (open → pending → closed → archived).

**Design:**
```typescript
// apps/api/src/modules/matters/routes.ts
app.post('/api/matters', {
  schema: {
    body: {
      type: 'object',
      required: ['displayName'],
      properties: {
        displayName: { type: 'string' },
        templateId: { type: 'string', format: 'uuid' },
        saliClassification: { type: 'object' },
        jurisdiction: { type: 'object' },
        billing: { type: 'object' },
        details: { type: 'object' },
        responsibleAttorneyId: { type: 'string', format: 'uuid' }
      }
    }
  }
}, async (req, reply) => {
  // Auto-generate matter_number (tenant-specific sequence)
  // If templateId provided, pre-populate tasks, pipeline stages
  // Validate details against template's JSON Schema
  // Create matter + audit log
});
```

**Testing:**
- Create matter with display_name → auto-generated matter_number (e.g., "2026-00001")
- Create matter from template → tasks, pipeline stages, and billing defaults pre-populated
- Assign SALI classification → `sali_classification` JSONB stored with IRI and label
- Update matter status from `open` to `closed` → `date_closed` auto-set
- Reopen closed matter → status back to `open`, `date_closed` cleared
- Create matter with invalid billing method → 400
- Matter details validated against JSON Schema referenced by template
- Matter number is unique per tenant

### Task 2.4: Matter-Contact Roles

**What:** Link contacts to matters with typed roles (client, opposing party, opposing counsel, judge, witness, expert witness, co-counsel). Support primary client designation.

**Design:**
```typescript
app.post('/api/matters/:matterId/contacts', {
  schema: {
    body: {
      type: 'object',
      required: ['contactId', 'role'],
      properties: {
        contactId: { type: 'string', format: 'uuid' },
        role: { enum: ['client', 'opposing_party', 'opposing_counsel', 'judge', 'witness', 'expert_witness', 'co_counsel'] },
        isPrimary: { type: 'boolean', default: false },
        metadata: { type: 'object' }
      }
    }
  }
});
```

**Testing:**
- Assign contact as client on matter → 201
- Assign same contact with same role again → 409 (duplicate)
- Assign same contact with different role → 201 (allowed: a contact can be client on one matter and witness on another)
- Set `isPrimary: true` → only one primary client per matter (previous primary unset)
- Remove contact from matter → 200, audit logged
- Query matter contacts → returns all contacts with roles and metadata

### Task 2.5: Task & Deadline Management

**What:** Create tasks linked to matters with due dates, priorities, and assignees. Support task templates that auto-create tasks from matter templates.

**Design:**
```typescript
app.post('/api/matters/:matterId/tasks', {
  schema: {
    body: {
      type: 'object',
      required: ['title'],
      properties: {
        title: { type: 'string' },
        description: { type: 'string' },
        priority: { enum: ['low', 'medium', 'high', 'urgent'], default: 'medium' },
        dueDate: { type: 'string', format: 'date' },
        assignedTo: { type: 'string', format: 'uuid' }
      }
    }
  }
});
```

**Testing:**
- Create task with due date → 201
- Mark task as completed → `completed_at` timestamp set automatically
- Query overdue tasks → returns tasks where `due_date < today` and `status != completed`
- Create matter from template with 5 default tasks → 5 tasks created with offset-calculated due dates
- Assign task to user not in tenant → 400

### Task 2.6: Matter Dashboard & Kanban UI

**What:** Build the frontend matter list view (sortable/filterable table), matter detail page, and kanban pipeline view with drag-and-drop stage transitions.

**Design:**
```typescript
// apps/web/src/app/(dashboard)/matters/page.tsx
// Server component that fetches matters with pagination and renders TanStack Table
// Columns: matter_number, display_name, status, pipeline_stage, responsible_attorney, date_opened

// apps/web/src/components/matters/kanban-board.tsx
// Client component using @hello-pangea/dnd for drag-and-drop
// Lanes defined by pipeline_stages from matter template
// Dropping a matter into a new lane calls PATCH /api/matters/:id with new pipeline_stage
```

**Testing:**
- Matter list loads with server-side pagination (25 per page default)
- Sorting by matter_number, date_opened, status works correctly
- Filtering by status, practice_group, responsible_attorney returns correct results
- Kanban board renders matters grouped by pipeline_stage
- Drag-and-drop matter from "Discovery" to "Trial Prep" → API call updates pipeline_stage
- Matter detail page shows all linked contacts, tasks, and metadata
- Empty state: no matters → helpful onboarding prompt shown

### Definition of Done — Phase 2
- [ ] Contact CRUD with JSONB details and JSON Schema validation
- [ ] SALI LMSS ontology loaded and searchable via API
- [ ] Matter CRUD with SALI classification, templates, and lifecycle management
- [ ] Matter-contact role assignments with primary client support
- [ ] Task and deadline management with template-based auto-creation
- [ ] Frontend: matter list (table), kanban board, matter detail page, contact list
- [ ] All endpoints documented in OpenAPI spec
- [ ] Integration tests covering cross-tenant isolation for all new entities

---

## Phase 3: Time Tracking

**Duration:** 3 weeks
**Dependencies:** Phase 2
**Goal:** Attorneys can record time entries, run multiple timers, apply UTBMS codes, and review/approve time entries. Time rounding rules are configurable per tenant.

### Task 3.1: UTBMS Code Reference Data

**What:** Build a package (`packages/utbms`) that loads the five UTBMS code sets (Litigation, IP, Counseling, Project, Bankruptcy) and provides lookup/validation functions. Create an API endpoint for code search.

**Design:**
```typescript
// packages/utbms/src/codes.ts
export interface UtbmsCode { code: string; description: string; codeSet: string; type: 'phase' | 'task' | 'activity' | 'expense'; parentCode?: string; }
export function getTaskCodes(codeSet: string): UtbmsCode[];
export function getActivityCodes(): UtbmsCode[];
export function validateCode(code: string): boolean;
```

**Testing:**
- `getTaskCodes('litigation')` returns all L-series task codes
- `getActivityCodes()` returns all A-series activity codes
- `validateCode('L110')` returns true; `validateCode('ZZZZ')` returns false
- API search for "deposition" returns relevant task codes

### Task 3.2: Time Entry CRUD

**What:** Create, read, update, and delete time entries with UTBMS code assignment, billable/non-billable flag, and source tracking (manual, timer, ai_suggestion).

**Design:**
```typescript
app.post('/api/time-entries', {
  schema: {
    body: {
      type: 'object',
      required: ['matterId', 'date', 'durationHours', 'description'],
      properties: {
        matterId: { type: 'string', format: 'uuid' },
        date: { type: 'string', format: 'date' },
        durationHours: { type: 'number', minimum: 0.01, maximum: 24 },
        description: { type: 'string', minLength: 1 },
        utbmsTaskCode: { type: 'string' },
        utbmsActivityCode: { type: 'string' },
        isBillable: { type: 'boolean', default: true },
        source: { enum: ['manual', 'timer', 'ai_suggestion'], default: 'manual' }
      }
    }
  }
}, async (req, reply) => {
  // Look up attorney's hourly rate (matter override > user default)
  // Apply rounding rules from tenant config
  // Calculate total = rate * rounded_duration
  // Store both raw and rounded duration
});
```

**Testing:**
- Create time entry with 1.42 hours, 6-minute rounding (up) → `duration_hours` = 1.50, `duration_raw` in metadata = 1.42
- Rate calculated from matter billing config override, falling back to user's default hourly rate
- Total = rate * duration_hours (verified to 2 decimal places)
- Invalid UTBMS code → 400
- Time entry for closed matter → 400
- Time entry for matter in different tenant → 403 (RLS)
- Delete billed time entry → 400 (cannot delete billed entries)

### Task 3.3: Timer Management

**What:** Implement start/stop/pause timers that run server-side. Users can have multiple timers but only one running at a time. Converting a timer to a time entry.

**Design:**
```typescript
app.post('/api/timers', { /* create and start timer */ });
app.patch('/api/timers/:id/pause', { /* pause running timer */ });
app.patch('/api/timers/:id/resume', { /* resume paused timer */ });
app.post('/api/timers/:id/stop', { /* stop timer, create time entry */ });
app.delete('/api/timers/:id', { /* discard timer without saving */ });
```

**Testing:**
- Start timer → is_running = true, started_at set
- Pause timer → elapsed_seconds incremented, paused_at set
- Resume timer → is_running = true, paused_at cleared
- Stop timer → time entry created with duration = elapsed_seconds converted to hours
- Start second timer while one running → first timer auto-paused
- Timer for user in different tenant → 403

### Task 3.4: Time Entry Approval Workflow

**What:** Partners/admins can review and approve time entries. Approved entries cannot be edited without re-entering the approval flow. Support bulk approval.

**Design:**
```typescript
app.patch('/api/time-entries/bulk-approve', {
  schema: {
    body: {
      type: 'object',
      required: ['timeEntryIds'],
      properties: {
        timeEntryIds: { type: 'array', items: { type: 'string', format: 'uuid' }, minItems: 1, maxItems: 100 }
      }
    }
  }
}, async (req, reply) => {
  // Verify all entries are in 'draft' status
  // Set status = 'approved', approved_by, approved_at
  // Audit log each approval
});
```

**Testing:**
- Approve draft time entry → status changes to `approved`, `approved_by` and `approved_at` set
- Approve already-approved entry → 400
- Edit approved entry → status reverts to `draft` (requires re-approval)
- Bulk approve 50 entries → all 50 updated in single transaction
- Non-partner user attempts approval → 403 (RBAC enforced)

### Task 3.5: Time Tracking UI

**What:** Build the time entry list page (filterable by date range, matter, attorney), timer widget (floating, persistent across navigation), and time entry form with UTBMS code picker.

**Design:**
```typescript
// apps/web/src/components/time/timer-widget.tsx
// Persistent floating component showing active timer(s)
// Start/pause/stop controls
// Matter selector dropdown
// Description input
// Click "Stop" → opens time entry form pre-filled with timer data
```

**Testing:**
- Timer widget visible on all dashboard pages
- Starting timer shows elapsed time counting up in real-time
- Stopping timer opens time entry form with pre-filled duration and matter
- Time entry form UTBMS code picker shows searchable dropdown with code descriptions
- Time entry list filters by date range (this week, last week, custom) work correctly
- Daily/weekly time summary displayed at top of list

### Definition of Done — Phase 3
- [ ] UTBMS code reference data loaded and searchable
- [ ] Time entry CRUD with rounding rules, rate calculation, and UTBMS code assignment
- [ ] Timer management: start, pause, resume, stop → time entry conversion
- [ ] Approval workflow: draft → approved → billed lifecycle
- [ ] Frontend: time entry list, timer widget, time entry form with UTBMS picker
- [ ] Rounding rules configurable per tenant (6-min, 10-min, 15-min increments; round up/nearest)
- [ ] Integration tests: time entry rate calculation, rounding, and approval flow

---

## Phase 4: Billing & Invoicing

**Duration:** 5 weeks
**Dependencies:** Phase 3
**Goal:** Users can generate invoices from approved time entries, produce LEDES-formatted exports, and record payments. The billing module is the revenue-critical path and requires the most thorough testing.

### Task 4.1: Invoice Generation

**What:** Create invoices by selecting approved, unbilled time entries for a matter. Auto-populate invoice line items from time entries with UTBMS codes, rates, and amounts. Support expense line items entered manually.

**Design:**
```typescript
app.post('/api/invoices/generate', {
  schema: {
    body: {
      type: 'object',
      required: ['matterId'],
      properties: {
        matterId: { type: 'string', format: 'uuid' },
        timeEntryIds: { type: 'array', items: { type: 'string', format: 'uuid' } },
        // If omitted, includes all approved unbilled entries for the matter
        invoiceDate: { type: 'string', format: 'date' },
        dueDateDays: { type: 'integer', default: 30 },
        expenses: { type: 'array', items: {
          type: 'object', required: ['description', 'amount'],
          properties: {
            description: { type: 'string' },
            amount: { type: 'number' },
            utbmsExpenseCode: { type: 'string' },
            date: { type: 'string', format: 'date' }
          }
        }}
      }
    }
  }
}, async (req, reply) => {
  // Generate invoice_number (tenant-specific sequence)
  // Create invoice header with totals
  // Create fee line items from time entries
  // Create expense line items from manual entries
  // Mark time entries as is_billed = true
  // All in a single transaction
});
```

**Testing:**
- Generate invoice from 5 time entries → invoice with 5 fee line items, correct totals
- total_fees = sum of all fee line items
- total_expenses = sum of all expense line items
- total_amount = total_fees + total_expenses + total_tax - total_discounts
- Invoice number auto-increments per tenant (e.g., "INV-2026-0001")
- Attempting to include already-billed time entries → 400
- Attempting to include non-approved time entries → 400
- Transaction rollback: if invoice creation fails, time entries remain unbilled
- Generating invoice for matter with no unbilled time → 400

### Task 4.2: LEDES Export Engine

**What:** Build a package (`packages/ledes`) that serialises invoices into LEDES XML 2.2 and LEDES 1998B formats. The serialiser reads from the invoice data model and produces standards-compliant output.

**Design:**
```typescript
// packages/ledes/src/xml22.ts
export function generateLedesXml22(invoice: LedesInvoiceData): string {
  // Produce XML conforming to LEDES XML 2.2 schema
  // @INVOICE → @MATTER → @FEE/@EXPENSE → @TAX_ITEM_FEE/@TAX_ITEM_EXPENSE
  // Include tiered tax support (e.g., Japanese Withholding Tax)
}

// packages/ledes/src/1998b.ts
export function generateLedes1998B(invoice: LedesInvoiceData): string {
  // Produce pipe-delimited flat file conforming to LEDES 1998B spec
  // Header: LEDES1998B[]
  // Columns: INVOICE_DATE|INVOICE_NUMBER|CLIENT_ID|...
}
```

**Testing:**
- XML 2.2 output validates against the official LEDES XML 2.2 XSD
- 1998B output parses correctly with pipe delimiter and correct column count
- Invoice with VAT tax items produces correct @TAX_ITEM_FEE segments
- Invoice with tiered taxes (VAT + Withholding) produces multiple tax items per line
- UTBMS codes appear in correct LEDES fields (TASK_CODE, ACTIVITY_CODE)
- Timekeeper classification (partner, associate, paralegal) appears in LEDES output
- Round-trip test: generate LEDES XML → parse back → amounts match original invoice

### Task 4.3: Invoice Approval & Delivery Workflow

**What:** Implement the invoice lifecycle: draft → pending_review → approved → sent → paid/partial/overdue. Support email delivery with PDF attachment.

**Design:**
```typescript
// Invoice state machine
const INVOICE_TRANSITIONS = {
  draft: ['pending_review', 'void'],
  pending_review: ['approved', 'draft'],
  approved: ['sent', 'draft'],
  sent: ['paid', 'partial', 'overdue', 'void'],
  partial: ['paid', 'overdue', 'void'],
  overdue: ['paid', 'partial', 'void'],
};

app.patch('/api/invoices/:id/transition', {
  schema: {
    body: {
      type: 'object',
      required: ['targetStatus'],
      properties: { targetStatus: { type: 'string' } }
    }
  }
});
```

**Testing:**
- Transition draft → pending_review → approved → sent: each step updates status and audit log
- Transition sent → paid: sets `paid_at`, updates `amount_paid` and `balance_due`
- Invalid transition (e.g., draft → paid) → 400
- Void invoice → time entries' `is_billed` reset to false (entries become available for re-billing)
- Only admin/partner can approve (RBAC enforced)

### Task 4.4: Payment Recording

**What:** Record payments against invoices. Support multiple payment methods (check, wire, ACH, credit card via Stripe, trust transfer). Auto-update invoice status on payment.

**Design:**
```typescript
app.post('/api/invoices/:invoiceId/payments', {
  schema: {
    body: {
      type: 'object',
      required: ['amount', 'paymentMethod', 'paymentDate'],
      properties: {
        amount: { type: 'number', exclusiveMinimum: 0 },
        paymentMethod: { enum: ['check', 'wire', 'ach', 'credit_card', 'trust_transfer'] },
        paymentDate: { type: 'string', format: 'date' },
        reference: { type: 'string' },
        stripePaymentIntentId: { type: 'string' }
      }
    }
  }
}, async (req, reply) => {
  // Record payment
  // Update invoice: amount_paid += payment.amount, balance_due recalculated
  // If balance_due <= 0: status = 'paid'
  // If balance_due > 0 and amount_paid > 0: status = 'partial'
  // If paymentMethod === 'trust_transfer': create trust_transaction
});
```

**Testing:**
- Partial payment: $5,000 on $10,000 invoice → status `partial`, balance_due $5,000
- Full payment: remaining $5,000 → status `paid`, balance_due $0
- Overpayment → 400 (amount exceeds balance_due)
- Trust transfer payment → trust transaction created, client ledger balance decremented
- Payment with Stripe: verify payment intent ID stored, no card data stored (PCI compliance)

### Task 4.5: Billing Reports & UI

**What:** Build the invoice list, invoice detail/editor, LEDES export download, and billing realisation reports (billed vs. collected).

**Design:**
```typescript
// apps/web/src/app/(dashboard)/billing/page.tsx
// Invoice list with status badges, filters by status/date/matter/client
// Click invoice → detail page with line items, payments, LEDES download button

// apps/api/src/modules/reporting/billing-reports.ts
app.get('/api/reports/billing-realisation', {
  schema: {
    querystring: {
      dateFrom: { type: 'string', format: 'date' },
      dateTo: { type: 'string', format: 'date' },
      groupBy: { enum: ['attorney', 'matter', 'practice_group', 'month'] }
    }
  }
});
// Returns: { group, total_billed, total_collected, realisation_rate }
```

**Testing:**
- Invoice list shows correct status badges (colour-coded)
- LEDES XML download produces valid file with correct Content-Type header
- Billing realisation report: total_collected / total_billed = realisation_rate (verified with known data)
- Report grouped by attorney shows per-attorney billing performance
- Date range filter correctly bounds the report

### Definition of Done — Phase 4
- [ ] Invoice generation from approved time entries with correct totals
- [ ] LEDES XML 2.2 and 1998B export producing standards-compliant output
- [ ] Invoice lifecycle state machine with role-based transitions
- [ ] Payment recording with auto-status updates and trust transfer support
- [ ] Billing realisation reports by attorney, matter, and practice group
- [ ] Frontend: invoice list, invoice detail, LEDES download, payment recording form
- [ ] Financial calculation tests: every arithmetic operation verified to 2 decimal places with no floating-point errors (use Prisma Decimal / string-based arithmetic)

---

## Phase 5: Trust Accounting

**Duration:** 4 weeks
**Dependencies:** Phase 4
**Goal:** Full IOLTA trust accounting with three-way reconciliation per ABA Model Rule 1.15. This phase has the highest compliance risk and requires the most rigorous testing.

### Task 5.1: Trust Account Management

**What:** Create and manage trust accounts with jurisdiction-specific IOLTA rules stored in JSONB. Support multiple trust accounts per tenant.

**Design:**
```typescript
app.post('/api/trust-accounts', {
  schema: {
    body: {
      type: 'object',
      required: ['accountName', 'bankName', 'accountNumberLast4', 'jurisdictionCountry'],
      properties: {
        accountName: { type: 'string' },
        bankName: { type: 'string' },
        accountNumberLast4: { type: 'string', pattern: '^[0-9]{4}$' },
        jurisdictionCountry: { type: 'string', minLength: 2, maxLength: 2 },
        jurisdictionSubdivision: { type: 'string' },
        isIolta: { type: 'boolean', default: true },
        jurisdictionRules: { type: 'object' }
      }
    }
  }
});
```

**Testing:**
- Create trust account → 201 with account_number_last4 stored (never full account number)
- Jurisdiction rules JSONB validated (reconciliation_frequency required)
- Trust account marked inactive → no new transactions allowed
- Trust accounts are tenant-scoped (RLS)

### Task 5.2: Client Ledgers & Transactions

**What:** Manage per-client ledgers within trust accounts. Record deposits, disbursements, and transfers with running balance calculation. Enforce that disbursements cannot exceed client ledger balance.

**Design:**
```typescript
app.post('/api/trust-accounts/:accountId/transactions', {
  schema: {
    body: {
      type: 'object',
      required: ['clientLedgerId', 'transactionType', 'amount', 'description', 'transactionDate'],
      properties: {
        clientLedgerId: { type: 'string', format: 'uuid' },
        transactionType: { enum: ['deposit', 'disbursement', 'transfer_in', 'transfer_out', 'interest', 'bank_fee'] },
        amount: { type: 'number', exclusiveMinimum: 0 },
        description: { type: 'string' },
        transactionDate: { type: 'string', format: 'date' },
        reference: { type: 'string' },
        paymentId: { type: 'string', format: 'uuid' }
      }
    }
  }
}, async (req, reply) => {
  // Begin transaction
  // For disbursements: verify client ledger balance >= amount
  // Calculate running_balance
  // Update client_ledger balance
  // Update trust_account current_balance
  // Audit log
  // Commit
});
```

**Testing:**
- Deposit $10,000 → client ledger balance = $10,000, trust account balance incremented
- Disbursement $3,000 → client ledger balance = $7,000, running_balance correct
- Disbursement exceeding client ledger balance → 400 ("Insufficient funds in client trust ledger")
- Transfer between client ledgers → source decremented, destination incremented, trust account total unchanged
- Concurrent deposits to same ledger → no race condition (serialisable isolation or advisory lock)
- Running balance for each transaction is monotonically consistent with prior transactions
- Trust transactions are immutable: UPDATE and DELETE blocked by database policy

### Task 5.3: Three-Way Reconciliation

**What:** Implement monthly three-way reconciliation: bank statement balance vs. book balance (trust account ledger) vs. sum of all client ledger balances. Flag discrepancies.

**Design:**
```typescript
app.post('/api/trust-accounts/:accountId/reconcile', {
  schema: {
    body: {
      type: 'object',
      required: ['reconciliationDate', 'bankStatementBalance'],
      properties: {
        reconciliationDate: { type: 'string', format: 'date' },
        bankStatementBalance: { type: 'number' },
        outstandingDeposits: { type: 'number', default: 0 },
        outstandingChecks: { type: 'number', default: 0 },
        notes: { type: 'string' }
      }
    }
  }
}, async (req, reply) => {
  const bookBalance = trustAccount.currentBalance;
  const clientLedgerTotal = await sumClientLedgerBalances(accountId);
  const adjustedBankBalance = bankStatementBalance + outstandingDeposits - outstandingChecks;
  const isBalanced = (adjustedBankBalance === bookBalance) && (bookBalance === clientLedgerTotal);
  const discrepancy = Math.abs(adjustedBankBalance - bookBalance) + Math.abs(bookBalance - clientLedgerTotal);
  // Create reconciliation record
});
```

**Testing:**
- Balanced reconciliation: bank = book = client_ledger_total → `is_balanced` = true, discrepancy = 0
- Discrepancy: bank $100,050 vs book $100,000 → `is_balanced` = false, discrepancy = $50
- Outstanding deposits/checks adjust bank balance correctly
- Reconciliation date must be end-of-month or later (jurisdiction rule enforced)
- Reconciliation record is immutable after creation
- Cannot reconcile for a period already reconciled

### Task 5.4: Trust Accounting UI

**What:** Build trust account dashboard, client ledger view, transaction entry form, and reconciliation wizard.

**Testing:**
- Trust dashboard shows all accounts with current balances
- Client ledger view shows transaction history with running balances
- Reconciliation wizard: enter bank statement balance → shows comparison → flags discrepancy if any
- Transaction entry form validates amount > 0 and prevents disbursement exceeding balance (client-side validation mirrors server-side)

### Definition of Done — Phase 5
- [ ] Trust account CRUD with jurisdiction-specific IOLTA rules
- [ ] Client ledger management with deposit, disbursement, and transfer transactions
- [ ] Running balance calculation correct for all transaction types
- [ ] Disbursement overdraft prevention enforced at database level
- [ ] Trust transactions are immutable (no UPDATE/DELETE)
- [ ] Three-way reconciliation producing correct balanced/unbalanced determination
- [ ] Trust-to-invoice payment flow: disbursement from trust → payment on invoice
- [ ] All financial calculations tested with Decimal arithmetic (no floating-point)
- [ ] Frontend: trust dashboard, client ledger, transaction entry, reconciliation wizard

---

## Phase 6: Client Communication Portal

**Duration:** 3 weeks
**Dependencies:** Phase 2, Phase 4
**Goal:** Clients can log in to a secure portal to view matter status, documents, invoices, and exchange messages with their attorney. This is a separate authenticated surface (portal auth distinct from firm auth).

### Task 6.1: Portal Authentication

**What:** Create a separate client authentication system. Attorneys invite clients via email; clients set a password and log in to the portal. Portal sessions are scoped to a specific client contact and their matters.

**Design:**
```typescript
// apps/api/src/modules/portal/auth-routes.ts
app.post('/api/portal/invite', { /* attorney invites client by contact_id and email */ });
app.post('/api/portal/accept-invite', { /* client sets password via invite token */ });
app.post('/api/portal/login', { /* email/password → portal session */ });
// Portal session contains: tenantId, contactId, matterId[]
```

**Testing:**
- Attorney invites client → invitation email sent with secure token (expires in 7 days)
- Client accepts invite → portal account created, password set
- Portal login → session created with scope limited to client's matters
- Portal user cannot access other clients' matters
- Invitation token single-use (cannot be reused after acceptance)

### Task 6.2: Portal Matter View

**What:** Clients see a read-only view of their matter status, pipeline stage, upcoming deadlines, and linked documents.

**Testing:**
- Client sees only matters where they are listed as a contact with role `client`
- Matter status, pipeline stage, and key dates displayed
- Document list shows only documents the attorney has marked as client-visible
- Client cannot modify any matter data

### Task 6.3: Portal Messaging

**What:** Bidirectional secure messaging between client and attorney within the portal. Messages are linked to specific matters.

**Testing:**
- Client sends message → attorney receives notification
- Attorney responds → client sees response in portal
- Messages are linked to correct matter
- Messages are stored with sender identification (user_id for attorney, contact_id for client)

### Task 6.4: Portal Invoice View & Payment

**What:** Clients view their invoices and pay online via Stripe checkout session.

**Testing:**
- Client sees invoices for their matters with status and balance due
- Click "Pay" → Stripe checkout session opens → on success, payment recorded and invoice status updated
- Client cannot see invoice line item details marked as confidential (if implemented)
- Payment receipt sent via email

### Definition of Done — Phase 6
- [ ] Portal authentication with email invitation flow
- [ ] Client-scoped matter view (read-only)
- [ ] Bidirectional messaging linked to matters
- [ ] Invoice viewing and online payment via Stripe
- [ ] Portal is a separate route group with distinct auth middleware
- [ ] Client can view only their matters and documents

---

## Phase 7: Conflict Checking & Graph Layer

**Duration:** 4 weeks
**Dependencies:** Phase 2
**Goal:** Implement the graph layer from Data Model Suggestion 4 for comprehensive conflict-of-interest checking. This transforms conflict checking from name-matching to structural relationship traversal.

### Task 7.1: Graph Schema & Sync

**What:** Add `graph_nodes` and `graph_edges` tables. Create triggers that automatically create/update graph nodes when contacts, matters, and users are created/updated.

**Design:**
```sql
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    entity_id       UUID,
    entity_table    VARCHAR(50),
    label           VARCHAR(500) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    from_node_id    UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    to_node_id      UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    valid_from      DATE,
    valid_to        DATE,
    weight          NUMERIC(5,2) DEFAULT 1.0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing:**
- Creating a contact auto-creates a corresponding graph_node via trigger
- Updating a contact's name updates the graph_node label
- Assigning a contact to a matter creates a graph_edge with the appropriate edge_type
- Graph nodes and edges are tenant-scoped
- Backfill script: running against existing data correctly populates all graph nodes and edges

### Task 7.2: Conflict Check Engine

**What:** Implement graph-traversal conflict checking using recursive CTEs. Given a potential new client, traverse 3 hops through corporate hierarchies and check for adverse relationships in existing matters.

**Design:**
```typescript
// apps/api/src/modules/conflicts/engine.ts
export async function runConflictCheck(tenantId: string, contactId: string, maxDepth: number = 3): Promise<ConflictResult[]> {
  // Execute recursive CTE from Data Model 4
  // Return: { entity, depth, relationship_chain, conflicting_matter, adverse_role }
}

app.post('/api/conflict-checks', {
  schema: {
    body: {
      type: 'object',
      required: ['contactId'],
      properties: {
        contactId: { type: 'string', format: 'uuid' },
        matterId: { type: 'string', format: 'uuid' },
        maxDepth: { type: 'integer', minimum: 1, maximum: 5, default: 3 }
      }
    }
  }
});
```

**Testing:**
- Direct conflict: potential client is opposing_party in existing matter → detected at depth 0
- Corporate hierarchy conflict: potential client's parent company is adverse in a matter → detected at depth 1
- Officer conflict: potential client is an officer of a company that is adverse → detected at depth 2
- No conflict: unrelated entity → clear result
- Cycle detection: A is subsidiary of B, B is parent of A → no infinite loop
- Performance: conflict check on a graph with 10,000 nodes completes in < 2 seconds
- Conflict check results stored with full relationship chain for review

### Task 7.3: Conflict Check UI & Workflow

**What:** Build the conflict check interface: search for entity, run check, review results, record outcome (clear / potential_conflict / conflict_found / waiver_obtained).

**Testing:**
- Run conflict check from the "New Matter" flow before matter creation
- Results displayed with visual relationship chain ("Client A → subsidiary_of → Corp B → opposing_party_in → Matter X")
- Attorney can mark result as clear or flag as conflict
- Conflict check history stored for compliance/audit
- Waiver workflow: record that conflict was acknowledged and waived (with notes)

### Definition of Done — Phase 7
- [ ] Graph nodes and edges tables created with sync triggers
- [ ] Backfill script populates graph from existing relational data
- [ ] Conflict check engine detects direct, corporate hierarchy, and officer conflicts
- [ ] Recursive CTE handles cycles without infinite loops
- [ ] Conflict check integrated into "New Matter" creation flow
- [ ] Conflict check results persisted with full relationship chain
- [ ] Frontend: conflict check UI with visual relationship chain
- [ ] Performance: < 2 seconds for 10,000-node graph

---

## Phase 8: AI-Assisted Workflows

**Duration:** 5 weeks
**Dependencies:** Phases 2, 3, 4
**Goal:** Implement the three core AI features that differentiate this platform: automated matter intake, passive time capture suggestions, and predictive matter budgeting.

### Task 8.1: AI Infrastructure & MCP Server

**What:** Set up the Anthropic SDK integration, prompt caching for repetitive operations, and an MCP server that exposes matter data to AI agents.

**Design:**
```typescript
// apps/api/src/modules/ai/client.ts
import Anthropic from '@anthropic-ai/sdk';
const client = new Anthropic();

// apps/api/src/mcp/server.ts
// MCP server exposing:
// - matters.list, matters.get
// - contacts.search
// - time_entries.list
// - invoices.list
// - calendar_events.list
```

**Testing:**
- Anthropic SDK initialises with API key from environment
- MCP server starts and responds to capability discovery
- MCP tools return tenant-scoped data (respects authentication context)
- AI features gracefully degrade when Anthropic API is unavailable (return 503 with "AI features temporarily unavailable")

### Task 8.2: AI Matter Intake

**What:** Accept intake form submissions (text, email body, or document upload), use Claude to auto-classify matter type (SALI), suggest attorney assignment based on practice area, estimate key dates, and draft an engagement letter.

**Design:**
```typescript
app.post('/api/ai/intake', {
  schema: {
    body: {
      type: 'object',
      required: ['content'],
      properties: {
        content: { type: 'string' },           // intake form text or email body
        documentIds: { type: 'array', items: { type: 'string', format: 'uuid' } },
        clientContactId: { type: 'string', format: 'uuid' }
      }
    }
  }
}, async (req, reply) => {
  const result = await client.messages.create({
    model: 'claude-sonnet-4-20250514',
    system: INTAKE_SYSTEM_PROMPT,  // includes SALI taxonomy, firm's practice areas, attorney list
    messages: [{ role: 'user', content: req.body.content }],
    // Enable prompt caching for system prompt (reused across all intakes)
  });
  // Parse structured output: matter_type, area_of_law, suggested_attorney, key_dates, engagement_letter_draft
  // Return as suggestions (not auto-applied)
});
```

**Testing:**
- Personal injury intake text → SALI classification "Personal Injury > Motor Vehicle Accident"
- Suggested attorney matches practice area (PI attorneys suggested for PI cases)
- Key dates extracted: incident date, statute of limitations estimated
- Engagement letter draft contains client name, matter description, fee structure
- Ambiguous intake → multiple classification suggestions with confidence scores
- Non-legal content → response indicates "Unable to classify" (no hallucinated classification)

### Task 8.3: Passive Time Capture Suggestions

**What:** Analyse calendar events, email metadata (subject/timestamp/recipients, not body), and document edit timestamps to generate draft time entries for attorney review.

**Design:**
```typescript
app.post('/api/ai/time-suggestions', {
  schema: {
    body: {
      type: 'object',
      required: ['userId', 'date'],
      properties: {
        userId: { type: 'string', format: 'uuid' },
        date: { type: 'string', format: 'date' }
      }
    }
  }
}, async (req, reply) => {
  // Gather: calendar events, document edits, communication log for the day
  // Send to Claude with matter context
  // Return: array of suggested time entries with matter, duration, description, UTBMS codes
  // Each suggestion includes confidence score and source references
});
```

**Testing:**
- Calendar event "Deposition of John Smith re: Matter 2026-00142" → time entry suggestion: 2.0 hours, L210/A107, correct matter
- Document edited for 45 minutes → time entry suggestion with duration and document reference
- Suggestion includes source_ref linking back to calendar event or document
- Suggestions have confidence score (0.0-1.0)
- Attorney can accept, modify, or reject each suggestion
- No suggestions generated for matters not assigned to the attorney

### Task 8.4: Predictive Matter Budgeting

**What:** Given a new matter's characteristics (type, jurisdiction, opposing counsel), analyse historical matter data to produce probabilistic cost ranges.

**Design:**
```typescript
app.post('/api/ai/budget-prediction', {
  schema: {
    body: {
      type: 'object',
      required: ['matterId'],
      properties: { matterId: { type: 'string', format: 'uuid' } }
    }
  }
}, async (req, reply) => {
  // Query historical matters with similar: sali_classification, jurisdiction, billing_method
  // Send aggregated data (totals, durations, phases) to Claude
  // Return: { p25: amount, p50: amount, p75: amount, factors: string[], comparables: matterId[] }
});
```

**Testing:**
- New PI matter in California → budget prediction based on historical CA PI matters
- Result includes percentile ranges (p25, p50, p75) and contributing factors
- Fewer than 5 comparable matters → response indicates "Insufficient data for reliable prediction"
- Prediction does not expose confidential data from comparable matters (only aggregated statistics)

### Definition of Done — Phase 8
- [ ] Anthropic SDK integrated with prompt caching
- [ ] MCP server exposing matter data to AI agents
- [ ] AI intake: auto-classification, attorney suggestion, engagement letter draft
- [ ] Passive time capture: calendar/document-based time entry suggestions
- [ ] Predictive budgeting: probabilistic cost ranges from historical data
- [ ] All AI features are optional — system functions fully without AI
- [ ] AI suggestions clearly marked as drafts requiring human review
- [ ] Graceful degradation when Anthropic API is unavailable

---

## Phase 9: Mobile Application

**Duration:** 4 weeks
**Dependencies:** Phases 2, 3, 6
**Goal:** React Native mobile app for iOS and Android covering the most common mobile use cases: time entry, timer management, matter lookup, and client portal messaging.

### Task 9.1: React Native Setup & Authentication

**What:** Initialise React Native app with Expo, configure authentication using the existing API session endpoints, and implement biometric login (Face ID / fingerprint).

**Testing:**
- Login via email/password succeeds on iOS and Android
- Session token stored in Keychain (iOS) / Keystore (Android)
- Biometric unlock re-authenticates without re-entering credentials
- Logout clears all local credentials

### Task 9.2: Mobile Time Entry & Timers

**What:** Full timer management (start/pause/stop) and time entry creation from mobile. Timer continues running even when app is backgrounded.

**Testing:**
- Start timer → timer runs in background → open app → timer shows correct elapsed time
- Stop timer → time entry form pre-filled → submit → entry appears in web dashboard
- Time entry form includes matter selector and UTBMS code picker
- Offline support: time entries queued locally when offline, synced when reconnected

### Task 9.3: Mobile Matter Lookup & Portal

**What:** Matter list, matter detail view, and client portal messaging on mobile.

**Testing:**
- Matter list loads with pull-to-refresh
- Matter detail shows contacts, upcoming tasks, and recent activity
- Portal messages can be read and sent from mobile
- Push notifications for new portal messages

### Definition of Done — Phase 9
- [ ] React Native app builds and runs on iOS and Android
- [ ] Authentication with biometric unlock
- [ ] Timer management with background execution
- [ ] Time entry creation with offline queue
- [ ] Matter lookup and detail view
- [ ] Client portal messaging
- [ ] Push notifications for portal messages and overdue task reminders

---

## Phase 10: Integrations & Calendar

**Duration:** 4 weeks
**Dependencies:** Phase 4
**Goal:** Connect to external systems: accounting (QuickBooks, Xero), e-signature (DocuSign), calendar (Google Calendar, Outlook), and court systems (CourtListener).

### Task 10.1: Accounting Sync (QuickBooks / Xero)

**What:** Bidirectional sync of invoices and payments with QuickBooks Online and Xero. Invoices created in the matter management system are pushed to the accounting platform; payments recorded in the accounting platform are pulled back.

**Design:**
```typescript
// apps/api/src/modules/integrations/quickbooks/sync.ts
export async function pushInvoiceToQuickBooks(invoice: Invoice, qbClient: QuickBooksClient): Promise<void> {
  // Map invoice fields to QuickBooks Invoice object
  // Map line items to QuickBooks SalesItemLineDetail
  // Push via QuickBooks API
}

export async function pullPaymentsFromQuickBooks(qbClient: QuickBooksClient, since: Date): Promise<Payment[]> {
  // Query QuickBooks for payments received since last sync
  // Match to invoices by QuickBooks invoice ID
  // Return list of new payments to record
}
```

**Testing:**
- Invoice pushed to QuickBooks → appears in QuickBooks dashboard with correct amounts
- Payment recorded in QuickBooks → pulled and recorded against correct invoice in MMS
- Sync handles rate limiting (QuickBooks API: 500 requests/minute)
- Duplicate detection: same invoice not pushed twice
- OAuth token refresh on expiry

### Task 10.2: Calendar Integration (Google / Outlook)

**What:** Two-way sync of calendar events between the matter management system and Google Calendar / Microsoft Outlook. Court deadlines appear in attorney's external calendar; external events can be linked to matters.

**Testing:**
- Court hearing event created in MMS → appears in attorney's Google Calendar
- Event created in Outlook → available for linking to a matter in MMS
- Event updated in either system → reflected in the other
- Deleted event sync handled correctly
- Calendar sync permissions: only events for matters the attorney is assigned to

### Task 10.3: E-Signature (DocuSign)

**What:** Send documents from within a matter for e-signature via DocuSign. Signed documents are automatically uploaded back to the matter's document store.

**Testing:**
- Select document, specify signers → envelope sent via DocuSign API
- DocuSign webhook on completion → signed PDF downloaded and stored as new document version
- Signing status tracked on document record

### Task 10.4: Court System Integration (CourtListener)

**What:** Import court deadlines and docket entries from CourtListener API based on the matter's case number. Monitor for new docket entries.

**Testing:**
- Enter federal case number → docket entries imported as calendar events
- New docket entry detected via polling → calendar event created → attorney notified
- Court opinion linked to matter as document

### Definition of Done — Phase 10
- [ ] QuickBooks and Xero two-way invoice/payment sync
- [ ] Google Calendar and Outlook two-way event sync
- [ ] DocuSign e-signature with signed document return
- [ ] CourtListener docket import and monitoring
- [ ] OAuth 2.0 token management for all third-party integrations
- [ ] Integration health dashboard showing sync status and errors

---

## Phase 11: Document Management Enhancement

**Duration:** 3 weeks
**Dependencies:** Phase 2
**Goal:** Full-text search, document preview, version control UI, and OCR for scanned documents.

### Task 11.1: Full-Text Search with PostgreSQL tsvector

**What:** Index document content (text-based files) and metadata using PostgreSQL full-text search. Support search across all documents in a matter or across the entire tenant.

**Design:**
```sql
ALTER TABLE documents ADD COLUMN search_vector tsvector;
CREATE INDEX idx_documents_fts ON documents USING GIN(search_vector);

CREATE OR REPLACE FUNCTION update_document_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector := to_tsvector('english',
    COALESCE(NEW.title, '') || ' ' ||
    COALESCE(NEW.file_name, '') || ' ' ||
    COALESCE(NEW.metadata->>'description', '') || ' ' ||
    COALESCE(NEW.metadata->>'ai_summary', '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Testing:**
- Search for "motion dismiss" returns documents with those terms in title, filename, or summary
- Search scoped to a single matter returns only that matter's documents
- Search ranking: title matches rank higher than description matches
- Tenant isolation enforced on search results

### Task 11.2: Document Preview & OCR

**What:** Server-side PDF preview generation (first page thumbnail). OCR for scanned PDFs using Tesseract, with extracted text stored in document metadata for search indexing.

**Testing:**
- Upload PDF → thumbnail generated and stored
- Upload scanned PDF → OCR extracts text → text stored in metadata → searchable
- OCR runs as async job (does not block upload response)
- Non-PDF files: show file type icon instead of preview

### Task 11.3: Version Control UI

**What:** Document version history showing who uploaded each version and when. Side-by-side diff view for text documents.

**Testing:**
- Upload new version → previous version marked `is_current_version = false`
- Version history shows all versions with uploader and timestamp
- Clicking a historical version downloads that specific version
- Current version is always the latest uploaded

### Definition of Done — Phase 11
- [ ] Full-text search across documents with ranking
- [ ] PDF preview thumbnails generated on upload
- [ ] OCR for scanned documents with text extraction
- [ ] Version control UI with history and specific version download
- [ ] Document tagging and category filtering

---

## Phase 12: Production Hardening & Compliance

**Duration:** 4 weeks
**Dependencies:** All previous phases
**Goal:** Prepare the system for production deployment with security hardening, performance optimisation, compliance documentation, and self-hosted deployment packaging.

### Task 12.1: Security Hardening

**What:** Security audit, penetration testing remediation, rate limiting, CORS policy tightening, and Content Security Policy headers.

**Testing:**
- OWASP Top 10 checklist: all items addressed
- SQL injection: parameterised queries verified across all endpoints
- XSS: Content Security Policy headers set, user input sanitised
- Rate limiting: 100 requests/minute per user, 10 login attempts per 5 minutes
- CORS: only allowed origins can make API requests
- Helmet.js-equivalent headers set on all responses

### Task 12.2: Performance Optimisation

**What:** Database query analysis (EXPLAIN ANALYZE on all common queries), connection pooling (PgBouncer), response caching (Redis), and CDN for static assets.

**Testing:**
- All common queries execute in < 100ms (verified via EXPLAIN ANALYZE)
- Matter list page loads in < 500ms with 10,000 matters
- Connection pool handles 100 concurrent users without exhaustion
- Redis cache hit rate > 80% for reference data (SALI, UTBMS codes)

### Task 12.3: SOC 2 Control Documentation

**What:** Document security controls aligned with SOC 2 Type II Trust Services Criteria. Not a formal certification, but provides the control documentation that enterprise clients will request.

**Testing:**
- Control documentation covers: access control, encryption (at rest and in transit), audit logging, change management, incident response
- Every control has evidence: configuration files, test results, or policy documents
- Security guide published in `/docs/security/`

### Task 12.4: Self-Hosted Deployment Package

**What:** Docker Compose production configuration, Helm chart for Kubernetes, environment variable documentation, and backup/restore procedures.

**Design:**
```yaml
# docker/docker-compose.prod.yml
services:
  api:
    image: ghcr.io/matter-mgmt/api:latest
    environment:
      DATABASE_URL: postgres://...
      SESSION_SECRET: ...
      S3_ENDPOINT: http://minio:9000
  web:
    image: ghcr.io/matter-mgmt/web:latest
  postgres:
    image: postgres:16
    volumes: ['pgdata:/var/lib/postgresql/data']
  minio:
    image: minio/minio
    volumes: ['s3data:/data']
  redis:
    image: redis:7-alpine
```

**Testing:**
- `docker compose -f docker-compose.prod.yml up` starts all services successfully
- Fresh installation: run migrations, seed reference data, create admin user
- Backup procedure: `pg_dump` produces restorable backup
- Restore procedure: backup restores to a clean instance with all data intact
- Upgrade procedure: `docker compose pull && docker compose up -d` applies new images without data loss

### Definition of Done — Phase 12
- [ ] Security audit completed with all critical/high findings resolved
- [ ] All common queries < 100ms, page loads < 500ms
- [ ] SOC 2 control documentation published
- [ ] Docker Compose production configuration tested and documented
- [ ] Kubernetes Helm chart available
- [ ] Backup and restore procedures documented and tested
- [ ] GDPR data export and deletion endpoints operational
- [ ] OpenAPI 3.1 spec published at `/docs/api/`
- [ ] CONTRIBUTING.md and deployment guide written
- [ ] Licence determined and applied (Apache 2.0 recommended for OSS core)

---

## Summary

| Phase | Name | Duration | Dependencies | Key Deliverables |
|-------|------|----------|--------------|------------------|
| 1 | Foundation & Infrastructure | 4 weeks | None | Monorepo, DB, auth, multi-tenancy, CI/CD |
| 2 | Matter & Contact Management | 4 weeks | Phase 1 | Contacts, matters, SALI, templates, kanban |
| 3 | Time Tracking | 3 weeks | Phase 2 | Time entries, timers, UTBMS, approval |
| 4 | Billing & Invoicing | 5 weeks | Phase 3 | Invoices, LEDES export, payments, reports |
| 5 | Trust Accounting | 4 weeks | Phase 4 | IOLTA trust, three-way reconciliation |
| 6 | Client Communication Portal | 3 weeks | Phase 2, 4 | Portal auth, messaging, invoice payment |
| 7 | Conflict Checking & Graph | 4 weeks | Phase 2 | Graph layer, conflict engine, traversal |
| 8 | AI-Assisted Workflows | 5 weeks | Phases 2, 3, 4 | AI intake, time suggestions, budgeting |
| 9 | Mobile Application | 4 weeks | Phases 2, 3, 6 | React Native, timers, portal messaging |
| 10 | Integrations & Calendar | 4 weeks | Phase 4 | QuickBooks, Xero, DocuSign, CourtListener |
| 11 | Document Management Enhancement | 3 weeks | Phase 2 | Full-text search, OCR, versioning |
| 12 | Production Hardening | 4 weeks | All | Security, performance, compliance, deployment |

**Total estimated duration:** 47 weeks (critical path: Phases 1-2-3-4-5 = 20 weeks; remaining phases parallelisable)

**Critical path to revenue (MVP):** Phases 1 through 5 (20 weeks) deliver a functional matter management system with billing, trust accounting, and LEDES export — sufficient for pilot deployments with small law firms.
