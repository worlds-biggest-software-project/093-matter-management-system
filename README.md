# Matter Management System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source matter management platform for law firms — covering matter tracking, time capture, LEDES billing, and client communication.

Matter Management System is a practice management platform for solo attorneys, small and mid-market law firms, legal aid organisations, and corporate in-house legal teams. It aims to replace expensive proprietary suites such as Clio, Filevine, and Litify with a self-hostable, standards-aligned core that handles matter records, billing, trust accounting, and client portals — augmented with AI for intake, time capture, and budgeting.

---

## Why Matter Management System?

- **Incumbent pricing creep:** Clio's add-on model can cost a 5-attorney firm $13,000–$15,000/year, and enterprise platforms like Filevine and Litify run six figures annually with custom-quote pricing.
- **Open-source gap:** ArkCase has FedRAMP and HIPAA credibility but lacks LEDES billing, trust accounting, and a client portal; MyLegalNet has core features but a minimal community. No credible OSS option covers the full legal billing workflow.
- **Chronic billing leakage:** The industry loses 30–40% of billable time to manual timesheet reconstruction. Only Smokeball offers automatic time capture, and it is proprietary and limited to small firms.
- **Underserved buyers:** Legal aid clinics, law schools, and boutique firms cannot afford commercial SaaS and lack the engineering capacity to deploy ArkCase.
- **Data portability:** All major platforms are proprietary with high switching costs; an SALI/LMSS-compatible OSS data model enables migration between systems.

---

## Key Features

### Matter & Document Management

- Matter records with configurable fields by matter type and practice area, SALI/LMSS-compatible classification
- Matter-linked document storage with version control, tagging, and full-text search
- Contacts management linking clients, opposing counsel, courts, and expert witnesses to matters
- Kanban-style pipeline view for visual stage-based matter tracking
- Matter templates with pre-configured task lists, document checklists, and deadline sequences

### Time Tracking & Billing

- Multi-timer time tracking with mobile entry and configurable rounding rules
- LEDES-formatted invoice generation for corporate clients; standard invoices for individuals
- UTBMS task and activity code assignment
- Trust accounting with three-way reconciliation aligned to ABA Model Rule 1.15
- Online invoice payment with PCI DSS-compliant processing

### Tasks, Deadlines & Communication

- Assignable tasks with due dates and matter-linked correspondence logging
- Court calendar integration with jurisdiction-specific rule sets
- Secure client portal for messaging, document sharing, and invoice presentation
- Reporting on matter status, billing realisation, and time by matter and attorney

### AI-Assisted Workflows

- AI matter intake: auto-classify matter type, suggest attorney assignment, draft engagement letter from intake form input
- Passive time capture suggestions from calendar, email, and document activity
- Predictive matter budgeting from historical matter data (type, court, opposing counsel)
- Draft client status updates for attorney review and portal delivery
- Matter risk alerting on statutes of limitations, hearing dates, and outstanding obligations

---

## AI-Native Advantage

AI is built into the core workflow rather than bolted on as a premium tier. Intake documents and emails are auto-classified into matter records with deadlines and a draft engagement letter; passive time capture from calendar, email, and document edit history generates LEDES/UTBMS-coded draft invoices that target the industry's 30–40% billing leakage. A historical-data model produces probabilistic cost ranges enabling fixed-fee pricing and accurate accruals — a capability not available in any current commercial platform.

---

## Tech Stack & Deployment

- **Standards-aligned:** LEDES billing format, UTBMS codes, SALI/LMSS matter classification
- **Compliance targets:** SOC 2 Type II controls documented; HIPAA-compatible data handling; PCI DSS for payments; IOLTA trust accounting per ABA Model Rule 1.15
- **Deployment:** Self-hosted core (suitable for legal aid, law schools, government, and security-sensitive firms) with an optional hosted SaaS tier
- **Integrations:** REST API, accounting connectors (QuickBooks, Xero), e-signature (DocuSign), and calendar/email sync (Google, Microsoft 365)

---

## Market Context

The legal practice management software market was valued at approximately $3.14 billion in 2026 and is projected to reach $7.8 billion by 2032 at a 12.1% CAGR (Allied Market Research; 360iResearch). North America accounts for roughly 60% of spend. Pricing for commercial alternatives ranges from $39–$219 per user per month for SaaS to $100K–$500K+/year for enterprise platforms like Filevine and Litify; primary buyers are solo and small-firm attorneys, mid-market litigation firms, corporate in-house legal teams, and budget-constrained legal aid operators.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
