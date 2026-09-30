# Notaire SaaS: Discovery and Project Plan

> Status: **draft for validation**. No code has been written yet.
> Decisions so far: **Morocco**, **French + Arabic (RTL)**, **web app**, **multi-office SaaS**, **hosting in Morocco**.
> No notaire advisor and no deed templates yet (see §8b).

---

## 1. Vision

A cloud web application, sold by subscription to Moroccan notary offices (*études notariales*). It lets an office manage its whole activity in one place:
clients → files (*dossiers*) → deeds (*actes*) → signature → formalities → accounting → archiving.

Each office (tenant) has its own data, which is fully isolated from every other office's.

---

## 2. Legal and business context (Morocco)

> ⚠️ Every rate, deadline and rule below must be **checked with a practising notaire** before we build on it. In the app, rates and amounts are **configurable tables**, never hard-coded.

| Topic | What it means for the app |
|---|---|
| **Loi 32-09** (organisation of the notarial profession) | The office must keep a **répertoire** (a register of deeds with sequential numbering) and keep **minutes** (the signed originals). Signed deeds become read-only. |
| **Client funds and the CDG** | Money received for clients (sale price, taxes) goes to the **Caisse de Dépôt et de Gestion (CDG)**, separate from the office's own money. Accounting must keep the two apart. |
| **Droits d'enregistrement** (CGI) | Registration duties on each deed, for example about 4% on real-estate sales, with reduced or exempt cases. There is a registration deadline (about 30 days), so the app needs deadline alerts. The DGI e-filing platform is **SIMPL-Enregistrement**. |
| **Conservation foncière (ANCFCC)** | Land registry fees (about 1.5% plus fixed fees). Titre foncier and réquisition numbers, the *certificat de propriété*, and electronic filing. |
| **TPI** (tax on real-estate gains) | The seller's tax. The notary checks it or collects it before the transfer. The app needs a TPI simulator. |
| **Notary fees (honoraires)** | A regulated fee scale plus **VAT at 10%**. The app needs a configurable fee scale and quotes (*devis* / *provision*). |
| **Loi 43-05** (anti-money-laundering) | Notaries are subject to this law. The app needs KYC checks on clients, a record of the source of funds, a list of suspicious operations, and an export for declarations to the **ANRF**. |
| **Loi 09-08** (personal data, CNDP) | Personal data is protected. The service must be declared to the CNDP, and needs consent, access logs and a retention policy. The **hosting location** is an open question. |
| **Lois 53-05 / 43-20** (electronic signature and trust services) | Electronic signature can come later, through a certified provider. |

---

## 3. Users and roles

| Role | Scope |
|---|---|
| **Super-admin (platform)** | Manages offices, subscriptions and support. Never sees office data without explicit consent. |
| **Notaire titulaire** | Full access to their office. Signs deeds, validates accounting. |
| **Notaire associé / collaborateur** | Same as above, limited to their own files if the office configures it. |
| **Clerc** | Prepares files and drafts deeds. |
| **Secrétaire / accueil** | Clients, appointments, incoming documents. |
| **Comptable** | Accounting, CDG, invoices. Read-only on deeds. |
| **Client (later, portal)** | Follows the progress of their file, uploads documents, pays. |

Permissions are fine-grained (RBAC), and every sensitive action goes into an **audit log**.

---

## 4. Functional modules

### MVP (must-have)
1. **Office and user management**: office profile (notaire, address, CDG account, ICE/IF), users, roles.
2. **Clients (répertoire clients)**
   - Individuals: CIN or passport, marital status and regime, nationality, FR/AR names.
   - Companies: RC, ICE, IF, legal representatives.
   - Scans of identity documents, KYC and AML checks, duplicate detection.
3. **Files (dossiers)**
   - Deed type, parties with their roles (seller, buyer, lender…), property (titre foncier, address, area), price.
   - **Status workflow**: opened → documents collected → draft → signature → registration → land registry → closed.
   - **Checklist of required documents** for each deed type (certificat de propriété, quitus fiscal, note de renseignement urbanistique…).
   - Assigned clerk, deadlines and alerts.
4. **Deeds (actes)**
   - A **template editor**: each office imports its own Word models (FR/AR) and inserts variables that are filled in automatically from the file.
   - Generates Word and PDF, with versions.
   - When signed: gets a **répertoire number**, is locked, and is archived as a minute.
5. **Répertoire**: the official register, with automatic numbering per year. It can be exported and printed.
6. **Fee and tax calculation**: registration, land registry, fees + VAT, and a TPI simulation. Produces the quote (*provision*) for the client.
7. **Agenda**: appointments and signature meetings, linked to files.
8. **Documents (GED)**: upload documents to a file, with search.
9. **Dashboard**: open files, deadlines, late formalities, revenue.

### Phase 2
10. **Notarial accounting**: client-fund account (CDG) vs office account, receipts and payments per file, balance per file, invoices, bank reconciliation, accounting exports.
11. **Formality tracking**: registration receipts, land-registry filings, commercial court (RC) filings for companies.
12. **AML module**: risk scoring, list screening, and a list of suspicious operations.

### Phase 3
13. **Client portal**: file progress, secure document upload, online payment.
14. **SMS and email notifications** (appointments, missing documents).
15. **Integrations**: DGI SIMPL, ANCFCC e-services (if APIs are open to us), e-signature, and a payment provider (CMI).
16. **SaaS billing**: plans and subscriptions for each office.

### Main deed types to cover (templates)
Real-estate sale · VEFA (Loi 44-00) · Mortgage and release (*hypothèque / mainlevée*) · Loan · Gift (*donation*) · Division (*partage*) · Commercial lease · Sale of a business (*cession de fonds de commerce*) · Company formation (SARL, SA) · Share transfers (*cession de parts*) · Power of attorney (*procuration*) · Certifications.

---

## 5. Data model (first draft)

```
Tenant(Office) ─┬─ User ── Role
                ├─ Client (Person | Company) ── IdentityDoc, KycCheck
                ├─ Property (titre foncier, réquisition, address, area)
                ├─ Dossier ─┬─ Party (client + role)
                │           ├─ Property link
                │           ├─ Task / Checklist item / Deadline
                │           ├─ Document (file, type, version)
                │           ├─ Acte ── ActeVersion ── RepertoireEntry
                │           ├─ FeeCalculation (lines: droits, CF, honoraires, TVA)
                │           ├─ Formality (enregistrement, CF, RC…)
                │           └─ LedgerEntry (client funds / office funds)
                ├─ Template (acte type, language, body with variables)
                ├─ Appointment
                ├─ RateTable (configurable fees and taxes, with validity dates)
                └─ AuditLog
```
Every table carries a `tenant_id`, and isolation is enforced **in the database** with PostgreSQL row-level security.

---

## 6. Recommended technical architecture

| Layer | Choice | Why |
|---|---|---|
| Frontend | **Next.js (React) + TypeScript**, Tailwind, shadcn/ui | Modern, and supports RTL well |
| i18n | `next-intl`, **FR / AR with a full RTL mirror** | Matches your two languages |
| Backend | **NestJS (TypeScript)** REST API, or Next.js server actions for the MVP | One language across the whole project |
| Database | **PostgreSQL** + Prisma, with row-level security per tenant | Relational data, and strong tenant isolation |
| Files | MinIO (S3-compatible), self-hosted, encrypted | Scans and deeds |
| Documents | `docxtemplater` for Word templates, Gotenberg/LibreOffice for PDF | Notaries work in Word |
| Auth | Email + password + **2FA**, sessions, per-tenant RBAC | Data is sensitive |
| Jobs | Queue (BullMQ + Redis) for alerts, PDF, emails | Deadlines and notifications |
| Hosting | **In Morocco (required)**: servers from a Moroccan provider, everything in Docker (PostgreSQL, MinIO, Redis). No managed foreign cloud services | Loi 09-08, data residency |
| Quality | Tests (Vitest, Playwright), CI on GitHub Actions, daily backups | |

The alternative is **Django + PostgreSQL**. It is faster for back-office screens, but the frontend would then be less modern. I recommend TypeScript end to end.

---

## 7. Roadmap (indicative)

| Phase | Content | Duration* |
|---|---|---|
| 0. Discovery | Validate this plan, collect real deed templates and rates, test mockups with 1–2 notaries | 2–3 weeks |
| 1. Foundations | Repo, CI, auth, tenants, roles, FR/AR RTL layout, design system | 3 weeks |
| 2. MVP core | Clients, files, checklists, agenda, documents | 5 weeks |
| 3. MVP deeds | Templates, deed generation, répertoire, fee and tax calculator | 5 weeks |
| 4. Pilot | Deploy to 1–2 pilot offices, fix issues, migrate their data | 4 weeks |
| 5. Phase 2 | Accounting (CDG), formalities, AML | 8 weeks |
| 6. Phase 3 | Client portal, notifications, integrations, SaaS billing | 8+ weeks |

\*For 1–2 developers. We will adjust after discovery.

---

## 8. Non-functional requirements
- **Security**: encryption in transit and at rest, 2FA, audit log, strict tenant isolation, backups with a restore test.
- **Legal archiving**: signed deeds cannot be changed, and are kept for the long term.
- **Performance**: the app stays fast at 100+ offices and tens of thousands of files each.
- **Ergonomics**: forms follow the office's real workflow, with keyboard shortcuts for clerks and Arabic input.
- **Availability**: 99.5% or better, with monitoring and alerts.

---

## 8b. Consequences of the discovery answers

With no notaire and no templates, the app must ship **no legal content we cannot verify**. It provides the tools; each office supplies its own templates and validates its own rates.

| Answer | Risk | What we do |
|---|---|---|
| No practising notaire | We build rules or workflows that are wrong, and notaries won't trust the app. | Recruit a paid notaire advisor (a few hours a month) before the pilot. Until then, work only from public texts: Loi 32-09, the CGI, ANCFCC fee rules. Every rate lives in an editable table that each office validates. The first pilot office acts as the validator. |
| No deed templates | We cannot ship ready-made deeds. | Build a **template editor**: an office uploads its own Word models and inserts variables (parties, price, titre foncier…). Offer a starter template only after a notaire has validated it. |
| Hosting in Morocco | No AWS, Azure or GCP region we can count on, so managed cloud services are not an option. | Rent servers or a private cloud from a Moroccan provider (for example N+ONE, inwi business, Maroc Telecom: to compare on price, SLA and certifications). Keep encrypted backups in a second Moroccan datacenter. Declare the processing to the CNDP. |

## 9. Open questions (to answer before coding)

1. ~~**Legal expert**~~: none. Recruit a notaire advisor before the pilot.
2. ~~**Deed templates**~~: none. Offices import their own through the template editor.
3. **Deed language**: which deeds must be written in Arabic, in French, or in both?
4. **Hosting**: data must stay in Morocco (decided). What is the monthly hosting budget?
5. **Accounting**: which accounting software do offices use today? Do we export to it, or replace it?
6. **Competition**: which software do Moroccan notaries use today, and what do they complain about?
7. **Pricing model**: monthly subscription per office? Per user? Is there a free trial?
8. **Data migration**: will pilot offices bring existing clients and files (Excel, old software)?
9. **Priorities**: is the proposed MVP scope right? Is anything missing or unnecessary?
10. **Team and deadline**: who develops, and is there a target launch date?
11. **Branding**: the product name, logo and colours.
12. **Adouls**: are they out of scope? (They are a separate profession with their own rules.)
13. **First pilot office**: do you know a notary office willing to test the app?

---

## 10. Next steps
1. Find a notaire advisor (paid, part-time) and one pilot office. This is the biggest risk in the project.
2. Compare Moroccan hosting providers on price, SLA, backups and certifications.
3. Answer the remaining open questions in §9.
4. Produce clickable **mockups** (UI in FR/AR) of the main screens: dashboard, file, deed, template editor, répertoire.
5. Freeze the MVP scope, then start **Phase 1** (project setup).
