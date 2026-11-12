# Master Product Requirements Document (PRD)

## Integrated ERP Platform: Accounting, HR, Payroll, Inventory, Procurement

| Field | Value |
|---|---|
| Document version | 0.1 (Draft) |
| Status | For review |
| Owner | Software Architecture |
| Last updated | 2026-11-09 |
| Tech stack | ASP.NET Core, SQL Server, Clean Architecture, Cortex.Mediator, Serilog |

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
- G5. Support multi-company and multi-branch operation from day one.

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

## 2. Assumptions and Open Questions

### 2.1 Assumptions
- A1. Single deployable solution, **modular monolith** first, with clear module boundaries so modules can later be split into services.
- A2. Web application: ASP.NET Core Web API backend plus a separate SPA or MVC frontend (to be confirmed).
- A3. One SQL Server database with one schema per module (`acc`, `hr`, `pay`, `inv`, `proc`, `common`).
- A4. Multi-currency is supported; one base currency per company.
- A5. English UI in v1; localization-ready (resource files).
- A6. Statutory rules (tax, social contributions) are **configurable**, not hard-coded, because they differ by country.

### 2.2 Open Questions (to confirm with the stakeholder/teacher)
1. Which country/jurisdiction rules apply to payroll tax and VAT/GST?
2. Is the frontend in scope (Blazor, MVC, Angular, React), or is this API-only?
3. Single-tenant or multi-tenant SaaS? (This PRD assumes multi-company within one tenant.)
4. Which inventory costing methods are required: weighted average, FIFO, or both?
5. Are batch/serial/expiry tracking required?
6. Is integration with an external bank or attendance device required?
7. Hosting target: on-premises IIS, Azure, Docker/Kubernetes?
8. Expected scale: number of users, employees, transactions per day?

---

## 3. Technology Stack

| Concern | Choice |
|---|---|
| Runtime / Framework | .NET (current LTS) and ASP.NET Core Web API |
| Architecture | Clean Architecture, CQRS, modular monolith |
| Mediator / CQRS | **Cortex.Mediator** (commands, queries, notifications, pipeline behaviors) |
| ORM (writes) | Entity Framework Core (SQL Server provider) |
| Reads | EF Core with `AsNoTracking`, or Dapper for heavy reports |
| Database | Microsoft SQL Server |
| Logging | **Serilog** (structured logging) with sinks: Console, File, SQL Server or Seq |
| Validation | FluentValidation (executed in a Cortex.Mediator pipeline behavior) |
| Mapping | Mapster or AutoMapper (explicit mappings preferred) |
| Auth | ASP.NET Core Identity plus JWT bearer tokens and refresh tokens |
| Background jobs | Hangfire or Quartz.NET (payroll runs, scheduled reports) |
| API docs | Swagger / OpenAPI |
| Caching | `IMemoryCache`, optional Redis for scale-out |
| Testing | xUnit, FluentAssertions, Moq/NSubstitute, Testcontainers (SQL Server), Respawn |
| Reporting | QuestPDF/ClosedXML for PDF/Excel export |
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

### 4.2 Solution Structure

```
src/
  BuildingBlocks/
    ERP.SharedKernel/            # Base Entity, AggregateRoot, ValueObject, Result<T>, IDomainEvent
  Modules/
    Accounting/
      ERP.Accounting.Domain/
      ERP.Accounting.Application/
      ERP.Accounting.Infrastructure/
    HR/            (same 3 projects)
    Payroll/       (same 3 projects)
    Inventory/     (same 3 projects)
    Procurement/   (same 3 projects)
    Common/        # Auth, Company, Branch, Currency, Audit, Settings, Notifications
  ERP.Api/                       # Presentation layer and composition root
tests/
  ERP.UnitTests/
  ERP.IntegrationTests/
  ERP.ArchitectureTests/         # NetArchTest rules enforcing layer dependencies
docs/
  prd.md
  adr/                           # Architecture Decision Records
```

> A simpler alternative for a student project is one `Domain`, `Application`, `Infrastructure`, `Api` project with **feature folders per module** inside each. Decide with the teacher (see Open Questions).

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
- **Multi-company filter**: EF Core global query filter on `CompanyId`.
- **Health checks** for database and background jobs.

---

## 5. Common / Platform Requirements

| ID | Requirement | Priority |
|---|---|---|
| CMN-01 | Company, Branch and Department master data | Must |
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
| INV-09 | **Valuation**: weighted average cost (required); FIFO (optional/configurable) | Must |
| INV-10 | **Stock ledger** (immutable movement log) and real-time on-hand, reserved, available quantities | Must |
| INV-11 | **Batch/lot and serial tracking**, expiry dates | Could |
| INV-12 | **Reorder alerts** and suggested purchase requisitions | Should |
| INV-13 | **Return to vendor** and **return from department** | Should |
| INV-14 | **Prevent negative stock** (configurable per warehouse) | Must |
| INV-15 | **Accounting integration**: each stock movement publishes an event; Accounting posts per item category mapping | Must |
| INV-16 | **Reports**: stock on hand, stock ledger/card, valuation, ageing, slow-moving, movement summary, reorder report | Must |

### 9.3 Key Entities
`Item`, `ItemCategory`, `UnitOfMeasure`, `UomConversion`, `Warehouse`, `Location`, `StockLedgerEntry`, `StockBalance`, `GoodsReceipt`, `GoodsIssue`, `StockTransfer`, `StockAdjustment`, `StockCount`, `Batch`, `SerialNumber`.

### 9.4 Business Rules
- BR-INV-1: Stock quantity changes **only** through posted stock documents; direct edits are not allowed.
- BR-INV-2: Posted stock documents are immutable; corrections via reversing documents.
- BR-INV-3: Valuation is recalculated deterministically on each receipt (weighted average) and stored on each ledger entry.
- BR-INV-4: Stock postings are concurrency-safe (optimistic concurrency or `UPDLOCK` on balance rows).

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
Goods issue / adjustment / transfer -> StockLedgerEntry -> event
   -> Accounting posts per category mapping (Dr COGS/Expense, Cr Inventory)
```

---

## 12. Data and Database Requirements (SQL Server)

- **Schemas** per module: `common`, `acc`, `hr`, `pay`, `inv`, `proc`.
- **Primary keys**: `bigint IDENTITY` for transactional tables or `uniqueidentifier` (sequential) if distributed generation is needed. Pick one convention and apply it everywhere.
- **Standard columns** on all entities: `Id`, `CompanyId`, `CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy`, `IsDeleted`, `RowVersion`.
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
- Rate limiting and CORS configured by environment.

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
| Scalability | Stateless API for horizontal scale; read-heavy reports may use read replicas later |
| Availability | Target 99.5% in business hours (adjust per hosting) |
| Security | OWASP Top 10 compliance; HTTPS only; hashed passwords; secrets in Key Vault/user-secrets; least privilege DB account |
| Privacy | Encrypt sensitive columns (national ID, bank account); salary data restricted by role |
| Auditability | Every create/update/delete/approval logged with user, time, old/new values |
| Reliability | Optimistic concurrency; idempotent background jobs; retry with backoff for transient SQL errors |
| Maintainability | Architecture tests enforce layer rules; code coverage >= 70% on Domain and Application |
| Observability | Structured logs, correlation IDs, health endpoints, optional OpenTelemetry traces |
| Usability | Responsive UI, keyboard-friendly data entry, consistent validation messages |
| Localization | Resource-based text, culture-aware number/date formats, multi-currency |
| Compliance | Configurable statutory rules; data retention policy configurable |

---

## 15. Security and Access Control

- Authentication: ASP.NET Core Identity, JWT access tokens (short-lived) with refresh tokens.
- Authorization: permission strings such as `Accounting.Journal.Post`, `Payroll.Run.Approve`, enforced in `AuthorizationBehavior` and policies.
- **Segregation of duties**: creator cannot approve own document; payroll preparer differs from approver.
- Default roles: Admin, Accountant, Finance Manager, HR Manager, Payroll Officer, Store Keeper, Procurement Officer, Approver, Employee, Auditor.
- Account lockout, password complexity, optional MFA, session/token revocation.
- Input validation on all commands; parameterized queries only; protection against over-posting by using DTOs.

---

## 16. Logging and Monitoring (Serilog)

- Minimum levels: `Information` for application, `Warning` for Microsoft/System namespaces.
- Sinks: Console (dev), rolling File (JSON), SQL Server or Seq (central).
- Enrichers: `CorrelationId`, `UserId`, `CompanyId`, `RequestPath`, `MachineName`.
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
| Non-functional | Load test on posting and reports | k6 or NBomber |

---

## 18. Deployment and DevOps

- Environments: Development, Test/UAT, Production.
- Docker image for the API; `docker-compose` for local (API plus SQL Server plus Seq).
- CI pipeline: restore, build, test, architecture tests, static analysis, publish image.
- CD pipeline: apply EF migrations (script-based), deploy, run smoke tests.
- Configuration through environment variables and secrets store; no secrets in source control.
- Branching: trunk-based or GitFlow; mandatory pull-request review.

---

## 19. Delivery Roadmap

| Phase | Deliverables |
|---|---|
| **Phase 0: Foundation** | Solution skeleton, Clean Architecture projects, Cortex.Mediator behaviors, Serilog, Identity/RBAC, audit, company/branch, numbering, approval engine, CI |
| **Phase 1: Accounting core** | Chart of accounts, fiscal periods, journals, posting-rule engine, trial balance, GL |
| **Phase 2: Inventory + Procurement** | Items, warehouses, PR -> PO -> GRN -> Invoice, stock ledger, valuation, AP, accounting integration |
| **Phase 3: HR** | Employees, org structure, attendance, leave, self-service |
| **Phase 4: Payroll** | Salary structures, payroll run, payslips, statutory rules, accounting integration |
| **Phase 5: Hardening** | Reports, bank reconciliation, fixed assets, performance tuning, security review, UAT, documentation |

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

---

## 21. Acceptance Criteria (Definition of Done)

- All **Must** requirements implemented and demonstrated on seed data.
- Procure-to-Pay, Hire-to-Pay and Stock-to-Ledger scenarios pass end-to-end and produce balanced journals.
- Trial balance always balances; sub-ledgers (AP, Inventory) reconcile to control accounts.
- Architecture tests pass (no dependency rule violations).
- Unit and integration tests pass in CI; coverage target met.
- Audit trail shows every create/update/approve action.
- No secrets or sensitive data appear in logs.
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

---

## 23. Appendix: Suggested Architecture Decision Records (ADRs)

1. ADR-001: Modular monolith with Clean Architecture
2. ADR-002: CQRS using Cortex.Mediator and pipeline behaviors
3. ADR-003: One SQL Server database with schema-per-module
4. ADR-004: Integration events between modules (and optional outbox)
5. ADR-005: Inventory costing method (weighted average first)
6. ADR-006: Payroll rule engine design (data-driven, effective-dated)
7. ADR-007: Authentication and permission model
