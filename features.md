# Matter Management System — Feature & Functionality Survey

> Candidate #93 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Clio Manage | Cloud practice management | Commercial SaaS | https://clio.com |
| MyCase | All-in-one matter management | Commercial SaaS | https://mycase.com |
| Filevine | Litigation-focused matter management | Commercial SaaS | https://filevine.com |
| PracticePanther | Solo / small firm practice management | Commercial SaaS | https://practicepanther.com |
| Litify | Salesforce-native enterprise matter management | Commercial SaaS | https://litify.com |
| Smokeball | Matter management with automatic time capture | Commercial SaaS | https://smokeball.com |
| ArkCase | Open-source, FedRAMP-authorised case management | Open Source + paid support | https://arkcase.com |
| MyLegalNet | Open-source case management with court API | Open Source | https://mylegalnet.com |
| RunSensible | Mid-market matter management + CRM | Commercial SaaS | https://runsensible.com |
| Centerbase | Mid-market with native accounting | Commercial SaaS | https://centerbase.com |

## Feature Analysis by Solution

### Clio Manage

**Core features**
- Matter hub: each matter is a central workspace linking documents, time entries, billing, tasks, notes, contacts, and calendar events
- Matter templates: pre-configure standard task lists, document checklists, and deadline sequences for common case types (e.g., "PI Matter" auto-creates standard tasks and deadlines on matter opening)
- Time tracking with multiple timers, mobile entry, and rounding rules
- Billing: LEDES-compatible invoice generation, trust accounting, online payment collection
- Client portal: secure two-way communication, document sharing, and invoice presentation
- Kanban-style pipeline view for tracking matters across workflow stages by practice area

**Differentiating features**
- Dominant market position with 150,000+ law firms; largest legal practice management ecosystem (250+ integrations)
- Clio Duo: AI assistant (included on Advanced/Complete plans) that answers natural-language questions about matter data — e.g., "What matters are due this week for partner X?"
- Clio acquired vLex for $1 billion (June 2025): integrates a 1-billion-document global legal research library directly into practice management — a unique combination not available in any other platform
- Largest third-party integration marketplace in legal practice management

**UX patterns**
- Attorney-facing matter dashboard: intuitive navigation between linked matter records (contacts, documents, billing, tasks)
- Mobile app with time entry, document access, and client messaging
- Kanban board for visual pipeline management by practice area or case stage
- Client-facing portal for secure messaging, document review, and online invoice payment

**Integration points**
- Native integrations: QuickBooks, Xero, GSuite, Microsoft 365, Dropbox, Box, DocuSign, Zapier
- Clio App Marketplace: 250+ verified integrations across legal research, accounting, court filing, e-sign, and communication tools
- REST API for custom enterprise integrations
- vLex integration: legal research within the Clio UI post-acquisition

**Known gaps**
- Add-on pricing creep: Advanced features (court rules, advanced reporting, Clio Duo AI) require higher pricing tiers; a 5-attorney firm can spend $13,000–$15,000/year
- Weaker for complex enterprise litigation or corporate matter portfolios requiring sophisticated workflow customisation
- International billing and trust accounting compliance is improving but still primarily US/Canada/Australia focused

**Licence / IP notes**
- Fully proprietary; total funding exceeds $1B; valued at approximately $3B (2025)
- Clio Duo AI, matter template engine, and vLex integration are core commercial IP

---

### Filevine

**Core features**
- Highly customisable matter workflows: configurable stages, fields, task sequences, and document automation by practice area
- Supports 100,000+ users and 1.8 billion documents managed
- Integrated time tracking across Filevine, Outlook, Gmail, Word, and mobile — start/stop timers from any context
- Billing and payment processing (Filevine Payments): end-to-end timekeeping → invoice generation → payment collection in one platform
- Document assembly: generates polished documents automatically from matter data fields
- AI features: document summarisation, intake automation, and matter analytics

**Differentiating features**
- The most customisable workflow engine in the legal practice management category; suitable for complex litigation practices with non-standard case management needs
- Full timekeeping-to-payment platform in a single product — the only major platform claiming complete end-to-end billing and payment without third-party integration
- Mass tort and plaintiffs' firm capabilities at scale (100K+ user deployments)

**UX patterns**
- Configurable stage-based matter pipeline with custom fields per practice area
- Attorney and staff-specific dashboards showing assigned tasks, upcoming deadlines, and matter activity
- Client-facing portal for document sharing and status updates

**Integration points**
- Outlook and Gmail email integration with matter-linked correspondence logging
- Word plugin for document assembly and time capture from Office
- DocuSign and Adobe Sign for e-signature
- Settlement management and medical records tools via partner integrations

**Known gaps**
- Steep learning curve; configuration complexity can be a barrier for small firms
- Custom enterprise pricing with no published rates makes budget planning difficult
- AI features are still developing relative to Clio's Duo assistant

**Licence / IP notes**
- Fully proprietary; raised $108M Series D (2022)
- Workflow customisation engine and document assembly module are core commercial IP

---

### Smokeball

**Core features**
- Automatic time capture: records time spent in email (Outlook), documents (Word), and within Smokeball without manual timer activation — unique automatic capture capability
- AI-powered email time tracking: automatically logs time spent processing emails in Outlook
- Matter management and billing integrated with document management
- Pre-built matter checklists and document templates by law area
- Strong compliance with Australian legal billing requirements alongside US market support

**Differentiating features**
- Automatic time capture without manual timer activation is Smokeball's single most distinctive feature and a strong differentiator in the market — addressing the industry's most common source of billing leakage
- Best-in-class for small US and Australian firms that bill by time and want to eliminate manual timesheet reconstruction

**UX patterns**
- Desktop-integrated experience (deep Outlook and Word integration for automatic capture)
- Matter-centric UI showing all activity, documents, and communications in a single view
- Automated billing summaries generated from captured time events

**Integration points**
- Deep Microsoft Office integration (Outlook, Word) for automatic time capture
- PEXA integration for Australian property conveyancing workflows
- Accounting integrations (QuickBooks, Xero)

**Known gaps**
- Limited scalability for firms above 20–30 attorneys
- Less suitable for complex litigation workflows vs. Filevine or Clio
- Weaker third-party integration ecosystem compared to Clio

**Licence / IP notes**
- Fully proprietary; automatic time capture methodology is a core trade secret / commercial differentiator
- No open-source components

---

### ArkCase

**Core features**
- Open-source case management platform with FedRAMP authorisation (rare for an OSS legal platform)
- HIPAA-compliant data handling
- Extensible API architecture for custom workflow and form development
- Covers core matter management: case creation, document management, task tracking, and invoice management
- Community edition is fully free; enterprise support tiers available via Armedia (the primary commercial backer)

**Differentiating features**
- The only credibly open-source case management platform with government-grade security certifications (FedRAMP)
- Suitable for government agencies, military, and healthcare-adjacent legal organisations where commercial SaaS vendors cannot meet data residency or security requirements
- Extensible plugin architecture allows custom legal-specific modules to be built and contributed back

**UX patterns**
- Web-based admin interface; not as polished as commercial SaaS alternatives
- Configuration requires technical administration expertise
- Self-hosted deployment with organisation-controlled data residency

**Integration points**
- REST API for custom integrations
- LDAP/Active Directory for identity management
- Custom module development using Spring Boot (Java)

**Known gaps**
- Legal-specific features are sparse relative to commercial platforms: no LEDES billing, no trust accounting, no client portal, no automatic time capture
- Sparse community relative to commercial platforms; implementation requires significant in-house technical capability
- UI quality significantly lags commercial tools

**Licence / IP notes**
- Open source; Apache 2.0 licence — fully permissive for OSS and commercial use
- FedRAMP authorisation is a government-recognised security certification that conveys credibility for public-sector deployments

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Matter creation and management: structured record linking all case information (parties, documents, tasks, time, billing, communications)
- Time tracking: multiple timers, mobile entry, and time-rounding rules; LEDES/UTBMS code assignment
- Invoice generation: LEDES-formatted invoices for corporate clients; standard invoices for individual clients
- Trust / client funds accounting: three-way reconciliation with state bar compliance
- Document management: matter-linked document storage, version control, and search
- Client communication portal: secure messaging, document sharing, and invoice presentation
- Task and deadline management: assigned tasks, court deadline calendaring with jurisdiction-specific rule sets
- SALI/LMSS-compatible matter type classification

### Differentiating Features
- Automatic time capture without manual timer activation (Smokeball — unique in the market)
- AI matter assistant: natural-language query of matter data and proactive status summaries (Clio Duo)
- Integrated legal research within the matter management UI (Clio + vLex post-acquisition)
- Highly customisable workflow stages and fields per practice area (Filevine)
- Predictive matter budgeting from historical matter data — not yet available in any commercial platform
- AI-assisted intake: auto-classify matter type, assign attorney, set deadlines from intake form or email

### Underserved Areas / Opportunities
- Open-source matter management with billing: ArkCase has FedRAMP credibility but no legal-specific billing engine; MyLegalNet has core features but a minimal community — the market for a credible OSS matter management system with LEDES billing is completely open
- Predictive matter budgeting: no commercial platform offers probabilistic cost forecasting from historical matter data; corporate clients and law firms with fixed-fee pricing have no reliable estimation tool
- Client communication automation: client portals remain passive document repositories; proactive AI-drafted status updates are not available in any current platform
- Legal aid and law school deployments: budget-constrained organisations have no suitable matter management tool — ArkCase requires too much technical investment, and commercial tools are unaffordable

### AI-Augmentation Candidates
- Automated matter intake and triage: AI ingests intake forms, emails, and intake documents to auto-classify matter type, assign to the appropriate attorney team, set initial deadlines, and draft the engagement letter — compressing a multi-day process to minutes
- Passive time capture and draft invoice generation: AI captures work performed from calendar, email, document edit history, and call logs, then generates LEDES/UTBMS-coded draft invoices for attorney review — directly addressing the 30–40% billing leakage problem
- Predictive matter budgeting: AI trained on historical matter data (type, court, opposing counsel, judge, outcome) generates probabilistic cost ranges for new matters — enabling fixed-fee pricing and accurate accruals
- Client communication drafting: AI generates draft status update communications for attorney review and client portal delivery, reducing the time attorneys spend on client communication overhead
- Matter risk alerting: AI monitors statute of limitations deadlines, scheduled hearing dates, and outstanding obligations, proactively alerting the responsible attorney before critical dates

## Legal & IP Summary

- LEDES billing format (Legal Electronic Data Exchange Standard) is an open standard — freely implementable in any OSS product; UTBMS codes are similarly open
- SALI Alliance / LMSS (Legal Matter Standard Specification) is an emerging open standard for describing legal matters — implementing SALI compatibility in an OSS matter management system would enable data portability between systems and adoption by corporate legal departments
- ABA Model Rule of Professional Conduct 1.15 (Safekeeping of Client Property) governs trust accounting requirements; any matter management system with a billing module must handle IOLTA trust accounting correctly or disclaim that capability explicitly
- PCI DSS compliance is required if the platform processes client credit card payments in its billing module
- SOC 2 Type II is expected by enterprise law firm clients when evaluating cloud SaaS vendors; an OSS project should document its security controls even without formal certification
- HIPAA compliance is relevant for matter management systems used by healthcare law practices; ArkCase's FedRAMP/HIPAA compliance is a significant differentiator for government and healthcare-sector deployments
- ArkCase is Apache 2.0 licensed — it can be studied, forked, or used as a technical reference without IP restriction

## Recommended Feature Scope

**Must-have (MVP)**:
- Matter creation with configurable fields by matter type and practice area; SALI-compatible matter classification
- Time tracking: multi-timer, mobile entry, LEDES/UTBMS task/activity code assignment, and rounding rules
- Invoice generation: LEDES-formatted for corporate clients; plain invoices for individuals; trust accounting and three-way reconciliation
- Document management: matter-linked storage with version control, tagging, and full-text search
- Task and deadline management: assignable tasks, due dates, and court calendar integration with jurisdiction-specific rule sets
- Client communication portal: secure messaging, document sharing, and online invoice payment
- Contacts management: clients, opposing counsel, courts, and expert witnesses linked to matters
- Basic reporting: matter status, billing realisation, time by matter and attorney

**Should-have (v1.1)**:
- AI matter intake assistant: auto-classify matter type, suggest attorney assignment, and draft engagement letter from intake form input
- Passive time capture suggestions: AI analyses calendar, email, and document activity to propose draft time entries for attorney review
- Kanban pipeline view: visual stage-based matter tracking with drag-and-drop matter progression
- Mobile application: full matter access, time entry, and document viewing on iOS and Android
- Integration connectors: QuickBooks/Xero for accounting sync; DocuSign for e-signature; Google Calendar/Outlook for deadline sync

**Nice-to-have (backlog)**:
- Predictive matter budgeting: AI generates probabilistic cost ranges for new matters based on historical matter data by type, court, and opposing counsel
- Client-facing status update drafting: AI generates draft updates for attorney review and client portal delivery
- LEDES/UTBMS bulk import for migrating historical billing records from incumbent systems
- Conflict check automation: AI-powered conflict-of-interest search across all matter contacts, related parties, and adverse parties
- Legal research integration: API connection to open legal research sources (CourtListener, Public.Resource.Org) or commercial databases for in-matter research linking
