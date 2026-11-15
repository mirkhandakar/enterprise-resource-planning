# Accounting Module PRD

**Product:** Integrated Multi-Tenant ERP Platform (SaaS)
**Module:** Accounting (bounded context `Accounting`, schema `acc`)
**Parent document:** `prd.md` (Master PRD v0.5). This file refines master requirements ACC-01 to ACC-17, BD-03 to BD-06, BD-14 and CMN-16.

| Field | Value |
|---|---|
| Version | 1.0 (Draft) |
| Status | For review |
| Authors | Software Architecture, UI/UX Design |
| Last updated | 2026-11-15 |
| Stack | ASP.NET Core Web API (MVC controllers), Cortex.Mediator, EF Core, SQL Server, Serilog, React + TypeScript + MUI |
| Architecture | Modular monolith, Clean Architecture inside the module |
| Target | Small and medium businesses, Bangladesh first |

---

## 1. Purpose, Scope and Success Criteria

### 1.1 Purpose
Accounting is the **financial system of record**. It owns the general ledger, receivables and payables sub-ledgers, cash and bank, tax, fixed assets, budgets and financial reporting. It receives business events from Inventory, Procurement, Payroll and HR and turns them into balanced, auditable journal entries.

### 1.2 Product Goals
1. **Trustworthy numbers:** every posted amount balances, is traceable to its source document and cannot be silently changed.
2. **Automatic by default:** operational documents post to the ledger without re-keying; accountants handle exceptions, not routine entries.
3. **Fast month-end:** a guided close that surfaces every blocker in one place.
4. **Bangladesh-ready:** BDT, July-June fiscal year, VAT/VDS/TDS, Mushak documents, BFRS/IFRS-aligned statements.
5. **Pleasant to use all day:** keyboard-first data entry, clear drill-down, calm and legible UI.

### 1.3 In Scope (v1)
Chart of accounts, fiscal calendar, vouchers/journals, posting engine, AP, AR (basic), cash and bank with reconciliation, multi-currency, tax (VAT, VDS, TDS/AIT), fixed assets, budgets, period/year close, financial and management reports, dimensions (cost center, branch, project), approvals and attachments.

### 1.4 Out of Scope (v1)
Full sales/CRM, payment gateway integration, direct NBR e-filing, consolidation across tenants, inter-company eliminations (basic inter-company later), bank feed integration (statement import by file only), advanced cost accounting/allocations.

### 1.5 Success Metrics

| Metric | Target |
|---|---|
| Posting correctness | 100% of posted entries balance (enforced by DB constraint and domain rule) |
| Sub-ledger reconciliation | AP, AR, Inventory control accounts equal their sub-ledgers at all times |
| Automatic posting rate | >= 95% of operational documents post with no human intervention |
| Month-end close time (SME) | <= 3 working days after go-live maturity |
| Manual voucher entry | 2-line voucher posted in <= 30 seconds using keyboard only |
| Trial balance (100k lines) | < 3 seconds |
| Usability | >= 80% task success and SUS score >= 75 in usability testing with 5 accountants |

---

## 2. Users and Jobs-to-be-Done

| Persona | Context | Top jobs-to-be-done |
|---|---|---|
| **Accountant** (primary) | Daily data entry and reconciliation | "Record and post vouchers quickly", "Fix posting exceptions", "Reconcile the bank", "Match payments to bills" |
| **Finance Manager / CFO** | Review and control | "See cash and profit at a glance", "Approve and lock periods", "Understand any number by drilling down" |
| **Business Owner (SME)** | Non-accountant | "Am I making money? Do I have cash? Who owes me?" in simple dashboards |
| **Tax Officer** | VAT and TDS compliance | "Prepare the monthly VAT return worksheet", "Track VDS/TDS deposits" |
| **Auditor** (read-only) | Verification | "Trace any balance to the source document and see who changed what" |
| **Approver** | Occasional, often on a phone | "Approve or reject a voucher/payment in under 20 seconds" |

---

## 3. Module Architecture (Modular Monolith)

### 3.1 Bounded Context and Data Ownership
- The Accounting module **owns** schema `acc` and all tables in it. No other module reads or writes `acc` tables.
- Accounting **does not** read other modules' tables. It learns about the outside world only through **integration events** and **contracts** from `ERP.Contracts`.
- Foreign keys across modules do not exist. References to external documents are stored as a **soft reference**: `(SourceModule, SourceDocumentType, SourceDocumentId, SourceDocumentNo)`.
- Party data (vendors) lives in Procurement. Accounting keeps a small **local read model** `acc.Party` fed by `VendorCreated/Updated` events (id, name, TIN, BIN, status). **Customers** are owned by Accounting in v1 (no Sales module yet) and can later move behind a Sales contract.

### 3.2 Projects (one per layer)

```
Modules/Accounting/
  ERP.Accounting.Domain/          # Aggregates, value objects, domain events, domain services, invariants
  ERP.Accounting.Application/     # Commands, queries, handlers, validators, DTOs, ports, event handlers
  ERP.Accounting.Infrastructure/  # AccountingDbContext, EF configs, migrations, repositories, report SQL, PDF templates
BuildingBlocks/ERP.Contracts/
  Accounting/                     # Events and interfaces Accounting PUBLISHES (JournalPosted, PeriodClosed ...)
  Inventory/ Procurement/ Payroll/ HR/   # Events Accounting CONSUMES (read-only contracts)
```

Allowed references: `Domain` -> SharedKernel only. `Application` -> own Domain, SharedKernel, Contracts. `Infrastructure` -> own Application and Domain. Enforced by architecture tests (NetArchTest): "Accounting.* must not reference Inventory.*, Procurement.*, Payroll.*, HR.*".

### 3.3 Internal Layering and Flow

```
Controller (ERP.Api)  -> Cortex.Mediator ICommand/IQuery
  -> pipeline: Logging -> Authorization -> Validation -> Transaction -> Performance
  -> Handler (Application) -> Aggregate (Domain) -> Repository (Infrastructure) -> SQL Server
  -> Domain events -> Outbox -> Integration events (other modules)
```

Controllers stay thin. Business rules live in the Domain (for example `JournalEntry.Post()` verifies balance, open period and account validity).

### 3.4 Integration Strategy

| Decision | Choice | Reason |
|---|---|---|
| Inbound posting from other modules | **Integration events (Cortex.Mediator notifications) + Inbox/Outbox tables**, processed asynchronously within about 2 seconds | Loose coupling; module can later become a service |
| Idempotency | Unique key `(TenantId, SourceModule, SourceDocumentType, SourceDocumentId, SourceVersion)` on `acc.PostingLog` | Replays never double-post |
| Failure handling | Failed postings go to **Posting Exceptions** queue with reason, payload and a one-click **Retry** after the fix (for example missing account mapping or closed period) | Accountants see and fix problems; nothing is lost silently |
| Reverse flow | Accounting publishes `JournalPosted`, `PaymentPosted`, `PeriodClosed` so Procurement can mark invoices paid and others can block back-dated documents | Business state stays consistent |
| Consistency | Eventual consistency between modules; strong consistency inside Accounting (one DB transaction per command) | Right trade-off for a modular monolith |
| Reconciliation | Nightly job compares sub-ledgers (AP, AR, Inventory valuation from event totals) with control accounts and raises alerts | Detects drift early |

### 3.5 Integration Contracts

**Published by Accounting (`ERP.Contracts.Accounting`)**
```csharp
public sealed record JournalPosted(Guid TenantId, long JournalId, string VoucherNo, DateOnly PostingDate,
    string SourceModule, string SourceDocumentType, Guid? SourceDocumentId);
public sealed record PaymentPosted(Guid TenantId, Guid PaymentId, Guid PartyId, decimal Amount, string CurrencyCode,
    IReadOnlyList<PaymentAllocationInfo> Allocations);
public sealed record PeriodClosed(Guid TenantId, Guid CompanyId, int FiscalYear, int PeriodNo, bool HardLocked);
```

**Consumed by Accounting**

| Event (publisher) | Accounting reaction |
|---|---|
| `GoodsReceived` (Inventory) | Dr Inventory, Cr GRNI at FIFO layer cost |
| `GoodsIssued` (Inventory) | Dr COGS or Expense (by issue purpose/cost center), Cr Inventory at consumed FIFO layer cost |
| `StockAdjusted`, `StockTransferred` (Inventory) | Dr/Cr Inventory vs Adjustment account; transfers post only if warehouses map to different branches/companies |
| `LandedCostAllocated` (Inventory) | Dr Inventory (remaining layers) and Dr Cost Variance (consumed part), Cr Landed Cost Clearing |
| `VendorInvoiceApproved` (Procurement) | Dr GRNI/Expense (+ input VAT), Cr Accounts Payable, apply VDS/TDS withholding lines |
| `PurchaseReturnPosted` (Procurement) | Reverse of the above (debit note) |
| `PayrollApproved` (Payroll) | Dr Salary Expense by department/cost center, Cr Net Salary Payable, Cr Tax/PF/Other deductions payable, Dr/Cr employer contributions |
| `PayrollPaid` (Payroll) | Dr Net Salary Payable, Cr Bank/Cash |
| `VendorCreated/Updated` (Procurement) | Upsert `acc.Party` |

Every event carries: `TenantId`, `CompanyId`, `EventId`, `OccurredAtUtc`, `SourceDocumentId/No/Date`, currency, and **pre-computed amounts and dimension tags**. Accounting never calls back into the publisher to ask for data.

### 3.6 Architecture Decision Records (Accounting)
- ACC-ADR-001: Immutable journals; correction by reversal only.
- ACC-ADR-002: Event-driven posting with outbox/inbox and exceptions queue.
- ACC-ADR-003: Posting rules as data (account mappings), not code.
- ACC-ADR-004: Period balances table maintained transactionally for fast trial balance.
- ACC-ADR-005: Money stored as `decimal(18,4)` with explicit currency and rate; rounding handled by a dedicated rounding line.
- ACC-ADR-006: Local `acc.Party` read model instead of cross-module joins.

---

## 4. Domain Model

### 4.1 Aggregates

| Aggregate root | Contains | Key invariants |
|---|---|---|
| `Account` | Code, name, type, normal balance, parent, flags (control, bank, tax, posting allowed), currency restriction, dimension requirements | Code unique per company; only leaf accounts accept postings; type cannot change once used; control accounts accept only system postings unless permitted |
| `FiscalYear` | `FiscalPeriod[]` | Periods contiguous, no gaps/overlaps; exactly one open year per company for posting by default |
| `JournalEntry` | `JournalLine[]`, status, voucher type, source reference | Debits = credits (document and base currency); >= 2 lines; each line has exactly one of debit/credit > 0; posting date in open period; posted entry is immutable |
| `Payment` (Payment/Receipt voucher) | `PaymentAllocation[]` | Allocations <= payment amount; cannot over-allocate an open item |
| `OpenItem` | Party, document, due date, original and outstanding amount | Outstanding between 0 and original |
| `BankReconciliation` | Statement lines, matches | Reconciled difference must be zero to finalize |
| `FixedAsset` | Cost, residual, life, method, `DepreciationEntry[]` | Depreciation never exceeds depreciable base; disposal closes the asset |
| `TaxCode` | Rate, type (VAT, SD, VDS, TDS), accounts, effective dates | Rates effective-dated, non-overlapping |
| `Budget` | Lines per account/dimension/period | One active budget version per year and scenario |
| `PostingRule` / `AccountMapping` | Source type, conditions, line templates | A rule must produce balanced lines for every sample payload (validated on save) |

### 4.2 Value Objects
`Money(Amount, CurrencyCode)`, `ExchangeRate(From, To, Rate, Date)`, `AccountCode`, `VoucherNumber`, `DimensionSet(CostCenterId, BranchId, ProjectId, PartyId)`, `DateRange`.

### 4.3 State Machines

**Journal / Voucher**
```
Draft -> PendingApproval -> Approved -> Posted -> (Reversed by a new Reversal entry)
Draft -> Cancelled        PendingApproval -> Rejected -> Draft
```
Only `Posted` entries affect balances. `Posted` and `Reversed` entries are never edited or deleted.

**Fiscal Period:** `Open -> SoftClosed (accountants only) -> HardLocked`. Reopening a hard-locked period needs the `Accounting.Period.Reopen` permission and writes an audit record with a mandatory reason.

**Bank Reconciliation:** `InProgress -> Finalized` (reopen allowed for the latest only).

### 4.4 Voucher Types
`JV` Journal, `PV` Payment, `RV` Receipt, `CV` Contra (bank/cash transfer), `PB` Purchase Bill (system), `SI` Sales Invoice, `DN` Debit Note, `CN` Credit Note, `OB` Opening Balance, `DP` Depreciation, `CL` Closing, `RJ` Reversal Journal. Numbering: `{Type}-{Branch}-{FY}-{Sequence}`, **gapless per type, company and fiscal year**, allocated at posting time from `acc.DocumentSequence` under an update lock.

### 4.5 Calculation Rules
- Amounts stored to 4 decimals; shown to 2.
- Base amount = `Round(Amount x Rate, 4)` per line; any difference after summing lines goes to the configured **Rounding Difference** account as a system line (maximum tolerance configurable, default 1.00 BDT).
- Unrealized FX gain/loss computed at period end on open monetary items; realized FX computed at settlement.

---

## 5. Functional Requirements

Priority: **M** Must, **S** Should, **C** Could. "Master" shows the traced master requirement.

### 5.1 Setup and Chart of Accounts

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-001 | Create/edit/deactivate accounts in a **hierarchical tree** (up to 6 levels) with code, name, type (Asset, Liability, Equity, Income, Expense), normal balance, parent | M | ACC-01 |
| ACC-FR-002 | Account flags: posting allowed (leaf), control account, bank/cash, tax account, requires cost center, requires party, allowed currency | M | ACC-01 |
| ACC-FR-003 | Load a **Bangladesh COA template** (BFRS/IFRS-aligned) at tenant provisioning; tenant may edit afterwards | M | ACC-17, BD-14 |
| ACC-FR-004 | Import/export COA via Excel with validation preview before commit | S | CMN-11 |
| ACC-FR-005 | Block deactivation or deletion of accounts with posted balances (deactivate only if balance is zero) | M | ACC-01 |
| ACC-FR-006 | **Account mappings** screen for system accounts (AP control, AR control, Inventory, GRNI, COGS, Adjustment, Salary expense/payable, VAT input/output, VDS/TDS payable, Rounding, FX gain/loss, Retained earnings, Cost variance) with health check "all required mappings present" | M | ACC-04 |
| ACC-FR-007 | Dimensions: cost centers (tree), branches, projects; mark which accounts require which dimension | S | ACC-06 |
| ACC-FR-008 | Company settings: base currency, fiscal year start, decimal places, rounding tolerance, voucher approval thresholds, back-dating days allowed | M | CMN-10 |

### 5.2 Fiscal Calendar and Period Control

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-010 | Create fiscal years with 12 periods (default 1 July to 30 June, configurable); optional adjustment period 13 | M | ACC-02, BD-03 |
| ACC-FR-011 | Period status Open / Soft-closed / Hard-locked with role-based override | M | ACC-02 |
| ACC-FR-012 | Posting blocked outside open periods; the UI explains why and offers a link to the period screen if permitted | M | ACC-02 |
| ACC-FR-013 | Publish `PeriodClosed` so other modules can block back-dated documents | M | ACC-02 |

### 5.3 Vouchers and Journals

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-020 | Create manual vouchers (JV, PV, RV, CV) as draft; save, edit, delete drafts; post when balanced | M | ACC-03 |
| ACC-FR-021 | Live **balance indicator** (total debit, total credit, difference) and a disabled Post button until the difference is zero | M | ACC-03 |
| ACC-FR-022 | Line fields: account, description, debit, credit, party (when required), cost center, branch, project, tax code, currency/rate (when foreign) | M | ACC-03 |
| ACC-FR-023 | **Recurring journals** (schedule, auto-draft or auto-post) and **reversing journals** (auto-reverse on first day of next period) | S | ACC-03 |
| ACC-FR-024 | **Copy voucher**, **save as template**, **paste lines from Excel** | S | ACC-03 |
| ACC-FR-025 | Approval workflow by amount, voucher type, and company; maker cannot approve own voucher | M | CMN-04 |
| ACC-FR-026 | Posted vouchers are immutable; **Reverse** action creates a linked reversal entry with a mandatory reason | M | ACC-15 |
| ACC-FR-027 | Attachments (PDF, image, Excel) on vouchers; view inline | S | CMN-08 |
| ACC-FR-028 | Opening balance entry (by account and by party) with tie-out check against previous system | M | ACC-03 |
| ACC-FR-029 | Back-dating limit per role (for example accountant 7 days, manager unlimited within open periods) | S | ACC-02 |
| ACC-FR-030 | Voucher list with filters (date, type, status, account, party, amount range, text), saved views, export | M | ACC-14 |

### 5.4 Automated Posting Engine

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-040 | Consume integration events (3.5) and create posted journals using the **posting rule + account mapping** catalogue (section 7) | M | ACC-04 |
| ACC-FR-041 | **Idempotent**: replaying an event never creates a second entry (PostingLog unique key) | M | ACC-04 |
| ACC-FR-042 | **Posting Exceptions** queue: list failures with reason (missing mapping, closed period, inactive account, unbalanced rule), full payload, Retry and Retry-all after fixing | M | ACC-04 |
| ACC-FR-043 | Every auto-journal stores source module, document type/id/no and a deep link back to the source document | M | ACC-04 |
| ACC-FR-044 | Rule simulator: choose a sample event and preview the generated lines before saving a rule change | S | ACC-04 |
| ACC-FR-045 | Source-document **update or cancellation** events create a reversal and a new entry (versioned via `SourceVersion`), never an edit | M | ACC-15 |
| ACC-FR-046 | Dashboard tile "Unposted documents" and "Posting exceptions" with counts and age | M | ACC-04 |

### 5.5 Accounts Payable

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-050 | Open items created from vendor invoices/debit notes (via events) and manual vendor bills for non-procurement expenses | M | ACC-07 |
| ACC-FR-051 | **Payment vouchers** to one party with multiple allocations; partial payments; advance payments and later allocation | M | ACC-07 |
| ACC-FR-052 | **Payment proposal**: list due and overdue bills by due date and discount, select, generate payment batch | S | ACC-07 |
| ACC-FR-053 | Withholding (VDS/TDS) applied on bill and/or payment per tax code; liability posted automatically | M | PRC-16 |
| ACC-FR-054 | Vendor ledger and **AP aging** (current, 1-30, 31-60, 61-90, 90+) with drill-down to documents | M | ACC-14 |
| ACC-FR-055 | Cheque register: issue, print (PDF), clear, void, stale-cheque alert | S | ACC-09 |
| ACC-FR-056 | Debit notes and vendor refund handling | S | ACC-07 |

### 5.6 Accounts Receivable (basic)

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-060 | Customer master (name, TIN/BIN, terms, credit limit, contacts) | S | ACC-08 |
| ACC-FR-061 | Manual **sales invoices** with tax and Mushak-format PDF; credit notes | S | ACC-08, BD-05 |
| ACC-FR-062 | **Receipt vouchers** with allocation, advances, partial receipts | S | ACC-08 |
| ACC-FR-063 | Customer ledger and AR aging; dunning list (overdue reminders by email) | S | ACC-08 |
| ACC-FR-064 | Interface prepared for a future Sales module: `SalesInvoicePosted` event contract defined, no code dependency | C | ACC-08 |

### 5.7 Cash and Bank

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-070 | Bank and cash accounts (linked to GL accounts), opening balance, currency, signatories | M | ACC-09 |
| ACC-FR-071 | Contra vouchers (bank-to-bank, cash-to-bank) | M | ACC-09 |
| ACC-FR-072 | **Bank statement import** (CSV/Excel with column mapping, saved per bank) | M | ACC-09 |
| ACC-FR-073 | **Reconciliation workspace**: auto-match by amount/date/reference with confidence score, manual match (1-to-1, 1-to-many, many-to-1), create missing voucher from an unmatched statement line (bank charge, interest) | M | ACC-09 |
| ACC-FR-074 | Reconciliation finalize only when the difference is zero; produce a signed reconciliation report (PDF) | M | ACC-09 |
| ACC-FR-075 | Cash book and bank book with running balance; petty cash imprest workflow | S | ACC-09 |
| ACC-FR-076 | Cash flow forecast (due AP, expected AR, scheduled payroll) for 30/60/90 days | C | ACC-09 |

### 5.8 Multi-Currency

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-080 | Currencies and daily exchange rates (manual entry and file import); rate type (buying, selling, average) | M | ACC-05 |
| ACC-FR-081 | Foreign-currency vouchers store amount, currency, rate and base amount | M | ACC-05 |
| ACC-FR-082 | Realized FX gain/loss at settlement; unrealized revaluation at period end (auto-reversing journal) | S | ACC-05 |
| ACC-FR-083 | Warn when no rate exists for the voucher date and offer the latest rate | M | ACC-05 |

### 5.9 Tax (Bangladesh Pack)

> All rates and rules are effective-dated data to be verified against current NBR publications before go-live (master BD-04 to BD-06).

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-090 | Tax codes: VAT (standard, reduced, zero-rated, exempt, truncated), supplementary duty, VDS, TDS/AIT; each with input/output/payable accounts and effective dates | M | ACC-10, BD-04 |
| ACC-FR-091 | Input VAT on purchases, output VAT on sales, VDS withheld from vendor payments, TDS/AIT withheld per payment type | M | ACC-16 |
| ACC-FR-092 | **Mushak documents and registers in the exact NBR-prescribed layout and fields** (for example tax invoice/challan Mushak-6.3, purchase book 6.1, sales book 6.2, monthly return worksheet 9.1), as PDF/Excel; templates versioned; validated against current NBR forms before go-live | M | BD-05 |
| ACC-FR-093 | Tax deposit tracking: challan number, bank, date, treasury reference; link deposit vouchers to liabilities | M | BD-06 |
| ACC-FR-094 | **VAT dashboard**: output minus input, VDS credit, payable/carry-forward, filing due date reminder | S | ACC-10 |
| ACC-FR-095 | TIN/BIN validation on parties (configurable patterns) | S | BD-13 |
| ACC-FR-096 | TDS/VDS certificate generation for vendors | S | BD-06 |

### 5.10 Fixed Assets

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-100 | Asset categories with default accounts, life, method; asset register (tag, location, custodian, acquisition cost, date, residual) | S | ACC-11 |
| ACC-FR-101 | Create asset from a purchase (manual link to the vendor bill reference) or opening balance | S | ACC-11 |
| ACC-FR-102 | Depreciation methods: straight-line and declining balance; monthly **depreciation run** with preview, then post `DP` voucher | S | ACC-11 |
| ACC-FR-103 | Disposal/write-off with gain/loss calculation and posting | S | ACC-11 |
| ACC-FR-104 | Asset register, depreciation schedule and movement reports | S | ACC-14 |

### 5.11 Budgeting

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-110 | Annual budget by account, cost center and month; copy from last year with percentage uplift; Excel import | C | ACC-12 |
| ACC-FR-111 | Budget vs actual report with variance and percent; budget check API used by Procurement at approval time (`ICheckBudget` contract) | C | PRC-12 |

### 5.12 Period-End and Year-End Close

| ID | Requirement | Pri | Master |
|---|---|---|---|
| ACC-FR-120 | **Month-end close checklist** with live status: unposted documents, posting exceptions, unapproved vouchers, bank reconciliations pending, depreciation run done, payroll posted, FX revaluation done, VAT worksheet prepared | M | ACC-13 |
| ACC-FR-121 | Soft-close then hard-lock actions with approver and timestamp | M | ACC-13 |
| ACC-FR-122 | **Year-end wizard**: verify checklist, post closing entries (income/expense to retained earnings), generate opening balances for the new year, lock the closed year | M | ACC-13 |
| ACC-FR-123 | Re-open closing: only latest year-end, with reason and full audit trail | S | ACC-13 |

### 5.13 Reports and Analytics

All reports: filter by company, branch, cost center, project, date range, currency; **drill-down** (report line -> ledger -> voucher -> source document); export PDF/Excel; saved filters; scheduled email (S).

| ID | Report | Pri |
|---|---|---|
| ACC-FR-130 | Trial Balance (opening, movement, closing; with/without zero balances; by level) | M |
| ACC-FR-131 | General Ledger and Account Ledger (running balance) | M |
| ACC-FR-132 | Day Book, Cash Book, Bank Book | M |
| ACC-FR-133 | Income Statement (P&L) with comparative period and percent of revenue | M |
| ACC-FR-134 | Balance Sheet (classified; comparative) | M |
| ACC-FR-135 | Cash Flow Statement (indirect method) | S |
| ACC-FR-136 | AP Aging, AR Aging, Party Ledger, Statement of Account | M |
| ACC-FR-137 | VAT, VDS, TDS reports and Mushak documents | M |
| ACC-FR-138 | Budget vs Actual | C |
| ACC-FR-139 | Fixed Asset Register and Depreciation Schedule | S |
| ACC-FR-140 | Financial statement designer: map accounts to statement lines (BFRS/IFRS presentation) | S |
| ACC-FR-141 | Audit reports: voucher change history, reversals, period reopen log, user activity | M |
| ACC-FR-142 | Owner dashboard: cash, revenue vs expense trend, receivables/payables due, top expenses, VAT payable | M |

### 5.14 Key Acceptance Criteria (Given / When / Then)

**AC-1 Balanced posting**
- Given a draft voucher with debit 10,000 and credit 9,999.50, when the user clicks Post, then the system blocks posting and shows "Out of balance by 0.50" next to the totals.

**AC-2 Closed period**
- Given period July is hard-locked, when an event with posting date 15 July arrives, then no journal is created, an exception "Period closed" appears in Posting Exceptions, and after a permitted user reopens or re-dates and clicks Retry the journal posts once.

**AC-3 Idempotency**
- Given `GoodsReceived` with id E1 was posted, when E1 is delivered again, then no new journal is created and the log records "duplicate ignored".

**AC-4 Reversal**
- Given a posted voucher JV-2026-0045, when a user with permission reverses it with a reason, then a new voucher RJ-... posts with swapped debit/credit, both are linked, and the original shows status "Reversed".

**AC-5 Allocation**
- Given bill A (50,000) and bill B (30,000), when a payment of 60,000 is allocated 50,000 to A and 10,000 to B, then A is closed, B has 20,000 outstanding, and AP aging reflects it immediately.

**AC-6 Bank reconciliation**
- Given a statement with 120 lines, when the user runs auto-match, then matched lines are shown with confidence, unmatched lines are listed, and Finalize stays disabled until the difference is 0.00.

**AC-7 Tenant isolation**
- Given tenants A and B, when a user from A requests any journal id belonging to B, then the response is 404 and the attempt is logged.

---

## 6. Permissions

| Permission | Accountant | Fin. Manager | Tax Officer | Approver | Auditor | Owner |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| `Accounting.Account.Manage` | - | Yes | - | - | - | - |
| `Accounting.Voucher.Create/Edit` | Yes | Yes | Limited (tax) | - | - | - |
| `Accounting.Voucher.Approve` | - | Yes | - | Yes | - | Yes |
| `Accounting.Voucher.Post` | Yes (under threshold) | Yes | - | - | - | - |
| `Accounting.Voucher.Reverse` | - | Yes | - | - | - | - |
| `Accounting.Period.Close` | - | Yes | - | - | - | - |
| `Accounting.Period.Reopen` | - | Yes (with reason) | - | - | - | - |
| `Accounting.Bank.Reconcile` | Yes | Yes | - | - | - | - |
| `Accounting.Payment.Create` | Yes | Yes | - | - | - | - |
| `Accounting.Payment.Approve` | - | Yes | - | Yes | - | Yes |
| `Accounting.Tax.Manage` | - | Yes | Yes | - | - | - |
| `Accounting.PostingRule.Manage` | - | Yes | - | - | - | - |
| `Accounting.Report.View` | Yes | Yes | Yes | Limited | Yes | Yes |
| `Accounting.Audit.View` | - | Yes | - | - | Yes | - |

Segregation of duties: maker != checker; the person who creates a payment cannot approve it; the person who reconciles a bank account cannot be the sole approver of its payments (configurable).

---

## 7. Posting Rule Catalogue (Default Mappings)

Account keys are resolved through `acc.AccountMapping` (by key, and optionally by item category, branch or tax code). Amounts come from the event payload.

| # | Source event | Debit | Credit | Notes |
|---|---|---|---|---|
| P1 | GoodsReceived (against PO) | Inventory (by item category) | GRNI | Amount = FIFO layer cost |
| P2 | VendorInvoiceApproved (stock items) | GRNI (+ price variance if any) ; Input VAT | Accounts Payable (party) ; VDS Payable | Variance to Purchase Price Variance account |
| P3 | VendorInvoiceApproved (services/expenses) | Expense (by line account/cost center) ; Input VAT | Accounts Payable ; VDS/TDS Payable | No GRN |
| P4 | PurchaseReturnPosted | Accounts Payable ; | GRNI/Inventory ; Input VAT | Reverses P1/P2 proportionally |
| P5 | GoodsIssued (department use) | Expense or COGS (by issue purpose, cost center) | Inventory | Amount = consumed FIFO layers |
| P6 | StockAdjusted (increase) | Inventory | Inventory Adjustment (gain) | |
| P7 | StockAdjusted (decrease) | Inventory Adjustment (loss) | Inventory | |
| P8 | LandedCostAllocated | Inventory (remaining qty share) ; Cost Variance (consumed share) | Landed Cost Clearing / AP | |
| P9 | PayrollApproved | Salary Expense (by cost center) ; Employer PF/Contribution Expense | Net Salary Payable ; Income Tax Payable ; PF Payable ; Other Deductions Payable ; Employer Contribution Payable | One entry per payroll run, lines per component and cost center |
| P10 | PayrollPaid | Net Salary Payable | Bank / Cash | |
| P11 | Payment voucher (manual) | Accounts Payable (party) ; Bank charges | Bank / Cash ; VDS/TDS Payable | Allocation to open items |
| P12 | Receipt voucher (manual) | Bank / Cash ; TDS Receivable | Accounts Receivable | |
| P13 | Sales invoice | Accounts Receivable | Revenue ; Output VAT | |
| P14 | Depreciation run | Depreciation Expense | Accumulated Depreciation | |
| P15 | Asset disposal | Bank/Receivable ; Accumulated Depreciation ; Loss | Asset cost ; Gain | |
| P16 | FX revaluation | FX Unrealized Loss / Monetary account | Monetary account / FX Unrealized Gain | Auto-reversing next period |
| P17 | Year-end close | Income accounts | Retained Earnings (and Expense accounts credited) | Net result to Retained Earnings |

Every rule is validated: sum(debits) = sum(credits) for the sample payload before activation.

---

## 8. UI/UX Design

> Design owner: UI/UX. Stack: React + TypeScript + **MUI** + MUI X Data Grid. English in v1; text externalized for Bangla later.

### 8.1 Design Principles
1. **Trust first.** Show balances, status, and source everywhere. Never hide a number's origin. Every figure is clickable to drill down.
2. **Keyboard-first, mouse-friendly.** Accountants live in data entry. Every common action has a shortcut and a logical Tab order.
3. **Calm density.** Dense enough for work, quiet enough for eight hours. Neutral palette, one accent, generous whitespace around dense tables.
4. **Prevent, then explain.** Block errors before posting (balance indicator, period check). When blocked, say why in plain language and offer the fix.
5. **Progressive disclosure.** Simple default views for owners; advanced fields (dimensions, FX, tax) appear when needed or enabled.
6. **Consistent patterns.** Same list/filter/detail/action layout in every module so users learn once.
7. **Accessible by default.** WCAG 2.2 AA; no information conveyed by color alone.

### 8.2 Information Architecture (Left Navigation)

```
Accounting
  Dashboard
  Transactions
    Vouchers (all)   Journal   Payment   Receipt   Contra
    Recurring & Templates
  Payables            Bills, Payments, Vendor Ledger, Aging
  Receivables         Invoices, Receipts, Customer Ledger, Aging
  Cash & Bank         Accounts, Statements, Reconciliation, Cheques
  Tax                 VAT, VDS/TDS, Mushak, Deposits
  Fixed Assets        Register, Depreciation, Disposals
  Budgets
  Reports             Financial, Ledgers, Tax, Management
  Period Close        Checklist, Periods, Year-end
  Posting Center      Exceptions, Posting Log, Rules & Mappings
  Setup               Chart of Accounts, Fiscal Years, Dimensions, Currencies & Rates, Tax Codes, Settings
```
Global: company/branch switcher (top bar), universal search (Ctrl+K: voucher no, party, account, amount), notifications (approvals, exceptions), user menu.

### 8.3 Visual System (MUI Theme Tokens)

| Token | Value / Rule |
|---|---|
| Typeface | Inter (fallback Roboto); **tabular numerals** (`font-variant-numeric: tabular-nums`) for all amounts |
| Base size | 14 px body, 13 px in compact grids, 12 px captions; line-height 1.4 |
| Primary | Deep blue `#1F4FD8` (actions, selection, links) |
| Neutrals | Slate scale `#0F172A` text, `#475569` secondary, `#E2E8F0` borders, `#F8FAFC` canvas |
| Semantic | Success `#15803D`, Warning `#B45309`, Error `#B91C1C`, Info `#0369A1` (each paired with an icon and text) |
| Debit/Credit | **Neutral colors**, never red/green (debit is not "bad"). Negative balances shown in parentheses `(1,250.00)` with optional red |
| Spacing | 4 px grid; page padding 24 px; card padding 16 px |
| Radius / elevation | 8 px radius; flat cards with 1 px borders; shadow only for overlays |
| Density | User toggle: Comfortable (row 44 px) / Compact (row 32 px); default Compact for grids |
| Dark mode | Supported via theme tokens (later phase); all contrast ratios maintained |
| Motion | 150 to 200 ms, reduced-motion respected |

Number rules: right-aligned; thousands grouping per locale (lakh/crore `12,34,567.00` when the company chooses it); currency code shown only for foreign amounts; zero shown as `-` in reports.

### 8.4 Key Screens and Wireframes

#### 8.4.1 Accounting Dashboard (Owner and Finance Manager)
```
+--------------------------------------------------------------------------------+
| Accounting > Dashboard             [Company: ABC Ltd v] [Period: Nov 2026 v] [?]|
+--------------------------------------------------------------------------------+
| Cash & Bank        | Receivables Due    | Payables Due       | VAT Payable      |
| 4,820,500          | 1,230,000 (3 late) | 2,110,000 (2 late) | 312,400          |
+--------------------------------------------------------------------------------+
| Revenue vs Expense (12 months, bar+line)          | Needs your attention        |
| [chart]                                           | ! 4 posting exceptions      |
|                                                   | ! 2 vouchers to approve     |
|                                                   | ! Bank rec pending: DBBL    |
+---------------------------------------------------+-----------------------------+
| Top expenses (this month)   [bar list]            | Recent vouchers  [table]    |
+--------------------------------------------------------------------------------+
```
Behavior: tiles are links; "Needs your attention" is a prioritized to-do list (the app's most important element); skeleton loaders; empty state explains the first action.

#### 8.4.2 Voucher / Journal Entry (most-used screen)
```
+--------------------------------------------------------------------------------+
| Journal Voucher  JV-(auto)   Status: Draft   [Save draft F8] [Post F9] [...]   |
+--------------------------------------------------------------------------------+
| Date [15 Nov 2026]  Branch [Dhaka v]  Currency [BDT v]  Ref [__________]       |
| Narration [_______________________________________________________________]   |
+--------------------------------------------------------------------------------+
| # | Account (type to search)    | Description | Party | Cost ctr |  Debit  | Credit |
| 1 | 1210 Bank - DBBL            |             |       |          | 50,000  |        |
| 2 | 4100 Service Revenue        |             |       | Sales    |         | 50,000 |
| + add line (Enter on last row)                                                  |
+--------------------------------------------------------------------------------+
| Total Debit 50,000.00   Total Credit 50,000.00   Difference 0.00  [OK Balanced] |
+--------------------------------------------------------------------------------+
| Attachments [drop files]    Approval trail    Audit history    Source link     |
+--------------------------------------------------------------------------------+
```
Behavior:
- **Enter** moves to the next cell; Enter on the last cell adds a line; **Ctrl+D** duplicates the line; **Alt+Up/Down** reorders; **Ctrl+Z** undo within the draft.
- Typing in Account searches by code or name with fuzzy match and shows type and balance in the dropdown.
- If the account requires party/cost center, the cell is highlighted and a hint appears (not just an error after posting).
- The difference chip is amber when unbalanced and offers **"Balance with this account"** (adds the offsetting amount to a chosen line).
- Posted vouchers open read-only with a **Reverse** action; edits are impossible, and the page says so in plain language.
- Paste a block of rows from Excel into the grid; invalid rows are flagged before import.

#### 8.4.3 Chart of Accounts
```
+---------------------------+--------------------------------------------------+
| Search / filter [type v]  |  Account 1210 - Bank - DBBL                        |
| v 1 Assets                |  Type: Asset   Normal: Debit   Parent: 12 Bank     |
|   v 12 Bank & Cash        |  [x] Bank account  [ ] Control  [x] Posting allowed |
|     1210 Bank - DBBL  <   |  Requires: [ ] Cost center  [ ] Party               |
|     1220 Cash in Hand     |  Balance (YTD): 4,820,500    [View ledger]           |
| > 2 Liabilities           |  Usage: 142 postings   [Deactivate disabled: has bal]|
| > 3 Equity ...            |                                                   |
+---------------------------+--------------------------------------------------+
```
Master-detail with tree on the left; drag to reorganize; inline validation; explains disabled actions ("Cannot deactivate: balance is not zero").

#### 8.4.4 Bank Reconciliation Workspace
```
+--------------------------------------------------------------------------------+
| Reconcile: DBBL Current A/c   Statement: 01-30 Nov 2026     [Auto-match] [Import]|
+----------------------------------------+---------------------------------------+
| BANK STATEMENT (120)  Unmatched 14     | BOOKS (118)   Unmatched 12            |
| 02 Nov  Cheque 0045    -20,000  [ ]    | 02 Nov PV-0123 Rahim Traders -20,000 [ ]|
| 05 Nov  Deposit        +75,000  [ ]    | ...                                   |
| 09 Nov  Bank charge       -550  [+ Create voucher]                             |
+----------------------------------------+---------------------------------------+
| Statement balance 4,830,000  Books 4,820,500  Outstanding 9,500  Difference 0.00|
| [Match selected]  [Unmatch]                          [Finalize (disabled: 0.00 req)]|
+--------------------------------------------------------------------------------+
```
Behavior: two panes with synchronized selection; green check = matched (with icon, not only color); confidence badge on suggested matches; keyboard: Space select, M match, U unmatch; sticky totals footer always visible.

#### 8.4.5 Month-End Close Checklist
```
+--------------------------------------------------------------------------------+
| Close: November 2026                                  Progress 6 / 9 complete   |
+--------------------------------------------------------------------------------+
| [OK] Bank reconciliations finalized (3 of 3)                                    |
| [OK] Payroll posted                                                             |
| [!!] Posting exceptions: 4 open                      [Review exceptions ->]     |
| [!!] Vouchers pending approval: 2                    [Open approvals ->]        |
| [..] Depreciation run                                [Run now ->]               |
| [..] FX revaluation                                  [Preview ->]               |
| [OK] VAT worksheet prepared                                                     |
+--------------------------------------------------------------------------------+
| [Soft close period]   (enabled when all Must items are complete)               |
+--------------------------------------------------------------------------------+
```
Each row has status icon + text + deep link. Soft/hard close buttons use a confirmation dialog stating exactly what will be locked and for whom.

#### 8.4.6 Posting Exceptions (Posting Center)
Table: Date, Source (e.g. "Inventory > GRN-0087"), Reason (plain text: "No account mapped for item category 'Packaging'"), Age, Actions. Row action **Fix** opens the relevant setting in a side drawer (for example the mapping row pre-filled), then **Retry**. Bulk "Retry all fixed".

#### 8.4.7 Reports
Left filter panel (collapsible), central report, top actions (Export PDF/Excel, Print, Save view, Schedule). Statement-style layout for financial statements (indented hierarchy, subtotals bold, comparative column). Every amount is a link (drill to ledger, then voucher, then source). Sticky header and totals; "as of" timestamp displayed; empty state suggests changing filters.

### 8.5 Interaction Patterns

| Pattern | Standard |
|---|---|
| Lists | MUI X Data Grid: server-side pagination/sort/filter, column chooser, saved views, density toggle, pinned totals row, Excel export, row actions menu |
| Forms | Single column up to 2 columns on desktop; labels above fields; required fields marked; validation on blur and on submit; summary of errors at top with anchors |
| Draft safety | Autosave drafts every 30 s; "unsaved changes" guard; recover drafts after crash |
| Confirmations | Only for irreversible or high-impact actions (post, reverse, close, hard lock); dialog text states the consequence; destructive button is labeled with the verb ("Reverse voucher") |
| Feedback | Inline status chips (Draft, Pending, Posted, Reversed), toasts for success (4 s), persistent banners for blocking problems |
| Undo | Prefer reversible design: drafts are deletable; posted items are reversed, never "undone"; UI explains this once, with a link to "Why can't I edit a posted voucher?" |
| Search | Global Ctrl+K command palette; accounts and parties as async autocomplete with keyboard selection |
| Drill-down | Breadcrumb trail (Balance Sheet > Cash at Bank > Ledger > JV-0045 > Source), back keeps filters |
| Audit visibility | "History" tab on every voucher: who, when, what changed, approvals, source event |

### 8.6 States and Microcopy
- **Empty:** explain and offer the next step ("No vouchers yet. Create your first journal voucher" + button).
- **Loading:** skeletons matching layout; avoid spinners for tables.
- **Error:** plain, specific, actionable: "This date falls in a closed period (July 2026). Choose a date in November or ask a Finance Manager to reopen July." Never "Error 400".
- **Permission:** hide what users can never do; show disabled with tooltip for what they can do later ("Needs approval").
- **Tone:** calm, precise, no jargon where avoidable (offer an "Explain" tooltip for terms like GRNI).

### 8.7 Responsive and Device Strategy
- **Desktop (>= 1280 px):** primary experience, full grids and split panes.
- **Tablet (768 to 1279 px):** all screens usable; reconciliation panes stack as tabs.
- **Mobile (< 768 px):** read-only dashboards, search, voucher view, and **approvals** (approve/reject with reason). Data entry of vouchers is not optimized for phones in v1.

### 8.8 Accessibility (WCAG 2.2 AA)
Contrast >= 4.5:1 for text; focus ring always visible; full keyboard operation including grids; ARIA labels on amounts and status chips ("Debit 50,000 taka"); error messages linked to fields; no color-only meaning; screen-reader announcement when balance state changes; target size >= 24 px; respects reduced motion and 200% zoom.

### 8.9 Keyboard Shortcuts

| Keys | Action |
|---|---|
| Ctrl+K | Global search / command palette |
| Alt+N | New voucher (context type) |
| F8 / F9 | Save draft / Post |
| Enter / Shift+Enter | Next / previous cell |
| Ctrl+D | Duplicate line |
| Alt+Up / Alt+Down | Move line |
| Ctrl+Shift+V | Paste rows from Excel |
| / | Focus grid filter |
| ? | Show shortcut help |

### 8.10 UX Success Measures and Research Plan
- Usability tests (5 accountants, 3 owners) on: post a journal, pay two bills, reconcile a bank, close a month, find "why is cash 4,820,500".
- Targets: task success >= 90% for core tasks; median 2-line voucher <= 30 s; zero data-entry errors that reach posting; SUS >= 75.
- Instrumentation (privacy-safe): time on voucher form, validation errors by field, exception queue age, drill-down usage.
- Design deliverables: Figma library (MUI-based tokens), clickable prototype for 8.4 screens, content style guide, accessibility audit before release.

---

## 9. Data Design (SQL Server, schema `acc`)

### 9.1 Conventions
All tables: `Id`, `TenantId`, `CompanyId` (where applicable), `CreatedAt/By`, `UpdatedAt/By`, `RowVersion`; money `decimal(18,4)`; rates `decimal(18,8)`; dates `datetime2` (UTC) and `date` for posting/business dates; Row-Level Security on `TenantId`; unique indexes start with `TenantId`.

### 9.2 Main Tables

| Table | Purpose | Key columns |
|---|---|---|
| `Account` | Chart of accounts | Id, TenantId, CompanyId, Code, Name, Type, NormalBalance, ParentId, Level, IsPosting, IsControl, IsBank, RequiresCostCenter, RequiresParty, CurrencyCode, IsActive |
| `AccountMapping` | System account keys | Key, ScopeType (Company/ItemCategory/Branch/TaxCode), ScopeId, AccountId |
| `FiscalYear`, `FiscalPeriod` | Calendar | FiscalYear, PeriodNo, StartDate, EndDate, Status |
| `JournalEntry` | Voucher header | Id, VoucherType, VoucherNo, PostingDate, FiscalPeriodId, Status, CurrencyCode, ExchangeRate, Narration, TotalDebit, TotalCredit, SourceModule, SourceDocType, SourceDocId, SourceDocNo, SourceVersion, ReversesJournalId, ReversedByJournalId, PostedAt, PostedBy |
| `JournalLine` | Voucher line | JournalId, LineNo, AccountId, Debit, Credit, BaseDebit, BaseCredit, PartyId, CostCenterId, BranchId, ProjectId, TaxCodeId, Description |
| `PeriodBalance` | Fast TB | AccountId, FiscalPeriodId, DimensionKey, DebitTotal, CreditTotal (updated in the posting transaction) |
| `DocumentSequence` | Gapless numbering | VoucherType, CompanyId, FiscalYear, LastNo |
| `PostingLog` | Idempotency and trace | SourceModule, SourceDocType, SourceDocId, SourceVersion, JournalId, Status, ProcessedAt |
| `PostingException` | Failed postings | EventId, Reason, Payload (JSON), Status, FirstFailedAt, RetryCount |
| `InboxMessage`, `OutboxMessage` | Reliable messaging | MessageId, Type, Payload, ProcessedAt |
| `Party` | Vendor read model and customers | PartyType, ExternalId, Name, TIN, BIN, Status |
| `OpenItem`, `Allocation` | AP/AR | PartyId, DocType, DocId, DueDate, OriginalAmount, OutstandingAmount; PaymentId, OpenItemId, Amount |
| `Payment` | PV/RV | Header, BankAccountId, Amount, ChequeNo, ApprovalStatus |
| `BankAccount`, `BankStatement`, `BankStatementLine`, `BankMatch` | Cash and bank | Statement lines, match groups, reconciliation status |
| `TaxCode`, `TaxDeposit` | Tax | Rate, Type, Accounts, EffectiveFrom/To; ChallanNo, Date, Amount |
| `FixedAssetCategory`, `FixedAsset`, `DepreciationEntry` | Assets | Cost, Residual, LifeMonths, Method, AccumulatedDepreciation |
| `Budget`, `BudgetLine` | Budgeting | AccountId, CostCenterId, PeriodNo, Amount |
| `CostCenter`, `Project`, `Currency`, `ExchangeRate` | Dimensions and FX | |
| `ReportLine`, `ReportLineAccount` | Statement designer | Layout and account mapping |

### 9.3 Integrity Rules (Database Level)
```sql
-- A posted journal must balance (enforced also in the domain)
ALTER TABLE acc.JournalEntry ADD CONSTRAINT CK_Journal_Balanced
  CHECK (Status <> 'Posted' OR TotalDebit = TotalCredit);

-- A line has debit XOR credit, never negative
ALTER TABLE acc.JournalLine ADD CONSTRAINT CK_Line_DebitXorCredit
  CHECK ((Debit > 0 AND Credit = 0) OR (Credit > 0 AND Debit = 0));

-- Idempotent posting from events
CREATE UNIQUE INDEX UX_PostingLog_Source
  ON acc.PostingLog (TenantId, SourceModule, SourceDocType, SourceDocId, SourceVersion);

-- Business-key uniqueness per tenant and company
CREATE UNIQUE INDEX UX_Account_Code ON acc.Account (TenantId, CompanyId, Code);
CREATE UNIQUE INDEX UX_Journal_VoucherNo ON acc.JournalEntry (TenantId, CompanyId, VoucherType, VoucherNo);
```
Posted rows are protected from `UPDATE/DELETE` by a trigger or application-role permissions; only reversal inserts are allowed.

### 9.4 Indexing and Performance
- `JournalLine (TenantId, AccountId, PostingDate)` include amounts for ledger queries.
- `JournalEntry (TenantId, CompanyId, PostingDate, Status)`.
- `OpenItem (TenantId, PartyId, DueDate) WHERE OutstandingAmount > 0` (filtered).
- Heavy reports (TB, BS, P&L) read `PeriodBalance`; ledgers read `JournalLine` with keyset pagination; heavy exports run as background jobs.

### 9.5 Concurrency
- Voucher number allocation: `UPDLOCK` on the `DocumentSequence` row inside the posting transaction.
- Optimistic concurrency (`RowVersion`) on drafts, accounts, rules.
- Period status change takes an exclusive lock on the period row; posting takes a shared check.

---

## 10. API Design (ERP.Api, controllers + Cortex.Mediator)

Base: `/api/v1/accounting`. JSON, JWT, tenant from token. Errors as RFC 7807 `ProblemDetails` with a stable `code` (for example `ACC.PERIOD_CLOSED`) so React can show specific guidance. `Idempotency-Key` header on POST commands that move money.

| Area | Endpoints (examples) |
|---|---|
| Accounts | `GET /accounts`, `GET /accounts/tree`, `POST /accounts`, `PUT /accounts/{id}`, `POST /accounts/{id}/deactivate`, `POST /accounts/import` |
| Mappings | `GET/PUT /account-mappings`, `GET /account-mappings/health` |
| Periods | `GET /fiscal-years`, `POST /fiscal-years`, `POST /periods/{id}/soft-close`, `/hard-lock`, `/reopen` |
| Vouchers | `GET /vouchers`, `GET /vouchers/{id}`, `POST /vouchers` (draft), `PUT /vouchers/{id}`, `POST /vouchers/{id}/submit`, `/approve`, `/reject`, `/post`, `/reverse` |
| Posting center | `GET /posting/exceptions`, `POST /posting/exceptions/{id}/retry`, `POST /posting/exceptions/retry-all`, `GET /posting/log`, `GET/PUT /posting/rules`, `POST /posting/rules/simulate` |
| AP / AR | `GET /open-items`, `POST /payments`, `POST /payments/{id}/allocate`, `GET /parties/{id}/ledger`, `GET /reports/aging` |
| Bank | `GET /bank-accounts`, `POST /bank-statements/import`, `POST /reconciliations/{id}/auto-match`, `POST /reconciliations/{id}/match`, `/finalize` |
| Tax | `GET/POST /tax-codes`, `POST /tax-deposits`, `GET /tax/vat-summary`, `GET /tax/mushak/{type}?period=` (PDF/Excel) |
| Assets | `GET/POST /fixed-assets`, `POST /depreciation/preview`, `POST /depreciation/post`, `POST /fixed-assets/{id}/dispose` |
| Close | `GET /close/checklist?periodId=`, `POST /close/year-end` |
| Reports | `GET /reports/trial-balance`, `/general-ledger`, `/income-statement`, `/balance-sheet`, `/cash-flow`, `/day-book` (filters as query; `format=pdf|xlsx` for export) |
| Dashboard | `GET /dashboard/summary` |

Cortex.Mediator naming: `PostVoucherCommand`, `ReverseVoucherCommand`, `RetryPostingExceptionCommand`, `GetTrialBalanceQuery`, `GetCloseChecklistQuery`; each with a validator and, for commands, `TransactionBehavior`.

---

## 11. Non-Functional Requirements (Accounting-Specific)

| Area | Requirement |
|---|---|
| Performance | Post voucher p95 < 400 ms; auto-posting from event < 2 s end to end; TB for 100k lines < 3 s; ledger page (50 rows) < 500 ms; P&L/BS < 3 s |
| Integrity | No update/delete on posted data; reconciliation job nightly; DB check constraints as a second safety net |
| Availability and recovery | RPO <= 15 min (log backups), RTO <= 4 h (SME target) |
| Security | Row-Level Security; permission checks in `AuthorizationBehavior`; sensitive exports audited; no financial data in logs |
| Auditability | Immutable audit trail on every command (who, when, before/after); voucher history tab |
| Scalability | Designed for SME volume (about 3,000 documents per tenant per day); background jobs for exports and depreciation |
| Maintainability | Architecture tests, Domain unit test coverage >= 85%, Application >= 75% |
| Localization | English v1, i18n ready, lakh/crore formats, BDT default, multi-currency |
| Compatibility | Latest two versions of Chrome, Edge, Firefox, Safari; IIS and Docker/Azure hosting without code change |

### 11.1 Observability (Serilog)
- Log each command: name, tenant, company, user, duration, outcome; **never** log amounts with party names together at `Information` level in shared sinks; redact bank numbers.
- Enrich with `TenantId`, `CompanyId`, `CorrelationId`, `JournalId`, `SourceDocumentId`.
- Metrics: postings per minute, exception queue size and age, outbox lag, TB query time, failed reconciliations. Alert when exception age exceeds 24 h or outbox lag exceeds 1 minute.

---

## 12. Testing Strategy

| Level | Focus |
|---|---|
| Domain unit tests | Journal balance, period rules, rounding, state machine, depreciation calculators, allocation math, FX |
| Property-based tests | "For any generated posting rule input, debits = credits"; "after any sequence of post/reverse operations the TB still balances" |
| Application tests | Handlers with in-memory fakes: permissions, validation, idempotency, event handling |
| Integration tests | EF mappings, constraints, RLS, numbering under concurrency (100 parallel posts produce gapless numbers), outbox/inbox |
| Contract tests | Event payloads between modules (`ERP.Contracts`) with schema/version checks |
| Scenario tests (end-to-end) | Procure-to-Pay, Hire-to-Pay, Stock-to-Ledger, bank reconciliation, month-end and year-end close |
| Golden dataset | A known set of transactions with expected TB/BS/P&L to catch regressions |
| UI tests | Playwright flows: post voucher with keyboard only, reconcile, close period; accessibility checks (axe) |
| Tenant isolation | Two tenants; attempt cross-tenant reads/writes through API, jobs and exports |
| Performance | Load tests with 1M lines, parallel posting, large reconciliation |

---

## 13. Delivery Plan for the Accounting Module

| Release | Scope | Exit criteria |
|---|---|---|
| **A1 Foundation** | Module skeleton, COA + BD template, mappings, fiscal calendar, voucher (JV) draft/post/reverse, numbering, audit, TB, GL, voucher UX (8.4.2) | AC-1, AC-4, AC-7 pass; TB reconciles on golden dataset |
| **A2 Posting engine** | Inbox/outbox, rules, exceptions center, consume Procurement/Inventory events (P1-P8), AP open items, payments and allocation, AP aging | AC-2, AC-3, AC-5 pass; Procure-to-Pay scenario green |
| **A3 Cash, tax, FX** | Bank accounts, statement import, reconciliation workspace, VAT/VDS/TDS, Mushak documents, multi-currency | AC-6 pass; Mushak outputs validated against NBR forms |
| **A4 Payroll, close, reports** | Payroll postings (P9-P10), month-end checklist, year-end wizard, P&L, BS, cash flow, dashboards | Hire-to-Pay scenario green; close in a pilot tenant |
| **A5 Assets, AR, budgets, polish** | Fixed assets, AR invoicing/receipts, budgets, scheduled reports, accessibility audit, performance tuning | Usability targets met (8.10); SUS >= 75 |

---

## 14. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Posting drift between operational modules and GL | High | Idempotent events, exception queue, nightly reconciliation, golden dataset |
| Wrong or missing account mappings | High | Mapping health check at setup, rule simulator, clear exception messages |
| NBR rule or form changes | High | Effective-dated tax data, versioned Mushak templates, validation before go-live |
| Period-lock race conditions | Medium | Locking strategy (9.5), integration tests for concurrent post vs close |
| Complexity overwhelming SME users | Medium | Progressive disclosure, owner dashboard, guided setup wizard, default COA |
| Performance of reports on pooled DB | Medium | `PeriodBalance`, indexes, keyset pagination, background exports, per-tenant limits |
| Scope creep (full AR/Sales) | Medium | AR kept basic; Sales contract defined but out of scope |

---

## 15. Definition of Done (Module)

- All **Must** requirements in section 5 implemented with passing acceptance criteria AC-1 to AC-7.
- Architecture tests prove Accounting references no other module.
- Posting catalogue P1 to P17 verified by scenario tests; sub-ledgers reconcile to control accounts.
- Every screen in 8.4 implemented, accessible (axe clean, keyboard-only walkthrough) and tested on supported browsers.
- Usability targets in 8.10 met with real accountants.
- Mushak outputs verified against current NBR forms.
- Serilog logs and metrics in 11.1 available; no sensitive data in logs.
- Documentation: API (Swagger), event contracts, posting catalogue, user guide and setup wizard help.

---

## 16. Open Questions

1. Should voucher numbers be strictly gapless for all types, or only for tax-relevant ones (sales invoices, debit/credit notes)? (Assumed: gapless for all vouchers.)
2. Is inter-branch accounting needed (branch-wise balance sheets) or only branch-wise P&L? (Assumed: branch is a dimension, balance sheet at company level.)
3. Required bank statement formats for the first banks (CSV columns) to prebuild importers.
4. Approval thresholds and who approves them in a typical SME (to ship sensible defaults).
5. Chart-of-accounts template: use a standard BFRS-based template or provide industry variants (trading, manufacturing, services)?

---

## 17. Glossary

| Term | Meaning |
|---|---|
| GL | General Ledger |
| Voucher | A journal entry document of a specific type (JV, PV, RV, CV...) |
| Control account | GL account summarizing a sub-ledger (AP, AR, Inventory) |
| GRNI | Goods Received Not Invoiced |
| Open item | An unpaid/unsettled invoice or credit |
| Allocation | Matching a payment or receipt to open items |
| Contra | Transfer between cash and bank accounts |
| VDS / TDS / AIT | VAT deducted at source / Tax deducted at source / Advance income tax |
| Mushak | NBR VAT forms and registers (Bangladesh) |
| Outbox / Inbox | Tables that make cross-module messaging reliable and idempotent |
| Posting rule | Data definition of how a business event turns into journal lines |
| Soft close / Hard lock | Period closed for most users / closed for everyone except explicit reopen |
