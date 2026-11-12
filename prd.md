# Master Product Requirements Document (PRD)

## Integrated Multi-Tenant ERP Platform (SaaS): Accounting, HR, Payroll, Inventory, Procurement

| Field | Value |
|---|---|
| Document version | 0.4 (Draft) |
| Status | For review |
| Owner | Software Architecture |
| Last updated | 2026-11-12 |
| Tech stack | ASP.NET Core Web API, React (all UI), SQL Server, Clean Architecture, Cortex.Mediator, Serilog |
| Target customers | Small and medium businesses (SME) |
| Target market | Bangladesh first, international expansion later |
| Hosting | IIS (initial), then Azure and Docker |

---

## 1. Introduction

### 1.1 Purpose
This document defines the product, functional, non-functional and architectural requirements for a modular ERP platform covering five business domains: **Accounting, HR, Payroll, Inventory and Procurement/Purchase**. It is the single source of truth for design, development, testing and acceptance.

### 1.2 Vision
One integrated system where a purchase, a stock movement, a salary run or an expense is recorded once and flows automatically into the general ledger, with full auditability.

### 1.3 Goals
- G1. Provide a single, consistent data platform across all five modules.
- G2. Automate financial postings from operational transactions (purchase, stock, payroll).
- G3. Enforce approval workflows, segregation of duties and a complete audit trail.
- G4. Deliver a maintainable, testable codebase using Clean Architecture and CQRS.
- G5. Operate as a **multi-tenant SaaS** (Tenant > Company > Branch) from day one.
- G6. Ship with a complete **Bangladesh localization pack** (VAT, income tax, Labour Act rules, BDT; Bangla UI in a later release) and a pluggable **country-pack** design for international expansion.
- G7. Run on IIS first and move to Azure/Docker later **without code changes** (configuration only).

### 1.4 Non-Goals (Out of Scope for v1)
- Sales/CRM and Point of Sale (planned later; the design must not block them).
- Manufacturing/MRP.
- Native mobile applications (responsive web UI only in v1).
- Full tax-authority e-filing integrations (export files only).
- Biometric hardware integration (import via CSV/API only).

### 1.5 Stakeholders and Personas

| Persona | Primary needs |
|---|---|
| System Administrator | Users, roles, company setup, configuration, logs |
| Finance Manager / Accountant | Chart of accounts, journals, reconciliation, reports, period close |
| HR Manager | Employee records, leave, attendance, recruitment basics |
| Payroll Officer | Salary structures, monthly payroll run, payslips, statutory deductions |
| Store Keeper | Receive, issue, transfer, count stock |
| Procurement Officer | Requisitions, RFQs, purchase orders, vendor management |
| Department Approver / Manager | Approve requests (leave, PR, PO, expense) |
| Auditor (read-only) | Audit trail and reports without edit rights |
| Employee (self-service) | Payslips, leave requests, personal data |

---

## 2. Decisions, Assumptions and Open Questions

### 2.1 Confirmed Decisions

| # | Topic | Decision |
|---|---|---|
| D1 | Market | Bangladesh first (BDT, NBR VAT/tax, Bangladesh Labour Act); international later via country packs |
| D2 | Frontend | **React SPA for everything**: login, onboarding, business screens, tenant admin, platform admin and document printing. **ASP.NET Core MVC is not used**, and is added only as a fallback if a concrete need appears (see 4.7, ADR-009) |
| D3 | Solution structure | **One project per module and per layer** (Domain, Application, Infrastructure for each module) |
| D4 | Tenancy | **Multi-tenant SaaS** (see 4.6) |
| D5 | Inventory costing | **FIFO** (cost layers); weighted average may be added later through a costing strategy |
| D6 | Hosting | **IIS initially**, later **Azure and Docker**; code must be portable from day one (see 4.8 and section 18) |
| D7 | Target customers and scale | **Small and medium businesses**; keep the design simple (see A8) |
| D8 | Language | **English in v1; Bangla UI in a later release.** The app is i18n-ready from day one |
| D9 | FIFO scope | FIFO cost layers kept **per item per warehouse** |
| D10 | Tenant isolation | Shared database by default; optional dedicated database per tenant |
| D11 | UI kit | **MUI** (Material UI) with MUI X Data Grid for data-heavy screens |
| D12 | SPA hosting | IIS phase: React build **served by the API from one IIS site**. Azure phase: optionally move to a separate static host/CDN (see 18.2) |
| D13 | Mushak documents | Must match the **exact NBR-prescribed layout and fields**; templates are versioned and validated against current NBR forms before go-live |

### 2.2 Assumptions
- A1. **Modular monolith** with strict module boundaries so modules can later be split into services.
- A2. One SQL Server database (shared schema, `TenantId` discriminator) by default, with the option of a **dedicated database per tenant** for large customers.
- A3. Schemas per module: `platform`, `common`, `acc`, `hr`, `pay`, `inv`, `proc`.
- A4. Multi-currency supported; base currency per company (BDT default).
- A5. UI language: English in v1. All text lives in resource files (i18next) so Bangla (`bn-BD`) can be added later without code changes.
- A6. Statutory rules (VAT, income tax, labour rules) are **data-driven and effective-dated**, never hard-coded.
- A7. Tenant subscription plans control modules, user limits and company limits. Online payment collection for subscriptions is out of scope for v1.
- A8. **Sizing (planning assumptions for SME):** up to about 200 tenants in the first year; typically 5 to 50 users per tenant (maximum about 200); 1 to 3 companies per tenant; up to about 500 employees per tenant; roughly 500 to 3,000 business documents per tenant per day. Consequences: pooled shared database, a single API instance that can scale out later, no microservices, no read replicas, Redis optional, simple background-job setup.

### 2.3 Open Questions
None blocking. All earlier questions are resolved (see 2.1). New questions will be added here as they arise during design.

---

## 3. Technology Stack

| Concern | Choice |
|---|---|
| Runtime / Framework | .NET (current LTS), ASP.NET Core Web API |
| Architecture | Clean Architecture, CQRS, modular monolith, multi-tenant |
| Mediator / CQRS | **Cortex.Mediator** (commands, queries, notifications, pipeline behaviors) |
| Frontend (all UI) | **React + TypeScript**, Vite, React Router, TanStack Query, React Hook Form + Zod, i18next (English now, Bangla later), UI kit **MUI** (+ MUI X Data Grid), typed API client generated from OpenAPI; code-split per module |
| ORM (writes) | Entity Framework Core (SQL Server provider) |
| Reads | EF Core `AsNoTracking`, or Dapper for heavy reports |
| Database | Microsoft SQL Server (Azure SQL later) |
| Logging | **Serilog** (structured) with sinks: Console, rolling File, SQL Server or Seq; Application Insights/OpenTelemetry on Azure |
| Validation | FluentValidation (in a Cortex.Mediator pipeline behavior) |
| Mapping | Mapster or AutoMapper (explicit mappings preferred) |
| Auth | ASP.NET Core Identity, JWT access tokens plus refresh tokens |
| Background jobs | Hangfire (SQL Server storage) or Quartz.NET; tenant-aware |
| API docs | Swagger / OpenAPI |
| Caching | `IMemoryCache`; Redis for scale-out (tenant-prefixed keys) |
| File storage | `IFileStorage` abstraction: local/network disk on IIS, Azure Blob later |
| Testing | xUnit, FluentAssertions, Moq/NSubstitute, Testcontainers (SQL Server), Respawn, Vitest/Playwright for React |
| Reporting / Printing | Server-side PDF with QuestPDF and Excel with ClosedXML; React shows the PDF and offers print/download |
| CI/CD | GitHub Actions or Azure DevOps; Docker image |

---

## 4. Architecture

### 4.1 Clean Architecture Layers

Dependencies point **inward only**.

```
Presentation (API)  ->  Application  ->  Domain
Infrastructure      ->  Application  ->  Domain
```

| Layer | Contents | May reference |
|---|---|---|
| **Domain** | Entities, value objects, aggregates, enums, domain events, domain exceptions, business rules | Nothing |
| **Application** | Commands, queries, handlers (Cortex.Mediator), DTOs, validators, pipeline behaviors, interfaces (`IRepository`, `IUnitOfWork`, `ICurrentUser`, `IDateTime`, `IEmailSender`), mapping profiles | Domain |
| **Infrastructure** | EF Core `DbContext`, configurations, migrations, repositories, Identity, Serilog setup, email/file storage, background jobs, external integrations | Application, Domain |
| **Presentation** | Controllers/Minimal APIs, middleware, filters, DI composition root, Swagger | Application (and Infrastructure only in the composition root) |

### 4.2 Solution Structure (one project per module and layer)

```
src/
  BuildingBlocks/
    ERP.SharedKernel/              # Entity, AggregateRoot, ValueObject, Result<T>, IDomainEvent, ITenantEntity
    ERP.Contracts/                 # Integration events and public cross-module interfaces
  Modules/
    Platform/                      # Tenants, plans, subscriptions, domains (catalog)
      ERP.Platform.Domain / .Application / .Infrastructure
    Common/                        # Auth, Company, Branch, Currency, Audit, Settings, Workflow, Notifications
      ERP.Common.Domain / .Application / .Infrastructure
    Accounting/
      ERP.Accounting.Domain / .Application / .Infrastructure
    HR/
      ERP.HR.Domain / .Application / .Infrastructure
    Payroll/
      ERP.Payroll.Domain / .Application / .Infrastructure
    Inventory/
      ERP.Inventory.Domain / .Application / .Infrastructure
    Procurement/
      ERP.Procurement.Domain / .Application / .Infrastructure
  CountryPacks/
    ERP.CountryPack.Abstractions/  # ICountryPack contracts
    ERP.CountryPack.Bangladesh/    # BD tax codes, payroll rules, leave policy, COA template, reports
  ERP.Api/                         # Presentation: Web API, composition root, (IIS phase) serves the SPA static files
frontend/
  erp-web/                         # React + TypeScript SPA (login, business UI, admin, printing)
tests/
  ERP.UnitTests/
  ERP.IntegrationTests/
  ERP.ArchitectureTests/           # NetArchTest: layer and module dependency rules
docs/
  prd.md
  adr/
```

Rules:
- Each module has three projects: `*.Domain` (no references), `*.Application` (references its own Domain and `ERP.Contracts`), `*.Infrastructure` (references its own Application and Domain).
- Modules reference **only** `ERP.SharedKernel` and `ERP.Contracts`, never another module's projects.
- `ERP.Api` is the only place that wires all modules together (composition root).
- Each module owns its own `DbContext` and schema.

### 4.3 CQRS with Cortex.Mediator

- **Commands** change state, return `Result<T>` or an ID, and run inside a transaction.
- **Queries** read only, return DTOs, never modify state.
- **Notifications** (domain/integration events) let modules react to each other without direct references.
- **Pipeline behaviors**, executed in this order:
  1. `LoggingBehavior`: Serilog entry/exit, duration, correlation ID.
  2. `AuthorizationBehavior`: permission check by attribute on the request.
  3. `ValidationBehavior`: FluentValidation; returns validation errors.
  4. `TransactionBehavior`: wraps commands in a unit of work; commits and dispatches domain events after success.
  5. `CachingBehavior` (queries only, opt-in).
  6. `PerformanceBehavior`: warns when a request exceeds a threshold (for example 500 ms).

Naming convention: `CreatePurchaseOrderCommand`, `CreatePurchaseOrderCommandHandler`, `CreatePurchaseOrderCommandValidator`, `GetPurchaseOrderByIdQuery`.

### 4.4 Cross-Module Communication
- Modules never reference each other's Domain or Infrastructure projects.
- They communicate through **integration events** (Cortex.Mediator notifications) and small public **contracts** interfaces.
- Examples:
  - `GoodsReceivedEvent` (Inventory) -> Accounting posts Dr Inventory / Cr GRNI.
  - `VendorInvoiceApprovedEvent` (Procurement) -> Accounting posts Dr GRNI / Cr Accounts Payable.
  - `PayrollApprovedEvent` (Payroll) -> Accounting posts salary expense and liabilities.
- v2 option: Outbox pattern table for reliable event delivery.

### 4.5 Cross-Cutting Concerns
- **Serilog**: configured in `Program.cs` via `UseSerilog`; request logging middleware; enrichers (`MachineName`, `CorrelationId`, `UserId`, `CompanyId`); never log passwords, tokens or personal salary data.
- **Global exception handling** middleware returning RFC 7807 `ProblemDetails`.
- **Audit trail**: `SaveChanges` interceptor writing who/when/what-changed to `common.AuditLog`.
- **Soft delete** and **concurrency tokens** (`rowversion`) on all transactional entities.
- **Tenant and company filters**: EF Core global query filters on `TenantId` (always) and `CompanyId` (where applicable); `TenantId` is stamped automatically by a `SaveChanges` interceptor and never accepted from request bodies.
- **Health checks** for database and background jobs.

### 4.6 Multi-Tenancy Architecture

**Hierarchy:** Tenant (customer organization) > Company (legal entity) > Branch.

**Isolation models (both supported behind one abstraction):**

| Model | Use | How |
|---|---|---|
| **Pooled** (default) | Most tenants | Shared database and schema; every tenant-owned table has `TenantId`; EF global filter plus SQL Server **Row-Level Security** as defense in depth (tenant id set via `SESSION_CONTEXT` on each connection) |
| **Dedicated DB** (premium) | Large or regulated tenants | One database per tenant; connection string resolved from the platform catalog by `ITenantConnectionResolver` |

**Platform catalog** (`platform` schema): `Tenant`, `TenantDomain`, `Plan`, `Subscription`, `TenantModule`, `TenantConnection`, `FeatureFlag`.

**Tenant resolution (in order):** JWT `tenant_id` claim (authoritative), then subdomain (`acme.erp.example`), then `X-Tenant-Id` header for service clients. A mismatch between claim and subdomain returns `403`.

**Key rules**
- MT-1: No query may run without a resolved tenant (fail closed).
- MT-2: Unique indexes and document numbers are scoped per tenant (`TenantId` is the first column of business-key indexes).
- MT-3: Cache keys, file storage paths, background jobs and log events carry `TenantId`.
- MT-4: Background jobs execute inside an explicit tenant scope.
- MT-5: Per-tenant rate limits and query timeouts to prevent noisy neighbours.
- MT-6: Per-tenant data export, backup/restore procedure and offboarding (data deletion on request).
- MT-7: Plan-based feature gating (modules, user limit, company limit) enforced in the `AuthorizationBehavior`.
- MT-8: Mandatory **tenant isolation tests** in CI (tenant A can never read or write tenant B data).

### 4.7 Frontend Architecture

A **single React SPA** (TypeScript) provides all user interfaces. The backend exposes only the Web API.

| Area | Implementation |
|---|---|
| Public pages | Login, signup/tenant onboarding, forgot/reset password, MFA, accept invitation, built in React and backed by Identity endpoints in `ERP.Api` |
| Business UI | Accounting, HR, Payroll, Inventory, Procurement: grids, forms, approvals, dashboards, reports; one lazy-loaded feature folder per module |
| Tenant admin | Users, roles/permissions, company/branch, settings, approval rules, branding |
| Platform admin | Route-guarded area for platform super-admins only (tenants, plans, usage, audited impersonation); loaded as a separate lazy chunk and never shipped to ordinary users' menus |
| Printing | Official documents (payslip, PO, vouchers, VAT challan, reports) are generated **server-side as PDF** by the API (QuestPDF); React displays them and offers print/download. Simple screens also use print-friendly CSS |
| Navigation and gating | Menus and buttons are shown from the user's permissions and the tenant's plan; the API remains the real enforcement point |

**Auth flow:** React login page calls the API; the API returns a short-lived JWT access token (held in memory) and sets a refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie. SPA and API must share the same registrable domain (for example `app.erp.example` and `api.erp.example`). Strict Content-Security-Policy and standard security headers are set on the SPA host.

**Fallback:** ASP.NET Core MVC is **not** part of the plan. It would be added as a separate `ERP.Web` project only if a concrete need appears (for example a server-rendered public marketing site or a document type that cannot be produced reliably as PDF). Any such change needs an ADR.

### 4.8 Hosting Portability

Phase 1 runs on **IIS**; phase 2 on **Azure** and **Docker**. To make the move a configuration change, the code follows these rules from the start:
- Stateless API; no in-process session state; no dependency on local file paths (use `IFileStorage`).
- All configuration through `appsettings.*` and environment variables; secrets outside source control.
- ASP.NET Core Data Protection keys persisted to the database (not machine-local).
- Time in UTC; forwarded-headers support for reverse proxies and load balancers.
- Health check endpoints (`/health/live`, `/health/ready`).
- Hangfire/Quartz use SQL Server storage so jobs survive restarts and multiple instances.

---

## 5. Common / Platform Requirements

| ID | Requirement | Priority |
|---|---|---|
| CMN-01 | Tenant, Company, Branch and Department master data | Must |
| CMN-02 | User registration, login, password policy, lockout, optional 2FA | Must |
| CMN-03 | Role-based and permission-based access control (module/feature/action level) | Must |
| CMN-04 | Configurable multi-level approval workflow engine (by amount, department, document type) | Must |
| CMN-05 | Document numbering sequences per company, per document type, per fiscal year | Must |
| CMN-06 | Currency and exchange-rate management | Must |
| CMN-07 | Audit log viewer with filters (user, entity, date) | Must |
| CMN-08 | Attachments on documents (file storage abstraction) | Should |
| CMN-09 | In-app and email notifications for pending approvals | Should |
| CMN-10 | System settings and fiscal year/period management | Must |
| CMN-11 | Data import/export (CSV/Excel) for master data | Should |
| CMN-12 | Tenant provisioning (self-service signup or admin-created): creates tenant, first company, admin user and seeds the selected country pack | Must |
| CMN-13 | Subscription plans, module licensing and limits per tenant | Must |
| CMN-14 | Platform super-admin area (React): manage tenants, plans, status, usage, impersonation with audit | Must |
| CMN-15 | Per-tenant branding (logo, document headers) and per-tenant settings | Should |
| CMN-16 | Server-side PDF generation for all printable documents with tenant branding; React viewer with print and download | Must |

### 5.1 Bangladesh Localization Pack (`ERP.CountryPack.Bangladesh`)

> All rates, thresholds and rules below are **effective-dated data**. They must be verified against current NBR (National Board of Revenue) and Ministry of Labour publications at implementation time and updated without code deployment.

| ID | Requirement | Priority |
|---|---|---|
| BD-01 | Base currency **BDT**; number formatting in lakh/crore grouping; amount-in-words in English (Bangla later) | Must |
| BD-02 | **Later release:** Bangla UI (`bn-BD`), Bangla name fields and Bangla amount-in-words. v1 is English only but i18n-ready (all text in resource files) | Could |
| BD-03 | Fiscal year default **1 July to 30 June** (configurable to Jan-Dec) | Must |
| BD-04 | **VAT**: tax codes for standard, reduced, zero-rated, exempt and truncated-base cases; supplementary duty; **VDS** (VAT deducted at source) on vendor bills; BIN on parties | Must |
| BD-05 | **Mushak** documents and registers in the **exact NBR-prescribed layout and fields** (for example tax invoice/challan, purchase and sales books, monthly return worksheet), generated as PDF/Excel; templates versioned and validated against current NBR forms before go-live; no direct NBR submission in v1 | Must |
| BD-06 | **Income tax**: TIN on parties and employees; **TDS/AIT** codes and rates per payment type; challan/deposit tracking | Must |
| BD-07 | **Salary income tax**: slabs per assessment year, gender/category thresholds and rebate rules as data | Must |
| BD-08 | **Labour Act** leave defaults (casual, sick, earned/annual, festival, maternity), weekly off, overtime rule, as an editable policy template | Must |
| BD-09 | **Festival bonus** (Eid) scheduling and calculation | Must |
| BD-10 | **Provident fund**, gratuity and (optional) **WPPF** (Workers' Profit Participation Fund) support | Should |
| BD-11 | Government/Eid holiday calendar maintained per year (lunar dates entered manually) | Must |
| BD-12 | Salary payment files: bank advice, **BEFTN** format, and mobile financial service (bKash, Nagad, Rocket) payment lists | Should |
| BD-13 | ID validation (NID, TIN, BIN) with configurable patterns | Should |
| BD-14 | Default **chart of accounts template** aligned with BFRS/IFRS | Must |
| BD-15 | **Import purchases**: LC reference, customs duty, advance income tax, freight; landed-cost allocation (see PRC-15) | Should |

### 5.2 Internationalization and Country Packs

- INT-1: A country pack implements `ICountryPack` and supplies: tax codes and rules, payroll statutory rules, leave policy templates, chart-of-accounts template, statutory reports, ID validators, holidays, currency/number formats, payment file formats.
- INT-2: The tenant (or company) selects a country pack at provisioning; multiple packs can co-exist in one tenant for multi-country groups.
- INT-3: No country-specific logic is allowed in module Domain or Application projects; it lives only in country packs.
- INT-4: All user-facing text is in resource files; formats come from `CultureInfo` and company settings.

### 5.3 Subscription and Plans (v1 scope)
Plans define enabled modules, maximum users, companies and storage. Plan changes take effect immediately. Billing/invoicing of tenants and online payment collection are out of scope for v1.

---

## 6. Module: Accounting

### 6.1 Scope
General ledger, accounts payable, accounts receivable (basic), cash and bank, fixed assets (basic), tax, financial reporting.

### 6.2 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| ACC-01 | Hierarchical **Chart of Accounts** (Asset, Liability, Equity, Income, Expense); account types, control accounts, active/inactive | Must |
| ACC-02 | **Fiscal years and periods**; open/close/lock period; prevent posting to closed periods | Must |
| ACC-03 | **Journal entries**: manual, recurring, reversing; debit must equal credit; draft -> posted workflow | Must |
| ACC-04 | **Automated postings** from Inventory, Procurement and Payroll via configurable posting rules/account mappings | Must |
| ACC-05 | Multi-currency journals with exchange-rate capture and revaluation | Should |
| ACC-06 | **Cost centers / dimensions** (department, project, branch) on journal lines | Should |
| ACC-07 | **Accounts Payable**: vendor bills, debit/credit notes, payment vouchers, payment allocation, aging | Must |
| ACC-08 | **Accounts Receivable** (basic): customer invoices, receipts, aging | Should |
| ACC-09 | **Cash and bank**: bank accounts, receipts/payments, cheque register, **bank reconciliation** | Must |
| ACC-10 | **Tax**: tax codes/rates (VAT/GST, withholding), tax on purchase documents, tax summary report | Must |
| ACC-11 | **Fixed assets**: asset register, categories, depreciation methods (straight-line, declining balance), monthly depreciation run, disposal | Should |
| ACC-12 | **Budgeting**: annual budget per account/cost center; budget vs actual | Could |
| ACC-13 | **Period-end close**: closing entries, retained earnings transfer | Must |
| ACC-14 | **Reports**: Trial Balance, General Ledger, Income Statement, Balance Sheet, Cash Flow, AP/AR Aging, Day Book, Tax report; export to PDF/Excel | Must |
| ACC-15 | Posted entries are immutable; corrections only by reversal | Must |
| ACC-16 | VAT/VDS and TDS/AIT accounting per the Bangladesh pack (BD-04 to BD-06): tax sub-ledgers, payable/receivable accounts, deposit tracking | Must |
| ACC-17 | Chart of accounts template per country pack, copied into each new tenant/company at provisioning | Must |

### 6.3 Key Entities
`Account`, `AccountGroup`, `FiscalYear`, `FiscalPeriod`, `JournalEntry`, `JournalLine`, `PostingRule`, `Vendor/CustomerLedger` (via Procurement/Sales parties), `BankAccount`, `BankStatementLine`, `Payment`, `PaymentAllocation`, `TaxCode`, `FixedAsset`, `DepreciationSchedule`, `Budget`, `CostCenter`.

### 6.4 Business Rules
- BR-ACC-1: Sum(debits) = Sum(credits) per journal entry, per currency and in base currency.
- BR-ACC-2: Posting date must fall in an open fiscal period.
- BR-ACC-3: Control accounts (AP, AR, Inventory) can only be posted by system documents, not manual journals (unless permission granted).
- BR-ACC-4: Every automated journal stores a reference to its source document (type + ID).

---

## 7. Module: Human Resources (HR)

### 7.1 Scope
Employee master, organization structure, attendance, leave, recruitment (light), performance (light), documents.

### 7.2 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| HR-01 | **Employee master**: personal, contact, emergency contact, bank details, national ID, photo, documents | Must |
| HR-02 | **Organization structure**: departments, designations/job titles, grades, reporting lines, branches | Must |
| HR-03 | **Employment lifecycle**: join, probation, confirmation, transfer, promotion, resignation, termination; history retained | Must |
| HR-04 | **Contracts**: type (permanent/contract/part-time), start/end dates, renewal alerts | Should |
| HR-05 | **Attendance**: manual entry, CSV/API import from devices, shifts and rosters, late/early/overtime calculation | Must |
| HR-06 | **Leave management**: leave types, yearly entitlement, accrual, carry-forward, request -> approval, balance, calendar, holidays | Must |
| HR-07 | **Self-service**: employees view profile, leave balance, attendance, payslips; submit requests | Should |
| HR-08 | **Recruitment (light)**: vacancy, candidate, interview stages, offer, convert candidate to employee | Could |
| HR-09 | **Performance (light)**: appraisal cycles, goals, ratings | Could |
| HR-10 | **Training and certifications** tracking with expiry alerts | Could |
| HR-11 | **Loans and advances** to employees (feeds Payroll deductions) | Should |
| HR-12 | **Exit management**: clearance checklist, final settlement trigger to Payroll | Should |
| HR-13 | **HR reports**: headcount, turnover, leave summary, attendance summary, birthdays/anniversaries | Must |
| HR-14 | Sensitive fields (salary, ID numbers) protected by field-level permissions | Must |
| HR-15 | Leave and attendance policies loaded from the country pack template (Bangladesh Labour Act defaults) and editable per company | Must |
| HR-16 | Configurable weekly off days, festival and government holidays per company/branch | Must |

### 7.3 Key Entities
`Employee`, `EmployeeContract`, `Department`, `Designation`, `Grade`, `Shift`, `AttendanceRecord`, `LeaveType`, `LeavePolicy`, `LeaveBalance`, `LeaveRequest`, `Holiday`, `EmployeeLoan`, `Vacancy`, `Candidate`, `AppraisalCycle`, `EmployeeDocument`.

### 7.4 Business Rules
- BR-HR-1: Leave request cannot exceed available balance unless the leave type allows negative balance.
- BR-HR-2: Overlapping leave requests for the same employee are rejected.
- BR-HR-3: Employee changes (position, grade, salary link) are effective-dated.
- BR-HR-4: An employee cannot approve their own request.

---

## 8. Module: Payroll

### 8.1 Scope
Salary structures, earnings and deductions, monthly payroll processing, statutory deductions, payslips, payments, accounting integration.

### 8.2 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| PAY-01 | **Salary components**: earnings (basic, allowances, bonus), deductions (tax, insurance, loan), employer contributions; fixed, percentage or formula based | Must |
| PAY-02 | **Salary structures** assigned to grades or individual employees, effective-dated | Must |
| PAY-03 | **Payroll period** setup (monthly default; weekly/bi-weekly optional) | Must |
| PAY-04 | **Payroll run**: draft -> calculate -> review -> approve -> lock -> pay; re-calculation allowed before approval | Must |
| PAY-05 | Inputs pulled from HR: attendance days, unpaid leave, overtime, loans, joiners/leavers (pro-rata) | Must |
| PAY-06 | **Statutory deductions**: configurable tax slabs, social security/provident fund; rules stored as data | Must |
| PAY-07 | **Overtime, bonuses, arrears, incentives** as variable inputs | Should |
| PAY-08 | **Loan/advance recovery** schedule with automatic deduction | Should |
| PAY-09 | **Payslips**: PDF generation, email distribution, self-service view | Must |
| PAY-10 | **Bank transfer file** generation (configurable CSV format) | Should |
| PAY-11 | **Final settlement**: unpaid salary, leave encashment, gratuity, deductions | Should |
| PAY-12 | **Accounting integration**: on approval, publish `PayrollApprovedEvent`; Accounting posts salary expense, tax payable, PF payable, net salary payable | Must |
| PAY-13 | **Reports**: payroll register, department cost summary, tax withheld, bank advice, year-to-date earnings | Must |
| PAY-14 | Approved payroll is immutable; corrections via adjustment in a later period | Must |
| PAY-15 | Payroll runs execute as a **background job** with progress tracking and idempotency | Should |
| PAY-16 | Bangladesh salary income tax computed from effective-dated slabs; monthly tax withheld, challan reference, annual tax statement per employee (BD-07) | Must |
| PAY-17 | Festival bonus run (BD-09) and overtime per Labour Act rule (BD-08) | Must |
| PAY-18 | Provident fund (employee + employer), gratuity accrual, optional WPPF (BD-10) | Should |
| PAY-19 | Bank advice, BEFTN and MFS payment lists (BD-12) | Should |

### 8.3 Key Entities
`SalaryComponent`, `SalaryStructure`, `SalaryStructureLine`, `EmployeeSalary`, `PayrollPeriod`, `PayrollRun`, `PayrollRunEmployee`, `PayrollRunLine`, `Payslip`, `TaxSlab`, `StatutoryRule`, `LoanRecovery`, `FinalSettlement`.

### 8.4 Business Rules
- BR-PAY-1: One approved payroll run per period per company/branch.
- BR-PAY-2: Net pay = total earnings - total deductions; negative net pay is flagged for review.
- BR-PAY-3: Salary calculation is deterministic and reproducible from stored inputs and rule versions.
- BR-PAY-4: Access to payroll data is restricted to the Payroll role; actions are fully audited.

---

## 9. Module: Inventory

### 9.1 Scope
Item master, warehouses and locations, stock transactions, valuation, adjustments, counts, reordering.

### 9.2 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| INV-01 | **Item master**: code, name, category, unit of measure (with conversions), barcode, tax code, reorder level, active flag | Must |
| INV-02 | **Item categories** with default accounting mappings (inventory, COGS, adjustment accounts) | Must |
| INV-03 | **Warehouses and bin locations** per branch | Must |
| INV-04 | **Goods Receipt Note (GRN)** against purchase order (full/partial) or direct | Must |
| INV-05 | **Goods Issue** to department/cost center/project with approval | Must |
| INV-06 | **Stock transfers** between warehouses with in-transit status | Should |
| INV-07 | **Stock adjustments** (increase/decrease) with reason codes and approval | Must |
| INV-08 | **Physical stock count**: count sheets, variance, approval, auto-adjustment | Should |
| INV-09 | **Valuation: FIFO** using cost layers (see 9.5); costing implemented behind an `ICostingStrategy` so weighted average can be added later | Must |
| INV-10 | **Stock ledger** (immutable movement log) and real-time on-hand, reserved, available quantities | Must |
| INV-11 | **Batch/lot and serial tracking**, expiry dates | Could |
| INV-12 | **Reorder alerts** and suggested purchase requisitions | Should |
| INV-13 | **Return to vendor** and **return from department** | Should |
| INV-14 | **Prevent negative stock** (configurable per warehouse) | Must |
| INV-15 | **Accounting integration**: each stock movement publishes an event; Accounting posts per item category mapping | Must |
| INV-16 | **Reports**: stock on hand, stock ledger/card, valuation, ageing, slow-moving, movement summary, reorder report, **FIFO layer report** (remaining quantity and cost per layer) | Must |
| INV-17 | **Landed cost** adjustment of receipt layers (duty, freight, LC charges) with variance posting if stock was already issued | Should |

### 9.3 Key Entities
`Item`, `ItemCategory`, `UnitOfMeasure`, `UomConversion`, `Warehouse`, `Location`, `StockLedgerEntry`, `StockBalance`, `GoodsReceipt`, `GoodsIssue`, `StockTransfer`, `StockAdjustment`, `StockCount`, `StockCostLayer`, `StockCostLayerConsumption`, `Batch`, `SerialNumber`.

### 9.4 Business Rules
- BR-INV-1: Stock quantity changes **only** through posted stock documents; direct edits are not allowed.
- BR-INV-2: Posted stock documents are immutable; corrections via reversing documents.
- BR-INV-3: Valuation is **FIFO**. Every receipt creates a cost layer; every issue consumes the oldest layers first and records exactly which layers and costs it consumed.
- BR-INV-4: Stock postings are concurrency-safe (optimistic concurrency or `UPDLOCK` on balance and layer rows).

### 9.5 FIFO Costing Design
- `StockCostLayer`: `TenantId`, `ItemId`, `WarehouseId`, `ReceiptDate`, `QtyReceived`, `QtyRemaining`, `UnitCost` (including landed cost), `SourceDocumentType/Id`, `IsClosed`.
- **Issue**: consume layers ordered by `ReceiptDate`, then `Id`; write one `StockCostLayerConsumption` row per layer touched; issue cost (COGS) = sum of `qty x layer cost`.
- **Transfer**: consume source layers and create destination layers preserving unit cost and original receipt date.
- **Return to vendor**: reduce the original receipt layer when possible; otherwise consume oldest layers.
- **Return from department**: create a new layer at the original issue cost.
- **Adjustment increase**: create a layer at a user-specified or last-known cost; **decrease** consumes oldest layers.
- **Landed cost**: increase remaining-layer cost; the portion already consumed is posted to a cost-variance account.
- **Negative stock** is not allowed by default (FIFO needs a layer to consume).
- Layers are the basis of valuation: Inventory control account = sum(`QtyRemaining x UnitCost`).
- Costing scope (confirmed): layers are kept **per item per warehouse**; transfers move layers between warehouses at their original cost.

---

## 10. Module: Procurement / Purchase

### 10.1 Scope
Vendors, purchase requisitions, RFQ and quotation comparison, purchase orders, receiving, vendor invoice matching, vendor performance.

### 10.2 Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| PRC-01 | **Vendor master**: profile, contacts, bank, tax ID, payment terms, categories, status (active/blocked) | Must |
| PRC-02 | **Purchase Requisition (PR)**: created by department; multi-level approval by amount/department | Must |
| PRC-03 | **RFQ**: send to multiple vendors, record quotations, side-by-side **comparison**, select winner | Should |
| PRC-04 | **Purchase Order (PO)**: from PR/quotation or direct; taxes, discounts, delivery terms, approval, PDF/email to vendor | Must |
| PRC-05 | PO **status lifecycle**: Draft -> Pending Approval -> Approved -> Partially Received -> Received -> Closed / Cancelled | Must |
| PRC-06 | **Goods receipt** integration with Inventory (partial and over-receipt tolerance) | Must |
| PRC-07 | **Vendor invoice** entry with **3-way match** (PO, GRN, invoice) and tolerance rules | Must |
| PRC-08 | **Purchase returns** and debit notes | Should |
| PRC-09 | **Payments** to vendors through Accounting (AP) with payment terms and due-date tracking | Must |
| PRC-10 | **Blanket/contract orders** and price lists per vendor | Could |
| PRC-11 | **Vendor evaluation**: on-time delivery, quality, price variance scoring | Could |
| PRC-12 | **Budget check** against Accounting budget at PR/PO approval | Could |
| PRC-13 | **Reports**: open POs, PO vs receipt vs invoice, purchase by vendor/item/period, price history, pending approvals | Must |
| PRC-14 | Non-stock purchases (services/expenses) supported without GRN | Should |
| PRC-15 | **Import purchases**: LC reference, customs duty, freight and other charges allocated to items as landed cost (feeds INV-17) | Should |
| PRC-16 | **VDS/TDS** computed on vendor invoices and payments per the Bangladesh pack | Must |

### 10.3 Key Entities
`Vendor`, `VendorContact`, `PurchaseRequisition`, `PurchaseRequisitionLine`, `Rfq`, `RfqVendor`, `Quotation`, `PurchaseOrder`, `PurchaseOrderLine`, `VendorInvoice`, `VendorInvoiceLine`, `PurchaseReturn`, `VendorPriceList`.

### 10.4 Business Rules
- BR-PRC-1: A PO can be issued only after approval; approved POs are edited only via amendment with re-approval.
- BR-PRC-2: Received quantity cannot exceed ordered quantity plus the configured tolerance.
- BR-PRC-3: A vendor invoice cannot be approved if quantity/price variance exceeds tolerance, unless overridden with permission.
- BR-PRC-4: Blocked vendors cannot receive new POs.

---

## 11. Cross-Module Process Flows

### 11.1 Procure-to-Pay
```
PR -> Approval -> RFQ (optional) -> PO -> Approval -> Send to vendor
   -> GRN (Inventory)  ---- event ---> Accounting: Dr Inventory, Cr GRNI
   -> Vendor Invoice (3-way match) --- event ---> Accounting: Dr GRNI (+ Tax), Cr Accounts Payable
   -> Payment (Accounting) : Dr Accounts Payable, Cr Bank
```

### 11.2 Hire-to-Pay
```
Employee joins (HR) -> Salary structure assigned (Payroll)
   -> Attendance and leave (HR) -> Payroll run (calculate, review, approve)
   -> Payslips + bank file -> Accounting journal (expense, liabilities)
   -> Salary payment -> Journal Dr Salary Payable, Cr Bank
```

### 11.3 Stock-to-Ledger
```
Goods issue / adjustment / transfer -> FIFO layer consumption -> StockLedgerEntry -> event
   -> Accounting posts per category mapping (Dr COGS/Expense, Cr Inventory)
```

---

## 12. Data and Database Requirements (SQL Server)

- **Schemas** per module: `platform`, `common`, `acc`, `hr`, `pay`, `inv`, `proc`.
- **Tenant isolation**: `TenantId` on every tenant-owned table; Row-Level Security policy per table; all business-key unique indexes start with `TenantId`.
- **Primary keys**: `bigint IDENTITY` for transactional tables or `uniqueidentifier` (sequential) if distributed generation is needed. Pick one convention and apply it everywhere.
- **Standard columns** on all entities: `Id`, `TenantId`, `CompanyId`, `CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy`, `IsDeleted`, `RowVersion`.
- **Money**: `decimal(18,4)`; quantities `decimal(18,4)`; rates `decimal(18,6)`. Never use `float`.
- **Dates**: `datetime2`, stored in UTC; display in the company time zone.
- **Constraints**: foreign keys, unique indexes on business keys (for example document number per company), check constraints for status values.
- **Indexes**: on foreign keys, document dates, status, and `(CompanyId, ...)` composite for filtered queries.
- **Migrations**: EF Core migrations only; seed data (roles, default chart of accounts, leave types) via idempotent seeders.
- **Transactions**: one transaction per command; no cross-module distributed transactions (use events and eventual consistency, or the outbox pattern).
- **Backups**: daily full, frequent log backups; restore tested.

---

## 13. API Requirements

- RESTful, versioned (`/api/v1/...`), JSON.
- Standard responses: `200/201/204`, `400` (validation), `401`, `403`, `404`, `409` (concurrency/business conflict), `500` as `ProblemDetails`.
- Pagination, sorting and filtering on all list endpoints (`page`, `pageSize`, `sort`, `filter`).
- `Idempotency-Key` header supported on critical POST endpoints (payments, payroll run).
- Controllers are thin: they only map HTTP to a Cortex.Mediator command/query and return the result.
- Rate limiting (global and per tenant) and CORS configured by environment.
- Tenant resolved from the token/subdomain (see 4.6); clients never send `TenantId` in request bodies.

Example endpoints:
```
POST /api/v1/procurement/purchase-orders
POST /api/v1/procurement/purchase-orders/{id}/approve
POST /api/v1/inventory/goods-receipts
POST /api/v1/payroll/runs/{id}/approve
GET  /api/v1/accounting/reports/trial-balance?periodId=...
```

---

## 14. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Typical API response < 500 ms (p95) for CRUD; reports < 5 s for 1M ledger rows with proper indexing; payroll for 1,000 employees < 2 min |
| Scalability | Sized for SME (see A8): single instance first; stateless API so it can scale out later; indexes and pagination first, read replicas only if proven necessary |
| Availability | Target 99.5% in business hours (adjust per hosting) |
| Security | OWASP Top 10 compliance; HTTPS only; hashed passwords; secrets in Key Vault/user-secrets; least privilege DB account |
| Privacy | Encrypt sensitive columns (national ID, bank account); salary data restricted by role |
| Auditability | Every create/update/delete/approval logged with user, time, old/new values |
| Reliability | Optimistic concurrency; idempotent background jobs; retry with backoff for transient SQL errors |
| Maintainability | Architecture tests enforce layer rules; code coverage >= 70% on Domain and Application |
| Observability | Structured logs, correlation IDs, health endpoints, optional OpenTelemetry traces |
| Usability | Responsive UI, keyboard-friendly data entry, consistent validation messages |
| Localization | Resource-based text (English v1, Bangla later), lakh/crore formatting, multi-currency, country packs |
| Multi-tenancy | Zero cross-tenant data access (verified by automated tests); per-tenant limits; tenant-level backup/export |
| Portability | Same build runs on IIS and in Docker/Azure using configuration only |
| Compliance | Configurable statutory rules; data retention policy configurable |

---

## 15. Security and Access Control

- Authentication: ASP.NET Core Identity, JWT access tokens (short-lived) with refresh tokens.
- Authorization: permission strings such as `Accounting.Journal.Post`, `Payroll.Run.Approve`, enforced in `AuthorizationBehavior` and policies.
- **Segregation of duties**: creator cannot approve own document; payroll preparer differs from approver.
- Default roles: Admin, Accountant, Finance Manager, HR Manager, Payroll Officer, Store Keeper, Procurement Officer, Approver, Employee, Auditor.
- Account lockout, password complexity, optional MFA, session/token revocation.
- Tenant isolation enforced at four levels: token claim, EF global filter, SQL Row-Level Security, automated tests.
- Platform super-admin actions (including impersonation) are separately logged and time-limited.
- Input validation on all commands; parameterized queries only; protection against over-posting by using DTOs.

---

## 16. Logging and Monitoring (Serilog)

- Minimum levels: `Information` for application, `Warning` for Microsoft/System namespaces.
- Sinks: Console (dev), rolling File (JSON), SQL Server or Seq (central).
- Enrichers: `CorrelationId`, `TenantId`, `UserId`, `CompanyId`, `RequestPath`, `MachineName`.
- On IIS: rolling file sink to a configurable folder outside the web root; on Azure/Docker: console sink plus Application Insights/Seq.
- `UseSerilogRequestLogging` for HTTP summary logs.
- `LoggingBehavior` logs request name, duration and outcome; failed validations logged at `Warning`, unhandled exceptions at `Error`.
- **Never log**: passwords, tokens, full bank/ID numbers, individual salary figures.
- Log retention: 30 days hot, 12 months archived (configurable).

---

## 17. Testing Strategy

| Level | Scope | Tools |
|---|---|---|
| Unit | Domain rules, handlers, validators, calculators (payroll, costing, depreciation) | xUnit, FluentAssertions, Moq |
| Integration | Handlers plus real SQL Server, EF mappings, migrations, repositories | Testcontainers, Respawn |
| API / Contract | Endpoint behavior, auth, status codes | `WebApplicationFactory` |
| Architecture | Layer dependency rules, naming conventions, no cross-module references | NetArchTest |
| End-to-end scenarios | Procure-to-Pay, Hire-to-Pay, Stock-to-Ledger | Automated scenario tests |
| Tenant isolation | Prove tenant A cannot read/write tenant B data across API, jobs, cache and files | Integration tests with two seeded tenants |
| Frontend | Component and end-to-end tests for key flows (login, PO approval, payroll run) | Vitest, Playwright |
| Non-functional | Load test on posting, FIFO issue and reports, with many tenants | k6 or NBomber |

---

## 18. Deployment and DevOps

### 18.1 Environments
Development, Test/UAT, Production.

### 18.2 Phase 1: IIS
- Windows Server with IIS and the ASP.NET Core Hosting Bundle; **in-process hosting**; one app pool per site (No Managed Code).
- **One IIS site** (`ERP.Api`) serves both the API (`/api/*`) and the React build (static files from `wwwroot` with `MapFallbackToFile("index.html")` for SPA routes). Recommended for SME because it means one deployment, one TLS certificate, one app pool, no CORS, and the refresh-token cookie stays same-origin.
- Caching rules: hashed JS/CSS files cached long-term; `index.html` served with `no-cache` so new releases appear immediately.
- Wildcard or per-tenant subdomain bindings and TLS certificates.
- Move to a separate static host only when there is a reason (CDN speed for distant users, independent frontend releases, or Azure move). The code supports both because the API base URL is a build/runtime setting.
- SQL Server on the same network (VM or dedicated server); least-privilege application login; scheduled backups (daily full, frequent log).
- Serilog rolling files outside the web root; log retention policy; Hangfire with SQL storage.
- Data Protection keys stored in the database; `IFileStorage` pointing to a protected network/local folder.
- Deploy via CI-produced package (Web Deploy or zip) with `app_offline.htm` during updates; EF migrations applied as reviewed SQL scripts.

### 18.3 Phase 2: Azure and Docker
- Multi-stage `Dockerfile` for the API (non-root user) and a static-file container (or Azure Static Web Apps/CDN) for the React build; `docker-compose` for local development (API, React dev server, SQL Server, Seq).
- Azure targets: App Service for Containers or Container Apps; Azure SQL (elastic pool for dedicated-database tenants); Key Vault for secrets; Blob Storage for `IFileStorage`; Azure Cache for Redis; Application Insights/OpenTelemetry; Front Door/WAF with custom tenant domains.
- Images stored in a container registry; blue/green or rolling deployments with health-check gates.

### 18.4 CI/CD
- CI: restore, build, unit, integration, architecture and tenant-isolation tests, static analysis, frontend build and tests, publish artifacts/images.
- CD: apply migrations (script-based, per tenant database where applicable), deploy, run smoke tests.
- Configuration through environment variables and secret stores; no secrets in source control.
- Branching: trunk-based or GitFlow; mandatory pull-request review.

---

## 19. Delivery Roadmap

| Phase | Deliverables |
|---|---|
| **Phase 0: Foundation** | Solution skeleton (one project per module and layer), Cortex.Mediator behaviors, Serilog, **multi-tenancy (catalog, resolution, filters, RLS, isolation tests)**, Identity/RBAC, audit, company/branch, numbering, approval engine, React app skeleton (login, onboarding, tenant admin, PDF printing), country-pack abstraction, IIS deployment pipeline, CI |
| **Phase 1: Accounting core** | Chart of accounts (Bangladesh template), fiscal periods, journals, posting-rule engine, trial balance, GL, VAT/TDS basics |
| **Phase 2: Inventory + Procurement** | Items, warehouses, PR -> PO -> GRN -> Invoice, stock ledger, **FIFO cost layers**, AP, VDS, accounting integration |
| **Phase 3: HR** | Employees, org structure, attendance, leave, self-service |
| **Phase 4: Payroll** | Salary structures, payroll run, payslips, Bangladesh tax/festival bonus/PF rules, bank/MFS files, accounting integration |
| **Phase 5: Hardening** | Reports, bank reconciliation, fixed assets, performance tuning, security and tenant-isolation review, UAT, documentation |
| **Phase 6: Cloud and Language** | Docker images, Azure deployment, Key Vault/Blob integration, dedicated-DB tenant option, **Bangla UI**, second country pack (proof of internationalization) |

---

## 20. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Scope too large for the timeline | High | Strict MoSCoW priorities; deliver vertical slices per phase |
| Statutory rules vary or change | High | Data-driven rules with effective dates |
| Inventory valuation errors | High | Immutable stock ledger, deterministic costing, extensive unit tests |
| Cross-module coupling | Medium | Integration events, architecture tests |
| Performance of ledger/reports | Medium | Proper indexing, read models, pagination, Dapper for reports |
| Payroll data leakage | High | Field-level permissions, encryption, audit, log redaction |
| Cross-tenant data leak | Critical | Fail-closed tenant resolution, global filters, SQL RLS, automated isolation tests, code review checklist |
| FIFO performance under heavy issue volume | Medium | Index layers by (Tenant, Item, Warehouse, ReceiptDate), short transactions, row locks on layers, load tests |
| Bangladesh rules change (NBR, labour) | High | Effective-dated data, country pack versioning, update without redeploy |
| IIS-to-Azure migration surprises | Medium | Portability rules in 4.8 enforced from Phase 0; Docker build in CI early |

---

## 21. Acceptance Criteria (Definition of Done)

- All **Must** requirements implemented and demonstrated on seed data.
- Procure-to-Pay, Hire-to-Pay and Stock-to-Ledger scenarios pass end-to-end and produce balanced journals.
- Trial balance always balances; sub-ledgers (AP, Inventory) reconcile to control accounts.
- Architecture tests pass (no dependency rule violations).
- Unit and integration tests pass in CI; coverage target met.
- Audit trail shows every create/update/approve action.
- No secrets or sensitive data appear in logs.
- Two-tenant isolation test suite passes.
- FIFO scenario tests pass (partial issues across layers, returns, transfers, landed cost) and valuation equals the inventory control account.
- Bangladesh pack scenarios pass: VAT/VDS posting, salary tax and festival bonus calculation, BEFTN/MFS file generation.
- The same build deploys to IIS and runs in a Docker container with configuration changes only.
- Swagger documentation is complete and accurate.

---

## 22. Glossary

| Term | Meaning |
|---|---|
| ERP | Enterprise Resource Planning |
| CQRS | Command Query Responsibility Segregation |
| GL | General Ledger |
| AP / AR | Accounts Payable / Accounts Receivable |
| PR / PO / RFQ | Purchase Requisition / Purchase Order / Request for Quotation |
| GRN | Goods Receipt Note |
| GRNI | Goods Received Not Invoiced (accrual account) |
| 3-way match | Matching PO, GRN and vendor invoice before payment |
| COGS | Cost of Goods Sold |
| Cortex.Mediator | Mediator library implementing commands, queries, notifications and pipeline behaviors |
| Serilog | Structured logging library for .NET |
| Outbox pattern | Storing events in the same transaction as data for reliable publishing |
| Tenant | A customer organization using the SaaS platform; owns companies and branches |
| RLS | SQL Server Row-Level Security |
| FIFO | First-In, First-Out inventory costing using cost layers |
| Country pack | Plug-in with country-specific tax, payroll, leave, COA and report rules |
| NBR | National Board of Revenue (Bangladesh) |
| Mushak | NBR VAT forms/registers (invoice, books, return) |
| VDS / TDS / AIT | VAT deducted at source / Tax deducted at source / Advance income tax |
| TIN / BIN | Taxpayer / Business Identification Number |
| BEFTN | Bangladesh Electronic Funds Transfer Network |
| MFS | Mobile financial services (bKash, Nagad, Rocket) |
| WPPF | Workers' Profit Participation Fund |
| BFRS/IFRS | Bangladesh / International Financial Reporting Standards |

---

## 23. Appendix: Suggested Architecture Decision Records (ADRs)

1. ADR-001: Modular monolith with Clean Architecture (one project per module and layer)
2. ADR-002: CQRS using Cortex.Mediator and pipeline behaviors
3. ADR-003: One SQL Server database with schema-per-module
4. ADR-004: Integration events between modules (and optional outbox)
5. ADR-005: Inventory costing with FIFO cost layers behind `ICostingStrategy`
6. ADR-006: Payroll rule engine design (data-driven, effective-dated)
7. ADR-007: Authentication and permission model
8. ADR-008: Multi-tenancy (pooled with RLS, optional dedicated DB)
9. ADR-009: React SPA for all UI (login, admin, printing); MVC only as a fallback
10. ADR-010: Hosting portability (IIS now, Azure/Docker later)
11. ADR-011: Country-pack plug-in design for Bangladesh and international expansion
