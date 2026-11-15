# Payroll Module PRD

**Product:** Integrated Multi-Tenant ERP Platform (SaaS)
**Module:** Payroll (bounded context `Payroll`, schema `pay`)
**Parent document:** `prd.md` (Master PRD v0.5). Refines master requirements PAY-01 to PAY-19, BD-07, BD-09, BD-10, BD-12, CMN-16. Sibling documents: `prd-hr.md`, `prd-accounting.md`.

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
Payroll turns **people, time and pay rules into correct, explainable, on-time salaries**. It owns salary components, salary structures, employee pay assignments, payroll runs, statutory deductions (income tax, provident fund), bonuses, overtime pay, loan recovery, final settlements, payslips and payment files. It consumes employee and attendance data from HR and publishes approved payroll to Accounting.

### 1.2 Product Goals
1. **Correct to the taka:** deterministic, reproducible calculations with a full audit trail.
2. **Explainable:** every number on every payslip can be traced step by step ("Why is my tax 4,200?").
3. **Fast, calm month-end:** guided run with automatic checks, variance review and one-click approval.
4. **Compliant, configurable:** Bangladesh income tax, provident fund, gratuity, festival bonus and overtime as **effective-dated data**, plus a country-pack design for other countries.
5. **Confidential:** salary data visible only to those who need it; every access to it is logged.
6. **Connected:** HR drives inputs; Accounting receives balanced postings; employees get payslips on their phone.

### 1.3 In Scope (v1)
Salary components and structures, employee salary assignment and revisions, payroll periods and runs (regular, supplementary, bonus, final settlement), attendance/leave/overtime-driven pay, income tax (projection method), provident fund, gratuity accrual, festival bonus, loan and advance recovery, arrears and one-time items, payslips (PDF, self-service), bank/MFS payment files, salary hold, payroll reports and tax statements, approval workflow, Accounting integration.

### 1.4 Out of Scope (v1)
Direct bank API payments, direct NBR e-filing, benefits and insurance administration, expense reimbursement claims (planned later), time-and-attendance capture (HR), multi-country payroll in one run, WPPF automatic profit calculation (pool amount entered manually), custom pay cycles other than calendar month (see open questions).

### 1.5 Success Metrics

| Metric | Target |
|---|---|
| Calculation accuracy vs golden cases | 100% |
| Reproducibility | Re-running a run from its snapshot gives byte-identical results (hash match) |
| Run compute time | 500 employees < 60 s; 1,000 employees < 2 min |
| Payroll Officer effort | Monthly run for 500 employees in <= half a working day (excluding approvals) |
| Posting to Accounting | 100% of approved runs post a balanced journal automatically |
| Payslip availability | Published to employees within 5 minutes of approval/payment release |
| "Why did my pay change?" support tickets | Reduce by >= 60% through built-in explanations |
| Usability | Task success >= 90%, SUS >= 75 (Payroll Officers and employees) |

---

## 2. Users and Jobs-to-be-Done

| Persona | Context | Top jobs-to-be-done |
|---|---|---|
| **Payroll Officer** (primary) | Desktop, month-end | "Prepare the run without surprises", "Fix issues before approval", "Generate bank files and payslips" |
| **Payroll Manager / Finance Manager** | Review | "Spot errors fast (variance)", "Approve with confidence", "Control salary revisions" |
| **Business Owner (SME)** | Phone, brief | "See total payroll cost and approve in under a minute" |
| **Employee** | Phone | "See my payslip", "Understand my deductions and tax", "Download tax certificate" |
| **HR Manager** | Inputs | "Make sure joiners, leavers and attendance reach Payroll correctly" |
| **Tax/Compliance Officer** | Filing | "Get tax deduction and PF statements" |
| **Auditor** (read-only) | Verification | "Trace any amount to inputs and rules; see who approved" |

---

## 3. Module Architecture (Modular Monolith)

### 3.1 Bounded Context and Data Ownership
- Payroll **owns** schema `pay`. No other module reads or writes `pay` tables; Payroll reads no other module's tables.
- Cross-module references are **soft references** (IDs, codes), never foreign keys.
- Payroll keeps **local read models**: `pay.EmployeeRef` (built from HR events: id, number, name, department, branch, designation, grade, join/exit dates, bank/mobile wallet), `pay.DepartmentRef`, `pay.BranchRef`, `pay.AttendanceSummary` (versioned snapshots from HR).
- Salary data lives **only** in Payroll. HR has no salary fields.

| Data | Owner | Payroll's access |
|---|---|---|
| Employees, attendance, leave, loans (terms and schedule) | HR | Events into local read models |
| Departments, branches | Common | Events into local read models |
| Chart of accounts, journals, accounting periods | Accounting | Payroll publishes **logical account keys** (for example `SALARY_EXPENSE`), never account ids; consumes `JournalPosted` and `PeriodClosed` |
| Salary structure, pay assignment, runs, results, payslips, PF/gratuity ledgers | **Payroll** | Owner |
| Users and roles | Common | Permission checks only |

### 3.2 Projects (one per layer)

```
Modules/Payroll/
  ERP.Payroll.Domain/          # Aggregates, value objects, calculation domain services, formula AST, domain events
  ERP.Payroll.Application/     # Commands, queries, handlers, validators, DTOs, ports (IStatutoryCalculator, IPaymentFileFormatter), event handlers
  ERP.Payroll.Infrastructure/  # PayrollDbContext, EF configs, migrations, repositories, formula evaluator, PDF payslips, bank-file formatters, Hangfire jobs
CountryPacks/
  ERP.CountryPack.Bangladesh/  # BangladeshSalaryTaxCalculator, PF/gratuity rule sets, bank advice (BEFTN) and MFS formatters, defaults
BuildingBlocks/ERP.Contracts/
  Payroll/    # Events Payroll publishes
  HR/         # EmployeeJoined, EmployeeChanged, EmployeeSeparated, AttendanceFinalized, LoanApproved ... (consumed)
  Accounting/ # JournalPosted, PeriodClosed (consumed)
```
Allowed references: `Domain` -> SharedKernel; `Application` -> own Domain, SharedKernel, Contracts; `Infrastructure` -> own Application/Domain; the Bangladesh pack implements Payroll's Application-level ports. Architecture tests forbid references from `ERP.Payroll.*` to HR, Accounting, Inventory, Procurement and Common projects.

### 3.3 Request Flow
```
React screen -> HTTPS JSON (+JWT) -> Payroll controller (thin, ERP.Api)
  -> Cortex.Mediator command/query
  -> pipeline: Logging -> Authorization -> Validation -> Transaction -> Performance
  -> Handler -> Aggregate / calculation service -> Repository -> SQL Server
  -> Domain events -> Outbox -> Integration events (Accounting, HR, notifications)
Long operations (calculate run, generate payslips/files) -> Hangfire background job with progress
```

### 3.4 Calculation Engine Design

**Pipeline per employee (deterministic, pure function of a frozen input snapshot):**
1. **Load snapshot:** employee data (version), salary assignment (effective-dated, may split a month), attendance summary (version), pending inputs (OT hours, arrears, bonuses, one-time items), loan installments due, rule versions (tax table, PF rule, formulas).
2. **Proration:** payable-day factor for joiners, leavers, unpaid leave, salary changes mid-month (segments).
3. **Earnings** in dependency order (topological sort of component graph; cycle detection at design time).
4. **Overtime pay** from HR hours and the company rule.
5. **Taxable income** and exemptions (country calculator).
6. **Statutory deductions:** provident fund, income tax (projection + true-up).
7. **Other deductions:** loans, advances, recoveries, penalties.
8. **Net pay**, rounding, validations (negative net, caps).
9. **Result lines** written with `ExplainTrace` (inputs, formula text, intermediate values).

**Component calculation types:** Fixed amount; Percent of another component; **Formula** (safe expression DSL, no code execution); Slab/table lookup; Attendance-based (per day/per hour); Variable input (entered monthly); System (overtime, loan, tax, PF).

**Formula DSL (sandboxed):** variables `BASIC`, `GROSS`, `PAYABLE_DAYS`, `CALENDAR_DAYS`, `WORKING_DAYS`, `ABSENT_DAYS`, `OT_HOURS_NORMAL`, `OT_HOURS_HOLIDAY`, `GRADE_RANK`, any component code; functions `MIN`, `MAX`, `ROUND`, `FLOOR`, `CEIL`, `IF`, `ABS`; operators `+ - * / > < = AND OR`. Parsed to an AST, validated, versioned; no loops, no I/O, execution time-bounded. A **test sandbox** evaluates a formula for a chosen employee before saving.

**Reproducibility:** each run stores the **input snapshot** and **rule version ids**; each employee result stores a hash. Re-calculation of a draft run always re-reads latest inputs; re-**verification** of an approved run replays the snapshot and must match.

**Number handling:** `decimal(18,4)` in storage; per-component rounding rule (none, nearest 1 taka, nearest 0.01); round-half-away-from-zero; company-level net rounding with the residual assigned to a rounding component; total of employee nets = run net payable exactly.

### 3.5 Integration Contracts

**Published by Payroll (`ERP.Contracts.Payroll`)**
```csharp
public sealed record PayrollApproved(Guid TenantId, Guid CompanyId, Guid PayrollRunId, string RunNo, string RunType,
    DateOnly PeriodStart, DateOnly PeriodEnd, DateOnly PostingDate, string CurrencyCode, int Version,
    IReadOnlyList<PayrollPostingLine> Lines);
// PayrollPostingLine: AccountingKey (e.g. SALARY_EXPENSE, NET_SALARY_PAYABLE, INCOME_TAX_PAYABLE, PF_PAYABLE,
//   EMPLOYER_PF_EXPENSE, LOAN_RECEIVABLE, OTHER_DEDUCTIONS_PAYABLE), Side (Debit/Credit), Amount,
//   DepartmentId?, BranchId?, ComponentCode, ComponentKind (Earning/Deduction/EmployerContribution)

public sealed record PayrollPaid(Guid TenantId, Guid PayrollRunId, Guid PaymentBatchId, DateOnly PaymentDate,
    string PaymentMethod, string BankKey, decimal Amount, string CurrencyCode); // Accounting: Dr Net Salary Payable, Cr Bank/Cash

public sealed record PayrollRunCancelled(Guid TenantId, Guid PayrollRunId, string Reason); // Accounting reverses the posting

public sealed record PayrollPeriodLocked(Guid TenantId, Guid CompanyId, int Year, int Month); // HR blocks back-dated changes

public sealed record LoanInstallmentDeducted(Guid TenantId, Guid LoanId, Guid EmployeeId, int InstallmentNo, decimal Amount, DateOnly DeductedOn);
public sealed record FinalSettlementCompleted(Guid TenantId, Guid EmployeeId, Guid SettlementId, decimal NetAmount, DateOnly PaidOn);
public sealed record PayslipPublished(Guid TenantId, Guid EmployeeId, int Year, int Month, Guid PayslipId); // notifications / ESS
```

**Consumed by Payroll**

| Event (publisher) | Payroll reaction |
|---|---|
| `EmployeeJoined`, `EmployeeChanged` (HR) | Upsert `EmployeeRef`; flag "salary not assigned" for new joiners |
| `EmployeeSeparated` (HR) | Mark leaver, create settlement case, stop regular pay after the last working day |
| `AttendanceFinalized` (HR, versioned) | Store `AttendanceSummary` snapshot; unlock run for that month; if a newer version arrives for a calculated run, mark the run **stale** and require recalculation |
| `LoanApproved`, `LoanScheduleChanged` (HR) | Create/refresh deduction schedule |
| `LeaveEncashmentApproved` (HR) | Add taxable earning in the next run |
| `DepartmentChanged`, `BranchChanged` (Common) | Upsert references |
| `JournalPosted` (Accounting) | Show "Posted to accounts: voucher no." on the run |
| `PeriodClosed` (Accounting) | Warn or block approval when the accounting period is closed |

> **Contract changes required in `prd-hr.md` (v1.1):** (1) extend `EmployeeSeparated` with `EncashableLeave` (leave type and days) and notice shortfall days; (2) add `LeaveEncashmentApproved(EmployeeId, LeaveTypeId, Days, Year)`. These are noted in section 16.

### 3.6 Architecture Decision Records (Payroll)
- PAY-ADR-001: Salary data exists only in Payroll; HR/Payroll authorization separated.
- PAY-ADR-002: Payroll run = frozen snapshot + deterministic pure calculation; hash per employee.
- PAY-ADR-003: Sandboxed formula DSL (AST) instead of embedding C# or SQL.
- PAY-ADR-004: Country rules behind `IStatutoryCalculator`; Bangladesh pack as plug-in.
- PAY-ADR-005: Accounting integration via logical accounting keys and aggregated posting lines.
- PAY-ADR-006: Approved runs are immutable; corrections by supplementary runs/adjustments, controlled cancellation by reversal.
- PAY-ADR-007: Tax by annual projection with monthly true-up.
- PAY-ADR-008: Background jobs with progress, idempotency and resumability.
- PAY-ADR-009: Amount confidentiality via RLS, restricted roles and access logging; bank/wallet numbers encrypted.

---

## 4. Domain Model

### 4.1 Aggregates

| Aggregate root | Contains | Key invariants |
|---|---|---|
| `SalaryComponent` | Code, name, kind (Earning, Deduction, EmployerContribution), calc type, formula/percent/fixed, taxable flag, tax-exempt limit rule, PF-applicable flag, proration flag, rounding, accounting key, display order, visibility on payslip | Code unique; formula valid and acyclic against other components; cannot delete if used; changes are versioned |
| `SalaryStructure` (template) | Ordered component lines with default values/formulas, grade/type applicability | Must include a basic component; dependency graph acyclic |
| `EmployeeSalaryAssignment` | Employee, structure, per-component amounts/overrides, effective dates, reason, approval | Effective-dated, no overlap; first assignment effective on or before the join date used for payroll; changes need approval |
| `PayrollCalendar` / `PayrollPeriod` | Month, start/end, pay date, status | Periods contiguous; one regular period per month per company |
| `PayrollRun` | Type, period, status, input snapshot reference, totals, approvals, payment batches | Status machine below; one **regular** run per period/company (supplementary runs allowed); approved run immutable |
| `PayrollRunEmployee` (child, batch-written) | Gross, deductions, net, status flags, hash, lines | Net = gross - deductions; flags for holds/exceptions |
| `PayrollRunLine` | Component result with `ExplainTrace` | Immutable after approval |
| `PayrollInput` | Variable inputs: bonus, arrears, one-time earning/deduction, recoveries | Belongs to a period; locked with the run snapshot |
| `Payslip` | Rendered data and PDF reference, publish state | Created only from an approved run; immutable |
| `PaymentBatch` | Method (bank/MFS/cash), bank, items, file reference, status | Sum of items = batch amount; item status tracked; cannot exceed run net payable |
| `TaxProfile` | Employee tax category, TIN, declarations, previous employer income, rebate inputs | Effective per assessment year |
| `TaxRuleSet` / `StatutoryRule` | Slabs, thresholds, rebate, minimum tax, exemption limits per assessment year | Effective-dated, non-overlapping |
| `ProvidentFundAccount` | Employee, balance ledger (`PfLedgerEntry[]`), vesting | Ledger append-only; balance derived |
| `GratuityPolicy` / `GratuityAccrual` | Eligibility and formula; monthly provision | Accrual idempotent per employee-month |
| `FinalSettlement` | Components, status, approval | Created from `EmployeeSeparated`; one per separation |
| `SalaryHold` | Employee, period, reason, release | Held amounts remain payable until released |
| `LoanDeduction` | Schedule received from HR, deduction status | Installments deducted once; reflects early settlement events |

### 4.2 State Machines

**Payroll run:** `Draft -> InputsLocked -> Calculating -> Calculated -> UnderReview -> (ApprovalPending) -> Approved -> Locked -> PartiallyPaid -> Paid`; side exits: `Stale` (new attendance version or input change after calculation; return to `Draft`), `Cancelled` (permitted only before payment, with reversal event).
Rules: recalculation is allowed in `Draft`, `Calculated`, `UnderReview`; `Approved` freezes results; `Locked` follows accounting posting confirmation or manual lock; `Paid` when all batches paid.

**Salary assignment/revision:** `Draft -> PendingApproval -> Approved -> Effective`. If effective date is in a past period already approved, the system proposes **arrears** for the next run.

**Payment batch:** `Generated -> Sent (file exported) -> Confirmed | PartiallyConfirmed | Failed`; failed items can be re-batched.

**Final settlement:** `Draft -> Calculated -> Approved -> Paid`.

### 4.3 Calculation Rules (defaults, configurable)

**Proration basis (company setting):** calendar days (default), working days, or fixed 30 days.
`Prorated amount = Monthly amount x (Payable days / Basis days)` for components flagged "prorate". Payable days come from HR (`PayableDays`) and exclude unpaid leave and absent days; joiners/leavers pay from join date or to last working day.

**Mid-month salary change:** compute segments by effective date; each segment prorated separately; trace shows both.

**Overtime pay:** `Hourly rate = (Basic + other eligible components) / (Standard monthly hours)` (rule from company policy; default divisor configurable), `OT pay = Hours x Hourly rate x Multiplier` (multiplier per bucket, for example normal and holiday). Hours come from the HR event; Payroll does not recalculate hours.

**Loan recovery:** deduct installments due in the period, capped so net pay does not go below a configurable minimum (default 0); the shortfall rolls to the next month with a warning.

**Net pay rules:** negative net is blocked with an explicit exception that requires a decision (carry forward deduction, adjust, or override with permission).

**Hold:** held employees are excluded from payment batches but included in the run totals and posting (liability remains).

**Provident fund (template):** employee % and employer % of PF base (usually basic), start after confirmation/eligibility, optional vesting schedule for employer share, PF ledger monthly entries, interest allocation yearly (manual or rate-based), withdrawal at separation per vesting.

**Gratuity (template):** eligible after N completed years; amount = `(Basic x days per year / 30) x completed years` (rule configurable); monthly provision accrued; paid in final settlement.

**Festival bonus (template):** percent of basic (or fixed), eligibility by service months, pro-rata for service shorter than threshold, run as a **bonus run** on a chosen date; taxed according to the country calculator.

**Income tax (Bangladesh pack, template):** annual projection method:
1. Projected annual taxable income = year-to-date taxable income + current month + (remaining months x current monthly taxable amount) + known bonuses.
2. Apply exemptions and limits (data), taxpayer category threshold, slabs, rebate from verified investments, minimum tax rule.
3. Monthly tax = `(Projected annual tax - tax already deducted YTD) / remaining months` (never negative; true-up in final month or when income changes).
4. Employees without TIN or declarations handled per configured rule.
> All thresholds, slabs, exemption limits, rebate rules and minimum tax values are **effective-dated data per assessment year**, loaded from NBR publications and verified before release.

---

## 5. Functional Requirements

Priority: **M** Must, **S** Should, **C** Could. "Master" shows the traced master requirement.

### 5.1 Setup

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-001 | Company payroll settings: pay frequency (monthly), proration basis, rounding rules, pay date rule, minimum net pay, currency, default accounting keys | M | PAY-03 |
| PAY-FR-002 | **Salary components** CRUD with kind, calc type, taxable/PF flags, proration, visibility, ordering, accounting key; dependency validation | M | PAY-01 |
| PAY-FR-003 | **Formula editor** with autocomplete, syntax and cycle validation, and a **test sandbox** for a chosen employee; formula versioning with effective dates | M | PAY-01 |
| PAY-FR-004 | **Salary structures** (templates) assigned by grade, employee type or individually; clone structure; compare two structures | M | PAY-02 |
| PAY-FR-005 | Default Bangladesh component set (basic, house rent, medical, conveyance, mobile allowance, festival bonus, PF, income tax, loan, advance) from the country pack, editable | M | BD-07 |
| PAY-FR-006 | Payroll periods (calendar month) generated per year; pay date; cut-off rules | M | PAY-03 |
| PAY-FR-007 | Tax rule sets per assessment year, taxpayer categories, exemption limits, rebate rules (data screens with import and effective dates) | M | BD-07 |
| PAY-FR-008 | PF, gratuity, festival bonus and overtime policy setup | M | BD-09, BD-10 |
| PAY-FR-009 | Bank accounts of the company for payment (bank key), payment methods and file formats (bank advice/BEFTN, MFS list, cash) | S | BD-12 |

### 5.2 Employee Pay Data

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-020 | **Assign salary** to an employee (structure plus amounts), effective-dated, with approval; shows monthly gross and annual CTC | M | PAY-02 |
| PAY-FR-021 | **Salary revision** (increment/promotion/correction) single and **bulk wizard** (percent or amount, by selection/grade, preview before apply) with approval | M | PAY-02 |
| PAY-FR-022 | Retroactive revisions produce **arrears** automatically for affected closed periods | S | PAY-07 |
| PAY-FR-023 | Employee **tax profile** (category, TIN, previous employer income, declared investments with proof upload, approval by Payroll Officer) | S | PAY-16 |
| PAY-FR-024 | New-joiner and missing-data checklist: employees without salary, bank/wallet, tax profile | M | PAY-05 |
| PAY-FR-025 | Salary history timeline per employee (who changed what, when, why); salary view requires permission and is **access-logged** | M | PAY-02 |
| PAY-FR-026 | **Salary hold/release** with reason and approval | S | PAY-04 |
| PAY-FR-027 | Variable inputs per period: bonus, incentive, arrears, one-time earning/deduction, recovery, with import from Excel and approval | M | PAY-07 |

### 5.3 Payroll Run

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-030 | **Create run** (regular, supplementary, bonus) for a period/company/branch scope; prevents duplicate regular runs | M | PAY-04 |
| PAY-FR-031 | **Pre-run checks** (blocking and warning): HR attendance finalized, joiners without salary, missing bank/wallet, missing TIN, loan schedules, accounting period open, unapproved inputs/revisions, previous run state | M | PAY-05 |
| PAY-FR-032 | **Lock inputs** and create an immutable snapshot; later HR changes mark the run stale | M | PAY-04 |
| PAY-FR-033 | **Calculate** as a background job with progress, partial failure reporting per employee, and retry for failed employees only | M | PAY-15 |
| PAY-FR-034 | **Employee calculation trace:** step-by-step breakdown (inputs, formulas, proration, tax projection) for any employee line | M | PAY-04 |
| PAY-FR-035 | **Variance analysis** vs previous month and vs previous run: by employee and total; reasons classified automatically (new joiner, leaver, revision, bonus, unpaid leave, OT change, tax change); threshold highlighting | M | PAY-04 |
| PAY-FR-036 | **Exceptions list:** negative net, zero pay, over threshold change, missing data; each with resolution actions; blocks approval until resolved or waived with reason | M | PAY-04 |
| PAY-FR-037 | **Recalculate** all or selected employees; manual adjustments are made through inputs (never by editing results) | M | PAY-04 |
| PAY-FR-038 | **Approval workflow** (via Common workflow): Payroll Officer prepares, Payroll/Finance Manager reviews, Owner approves above a threshold; approver cannot be the preparer | M | PAY-04 |
| PAY-FR-039 | **Approve** freezes the run; publishes `PayrollApproved`; run shows accounting posting status via `JournalPosted` | M | PAY-12 |
| PAY-FR-040 | **Lock** run (after posting confirmation or manually); publishes `PayrollPeriodLocked` | M | PAY-14 |
| PAY-FR-041 | **Cancel approved run** (unpaid only; permission; reason) publishing `PayrollRunCancelled`; then re-create run | S | PAY-14 |
| PAY-FR-042 | **Supplementary run** for late joiners, corrections and arrears without touching the regular run | M | PAY-07 |
| PAY-FR-043 | **Festival bonus run** with eligibility preview, per-employee amounts and tax effect | M | PAY-17 |
| PAY-FR-044 | Stale-run detection and clear banner explaining which change made it stale and what to do | M | PAY-04 |

### 5.4 Statutory and Benefits (Bangladesh Pack)

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-050 | **Income tax** calculation by annual projection with true-up (4.3); per-employee tax sheet; handles mid-year joiners, previous employer income, bonuses | M | PAY-16 |
| PAY-FR-051 | Tax-exempt component limits and category thresholds applied from data | M | PAY-16 |
| PAY-FR-052 | **Provident fund:** employee/employer contributions, PF ledger, statements, interest allocation (manual or rate), withdrawal and vesting at exit | S | PAY-18 |
| PAY-FR-053 | **Gratuity:** eligibility, monthly provision, balance report, payment in settlement | S | PAY-18 |
| PAY-FR-054 | **Overtime pay** from HR hours with company rule, buckets and multipliers | M | PAY-17 |
| PAY-FR-055 | **WPPF** support: enter the company's distributable amount and allocation rules, generate per-employee allocations (optional) | C | PAY-18 |
| PAY-FR-056 | Tax deduction statement (monthly) and annual **salary tax certificate** per employee; deposit challan references recorded for reporting | M | PAY-16 |
| PAY-FR-057 | Country-pack interface so another country's rules can replace Bangladesh rules without changing the core | M | INT-1 |

### 5.5 Loans, Advances and Recoveries

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-060 | Deduct loan installments from HR schedules automatically; show on payslip with remaining balance | S | PAY-08 |
| PAY-FR-061 | Publish `LoanInstallmentDeducted` after approval so HR balances stay correct | S | PAY-08 |
| PAY-FR-062 | Salary advance recovery and ad-hoc recoveries (asset damage, excess pay) as inputs with approval | S | PAY-07 |
| PAY-FR-063 | Handle early settlement or rescheduling events from HR in the next run, with a clear audit note | S | PAY-08 |

### 5.6 Payslips

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-070 | Generate **payslip PDF** per employee server-side with tenant branding: earnings, deductions, employer contributions (optional), net pay, YTD, tax summary, loan balance, attendance summary | M | PAY-09 |
| PAY-FR-071 | **Publish** payslips to ESS after approval (or after payment, configurable); notify employees; hold individual payslips | M | PAY-09 |
| PAY-FR-072 | Employee view: current and past payslips, YTD, tax statement downloads; **"Why did my pay change?"** comparison vs last month with plain-language reasons | M | PAY-09 |
| PAY-FR-073 | Email delivery with secure link (no salary data in email body) | S | PAY-09 |
| PAY-FR-074 | Payslip layout designer (choose sections and labels, logo) | C | CMN-15 |
| PAY-FR-075 | Bulk download of payslips (ZIP) for HR/Payroll with permission and access log | S | PAY-09 |

### 5.7 Payments

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-080 | **Payment batches** by bank/MFS/cash from an approved run; employees grouped by payment method and bank | M | PAY-10 |
| PAY-FR-081 | Generate **bank advice** (letter PDF) and **BEFTN/bank-specific files**; **MFS** payment list files; formats configurable per bank | S | BD-12 |
| PAY-FR-082 | Record payment confirmation per batch/item (reference, date); handle failed items and re-pay | M | PAY-10 |
| PAY-FR-083 | Publish `PayrollPaid` per confirmed batch so Accounting clears Net Salary Payable | M | PAY-12 |
| PAY-FR-084 | Held and failed payments tracked in an "Unpaid salaries" list with ageing | S | PAY-10 |

### 5.8 Final Settlement

| ID | Requirement | Pri | Master |
|---|---|---|---|
| PAY-FR-090 | Auto-create a settlement case from `EmployeeSeparated` | M | PAY-11 |
| PAY-FR-091 | Calculate: salary to last working day, leave encashment, bonus pro-rata, gratuity, PF settlement, notice pay/shortfall, loan and advance recovery, other deductions, tax on settlement | S | PAY-11 |
| PAY-FR-092 | Settlement statement (PDF), approval, payment, and `FinalSettlementCompleted` event | S | PAY-11 |
| PAY-FR-093 | Exclude separated employees from future regular runs after settlement | M | PAY-11 |

### 5.9 Reports and Analytics

All reports: filter by company, branch, department, period, run; export PDF/Excel; saved views; respect confidentiality permissions; drill-down to employee trace.

| ID | Report | Pri |
|---|---|---|
| PAY-FR-100 | Payroll register (by run) | M |
| PAY-FR-101 | Department/branch/cost-center payroll cost, employer cost | M |
| PAY-FR-102 | Bank advice and payment status | M |
| PAY-FR-103 | Income tax deducted (monthly), annual tax statement | M |
| PAY-FR-104 | PF ledger/statement, gratuity provision, bonus register | S |
| PAY-FR-105 | Variance report, joiners/leavers pay impact | M |
| PAY-FR-106 | Year-to-date earnings per employee, headcount cost trend | S |
| PAY-FR-107 | Loan recovery and outstanding | S |
| PAY-FR-108 | Salary revision history and pending revisions | S |
| PAY-FR-109 | Audit reports: who viewed/exported salary data, approvals, cancellations | M |
| PAY-FR-110 | Payroll dashboard (cost trend, next run status, exceptions, unpaid items) | M |

### 5.10 Key Acceptance Criteria (Given / When / Then)

**AC-1 Proration**
- Given monthly basic 30,000 on a calendar-day basis (30-day month), and 3 unpaid days reported by HR, when the run calculates, then prorated basic is 27,000 and the trace shows `30,000 x 27/30`.

**AC-2 Mid-month revision**
- Given basic 30,000 until 15 Nov and 33,000 from 16 Nov (30-day month), when November is calculated, then basic = `30,000 x 15/30 + 33,000 x 15/30 = 31,500` and the trace lists both segments.

**AC-3 Stale detection**
- Given a calculated run for November, when HR publishes `AttendanceFinalized` version 2, then the run becomes **Stale**, approval is blocked, and the banner explains "Attendance changed after calculation. Recalculate."

**AC-4 Reproducibility**
- Given an approved run, when "Verify run" replays its snapshot, then every employee hash matches and the verification result is stored.

**AC-5 Maker-checker**
- Given user U prepared the run, when U tries to approve it, then the action is blocked ("Another approver is required"); a different authorized user can approve.

**AC-6 Accounting posting**
- Given an approved run, then exactly one `PayrollApproved` event is published; Accounting posts a balanced journal; the run shows "Posted: JV-xxxx"; total debits equal total credits equal total of employee-level values.

**AC-7 Negative net pay**
- Given deductions exceed earnings for employee E, when calculating, then E appears in Exceptions with options (carry forward loan deduction, reduce deduction, waive with reason) and the run cannot be approved until resolved.

**AC-8 Confidentiality**
- Given a Line Manager or HR Officer without payroll permission, when requesting any salary or payslip endpoint (own team or all), then the API returns 403; a Payroll Officer's view of an employee's salary is recorded in the access log.

**AC-9 Tenant isolation**
- Given tenants A and B, when a user of A requests a payroll run or payslip id from B, then the API returns 404 and logs the attempt.

---

## 6. Permissions

| Permission | Payroll Officer | Payroll/Finance Manager | Owner/Approver | HR Manager | Employee | Auditor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| `Payroll.Setup.Manage` | - | Yes | - | - | - | - |
| `Payroll.Salary.View` | Yes (logged) | Yes (logged) | Yes (logged) | - | Self | Yes (logged) |
| `Payroll.Salary.Assign/Revise` (prepare) | Yes | Yes | - | - | - | - |
| `Payroll.Salary.Approve` | - | Yes | Yes | - | - | - |
| `Payroll.Input.Manage` | Yes | Yes | - | - | - | - |
| `Payroll.Run.Prepare/Calculate` | Yes | Yes | - | - | - | - |
| `Payroll.Run.Approve` | - | Yes | Yes | - | - | - |
| `Payroll.Run.Cancel` | - | Yes | - | - | - | - |
| `Payroll.Payment.Manage` | Yes | Yes | - | - | - | - |
| `Payroll.Payslip.ViewAll/Download` | Yes (logged) | Yes (logged) | - | - | Self | Yes (logged) |
| `Payroll.Settlement.Approve` | - | Yes | Yes | - | - | - |
| `Payroll.Report.View` | Yes | Yes | Yes (summary) | Counts only | Self | Yes |
| `Payroll.Audit.View` | - | Yes | - | - | - | Yes |

Rules: preparer != approver; salary revisions need a second person; HR roles never see salary unless also granted payroll permissions; every salary view, export and bulk payslip download is logged.

---

## 7. Bangladesh Pack: Default Policy Template

> **Starting values only.** Verify every item against the current Income Tax law and NBR publications, Labour Act/Rules and company policy; store as **editable, effective-dated data** (master BD-07, BD-09, BD-10).

| Area | Template default (to verify) | Configurable |
|---|---|---|
| Pay frequency | Monthly | Yes |
| Typical earnings | Basic, house rent, medical, conveyance, other allowances | Yes |
| Salary income tax | Employer withholds monthly using annual projection; slabs, taxpayer-category thresholds, exemption limits, investment rebate and minimum tax per assessment year | All data |
| Festival bonus | Commonly one or two per year as a percent of basic; eligibility by service months; taxable | Yes |
| Overtime pay | Commonly twice the ordinary hourly rate (basic plus eligible components); divisor and multiplier set by policy | Yes |
| Provident fund | Employee and employer percent of basic; eligibility after confirmation; optional vesting | Yes |
| Gratuity | By years of service per policy/law; monthly provision | Yes |
| WPPF | Allocation from company profit pool (amount entered in v1) | Yes |
| Payment methods | Bank transfer (BEFTN-style advice), mobile financial services (bKash, Nagad, Rocket), cash | Yes |
| Rounding | Nearest taka for net pay (setting) | Yes |

---

## 8. UI/UX Design

> Stack: React + TypeScript + MUI (shared design system with Accounting and HR). English v1, text externalized for Bangla later. **Two experiences:** Payroll Console (desktop-first, careful and dense) and Employee Payslip experience (mobile-first, simple and reassuring).

### 8.1 Design Principles
1. **No surprises.** Show what will happen before it happens: dry-run calculations, previews, variance, clear consequence statements on approval.
2. **Explain every number.** Any amount is one click from its calculation trace, in plain language plus the formula.
3. **Confidentiality by default.** Salary amounts can be hidden with a **Privacy mode** toggle (shoulder-surfing protection); screens avoid showing salary where not needed; access is logged and the user is told so.
4. **Guided, not rigid.** A stepper walks through the run; blockers are listed with fix links; experts can jump anywhere.
5. **Trustworthy status.** Always show run state, version, snapshot time, who prepared and approved.
6. **Calm under pressure.** Month-end is stressful: quiet visuals, one primary action per step, no decorative noise.
7. **Accessible and inclusive.** WCAG 2.2 AA; variance indicators use arrows and signs, not color alone.

### 8.2 Information Architecture

```
Payroll (Console)
  Dashboard
  Runs                Payroll Runs, Supplementary, Festival Bonus, Final Settlements
  Employees' Pay      Salary Assignments, Revisions (single/bulk), Holds, Tax Profiles
  Inputs              Bonuses, Arrears, One-time Items, Recoveries, Overtime view, Loans
  Payments            Batches, Bank/MFS Files, Unpaid Salaries
  Payslips            Published, Hold, Delivery log
  Statutory           Income Tax, Provident Fund, Gratuity, WPPF
  Reports
  Setup               Settings, Components, Structures, Formulas, Tax Rules, Policies, Bank & Formats, Workflows

My Pay (Self-Service, mobile-first)
  Payslips, Tax & Statements, Pay Changes, Salary Certificate request (optional)
```

### 8.3 Visual System (extends shared MUI theme)

| Token | Rule |
|---|---|
| Typeface | Inter (fallback Roboto); **tabular numerals** for all amounts; right-aligned numbers |
| Primary | Deep blue `#1F4FD8`; secondary teal `#0F766E` for confirmations |
| Semantic | Success `#15803D`, Warning `#B45309`, Error `#B91C1C`, Info `#0369A1`; each paired with icon and text |
| Variance | Up arrow with `+`, down arrow with `-`, neutral dash; magnitude chips (for example `+12.4%`); color is supplementary |
| Amounts | Currency symbol shown once per column header; negatives in parentheses; zero as `-` |
| Status chips | Draft, Inputs locked, Calculating, Calculated, Under review, Approved, Locked, Paid, Stale, Cancelled (icon + text) |
| Privacy mode | Amounts replaced by `••••`; toggle persists per session; keyboard shortcut |
| Density | Compact grids for the console; comfortable on mobile |
| Motion | 150 to 200 ms; progress indicators for jobs; reduced motion respected |

### 8.4 Key Screens and Wireframes

#### 8.4.1 Payroll Dashboard (desktop)
```
+--------------------------------------------------------------------------------+
| Payroll > Dashboard                    [Company v]    [Privacy mode: off]        |
+--------------------------------------------------------------------------------+
| November 2026 run: UNDER REVIEW     Employees 248   Net payable 18,420,500      |
| Step 4 of 6: Review variance          [Continue run ->]                          |
+--------------------------------------------------------------------------------+
| Needs your attention                    | Payroll cost, last 12 months (line)    |
| ! 5 exceptions (2 negative net)         |                                        |
| ! 3 joiners without salary assigned     | Cost by department (bar)               |
| ! 2 salary revisions awaiting approval  |                                        |
| ! 4 failed bank payments (Oct)          | Upcoming: festival bonus run (Dec 3)   |
+--------------------------------------------------------------------------------+
```

#### 8.4.2 Payroll Run Wizard (core experience)
```
+--------------------------------------------------------------------------------+
| Run: November 2026 (Regular)   Status: Calculated   Snapshot: 15 Nov 10:42      |
| [1 Period] > [2 Inputs & checks] > [3 Calculate] > [4 Review] > [5 Approve] > [6 Pay & Post]|
+--------------------------------------------------------------------------------+
| STEP 2: Inputs & checks                                          Ready: 7 of 9  |
| [OK] Attendance finalized (v1, 12 Nov)                                          |
| [OK] All 248 employees have salary assignments                                  |
| [!!] 3 employees missing bank/wallet           [Fix in Employees' Pay ->]       |
| [!!] 2 employees without TIN (tax at higher rate rule)   [Review ->]            |
| [OK] Loan schedules loaded (18 installments)                                    |
| [OK] Accounting period November is open                                         |
+--------------------------------------------------------------------------------+
| [Back]                          Blocking: 0   Warnings: 2        [Lock inputs & calculate] |
+--------------------------------------------------------------------------------+
```
Behavior: blocking items stop progression; warnings can be acknowledged; "Lock inputs" states exactly what is frozen; recalculation is always available before approval.

#### 8.4.3 Review & Variance (Step 4)
```
+--------------------------------------------------------------------------------+
| Totals          This run      Last month     Change                              |
| Gross           21,350,000    20,980,000     (up) +1.8%                          |
| Deductions       2,929,500     2,880,100     (up) +1.7%                          |
| Net payable     18,420,500    18,099,900     (up) +1.8%                          |
+--------------------------------------------------------------------------------+
| Why it changed:  New joiners +420,000 | Revisions +310,000 | Overtime -95,000 |  |
|                  Leavers -260,000 | Other -4,400                                 |
+--------------------------------------------------------------------------------+
| Employee            Gross    Net     Change vs last   Reason            Flags   |
| Rahim Uddin         55,000   47,300  (up) +12.4%      Revision          [!]     |
| Nusrat Jahan        82,000   68,950  (down) -0.0%     -                         |
| Abdul Karim         31,000   (1,200) --              Loan > earnings   [!!]     |
+--------------------------------------------------------------------------------+
| Filter: [Show flagged only]  [Threshold > 10%]   Click a row for calculation trace|
+--------------------------------------------------------------------------------+
```
Behavior: reasons are automatically classified; flagged rows sort first; exceptions must be resolved or waived with a reason before approval.

#### 8.4.4 Employee Calculation Trace (side drawer)
```
+------------------------------------------------------------+
| Nusrat Jahan  EMP-0042  Nov 2026       Net 68,950           |
|------------------------------------------------------------|
| EARNINGS                                                    |
| Basic             40,000   30,000/30 days x 30 payable days |
| House rent        20,000   50% of Basic                     |
| Medical            4,000   Fixed                            |
| Overtime           3,100   9.5 h x 326.3 x 1.0   [Formula] |
| Gross             67,100                                    |
| DEDUCTIONS                                                  |
| Provident fund     4,000   10% of Basic                     |
| Income tax         3,150   Projection: annual taxable ...   |
| Loan installment   2,500   3 of 12  (balance 22,500)        |
| Total deductions   9,650                                    |
| NET PAY           57,450                                    |
|------------------------------------------------------------|
| [Show tax projection]  [Show inputs & versions]  [Copy]    |
+------------------------------------------------------------+
```
Behavior: each line expands to formula and variable values; tax shows annual projection steps; inputs show HR attendance version and salary assignment version; the drawer works for any historical run.

#### 8.4.5 Salary Structure and Formula Editor
```
+--------------------------------------------------------------------------------+
| Structure: Staff Grade 3-5                         [Clone] [Compare] [Save]       |
+-----------------+------------+-------------------+---------+----------------------+
| Component       | Type       | Value / Formula   | Taxable | PF base | Prorate     |
| Basic           | Fixed      | 40,000            | Yes     | Yes     | Yes         |
| House rent      | % of Basic | 50                | Yes*    | No      | Yes         |
| Medical         | Fixed      | 4,000             | Yes*    | No      | Yes         |
| Overtime        | System     | per HR hours      | Yes     | No      | No          |
+-----------------+------------+-------------------+---------+----------------------+
| Formula: IF(GRADE_RANK > 3, BASIC * 0.10, BASIC * 0.05)         [Valid]           |
| Test for employee [Nusrat Jahan v]  Result: 4,000   Trace: BASIC=40,000 -> ...    |
+--------------------------------------------------------------------------------+
```
Behavior: autocomplete for variables; inline error markers; cycle detection message ("House rent depends on Gross, which depends on House rent"); test sandbox never saves anything.

#### 8.4.6 Bulk Salary Revision Wizard
Steps: choose employees (filters/selection) -> choose rule (percent, amount, set to grade scale) -> **preview table** (old, new, change %, new annual cost, arrears if backdated) -> set effective date and reason -> submit for approval. Totals bar shows total monthly cost impact. Invalid rows flagged inline; nothing applies until approved.

#### 8.4.7 Payment Batches
List of batches by bank/MFS/cash with count, amount, status; detail shows items with account tails (masked), file generation (download), confirmation entry (upload bank response or mark paid), failed items with reasons and "Re-pay" action. Totals always visible and reconcile to the run's net payable.

#### 8.4.8 Employee Payslip (mobile-first)
```
+----------------------------------+
| <  Payslip  November 2026        |
|                                  |
|        Net pay                   |
|        BDT 57,450       [eye]    |
|   Paid 30 Nov 2026 (bank ****7712)|
|                                  |
| (!) Your pay changed by +1,200   |
|     [Why did my pay change?]     |
|                                  |
| Earnings                    v    |
|   Basic ................ 40,000  |
|   House rent ........... 20,000  |
|   Overtime ..............  3,100 |
| Deductions                  v    |
|   Provident fund ........ 4,000  |
|   Income tax ............ 3,150  |
|   Loan installment ...... 2,500  |
|                                  |
| Year to date                v    |
| [  Download PDF  ]  [ Statements ]|
+----------------------------------+
```
Behavior: amounts masked until tapped (privacy default configurable); sections collapse; "Why did my pay change?" opens a plain-language list ("Overtime +3,100 because you worked 9.5 extra hours"; "Loan installment 3 of 12 started"); PDF download and share are logged; large touch targets.

#### 8.4.9 Owner Approval (mobile)
Summary card: total net payable, change vs last month with reasons, number of employees, exceptions resolved, preparer and reviewer names, top 5 variances; buttons **Approve** and **Request changes** (comment required). Approval confirmation states: "This will publish the payroll to Accounting and lock all results."

### 8.5 Interaction Patterns

| Pattern | Standard |
|---|---|
| Lists | MUI X Data Grid: server paging/sort/filter, column chooser, saved views, pinned totals row, Excel export (logged) |
| Long jobs | Progress bar with counts, per-employee failures listed, safe to leave the page, notification on completion |
| Forms | Labels above fields; inline validation; clear units (BDT, %, days); error summaries |
| Confirmations | Only for irreversible or high-impact actions (approve, lock, cancel, publish payslips); dialog states consequences and what it publishes |
| Undo | Prefer reversible design: recalculation before approval, cancellation by controlled flow after; never silent edits to results |
| Diff views | Side-by-side changes for salary revisions, structure comparisons, run vs previous run |
| Privacy | Privacy mode toggle, masked bank/wallet, reveal with logging, watermark on exported payroll files |
| Notifications | Run state changes, approvals needed, payslips published, payment failures |
| Search | Global Ctrl+K: employee, run, payslip, batch |

### 8.6 States and Microcopy
- **Specific and calm:** "3 employees don't have a bank or wallet number yet. They will be skipped in bank files." plus the fix link.
- **Negative net:** "Abdul Karim's deductions (32,200) are higher than earnings (31,000). Choose how to handle it."
- **Stale run:** "Attendance for November was updated after this calculation. Recalculate to include the change."
- **Errors never blame**; success toasts state what changed ("Payroll approved and sent to Accounting").
- **Loading:** skeletons; calculations show "Calculated 112 of 248".

### 8.7 Responsive Strategy
- **Mobile:** employee payslips, statements, pay-change explanations; owner/manager approval summary; notifications. Payroll Console is not optimized for phones except read-only status.
- **Tablet:** console usable; trace drawer full-width.
- **Desktop:** full console.

### 8.8 Accessibility (WCAG 2.2 AA)
Contrast >= 4.5:1; keyboard operation across grids, wizard and drawers; screen-reader labels for amounts and variance ("Net pay 57,450 taka, up 1,200 from last month"); no color-only meaning; announcements for job progress and errors; target size and zoom to 200%; reduced motion; privacy mode does not break assistive text (announces "amount hidden").

### 8.9 Keyboard Shortcuts (Console)

| Keys | Action |
|---|---|
| Ctrl+K | Global search |
| Ctrl+Shift+P | Toggle privacy mode |
| J / K | Next / previous employee in review list |
| Enter | Open calculation trace |
| Alt+R | Recalculate selected |
| / | Focus filter |
| ? | Shortcut help |

### 8.10 UX Research and Success Measures
- Test with 4 payroll officers, 3 owners/managers, 6 employees (mid-range Android).
- Tasks: run a month end-to-end; resolve a negative-net exception; build a structure with a formula; bulk revise salaries; find why a payslip changed; approve on a phone.
- Targets: task success >= 90%; run review of 250 employees in <= 20 minutes; owner approval <= 60 s; employees find "why my pay changed" in <= 20 s; SUS >= 75; zero confidentiality findings.
- Instrumentation (privacy-safe): step drop-off in the wizard, exception resolution time, trace-drawer usage, payslip open rate, support-question topics.
- Deliverables: Figma flows and prototype for 8.4, content style guide for payroll terms, accessibility audit.

---

## 9. Data Design (SQL Server, schema `pay`)

### 9.1 Conventions
Standard columns (`Id`, `TenantId`, `CompanyId`, audit fields, `RowVersion`), Row-Level Security on `TenantId`, unique indexes start with `TenantId`, `decimal(18,4)` money, `decimal(18,6)` rates, `date` for business dates, `datetime2` UTC for timestamps. **Bank and wallet numbers** are application-encrypted. Salary and result amounts are protected by RLS, restricted database roles, TDE and access logging (they must remain queryable for reports).

### 9.2 Main Tables

| Table | Purpose |
|---|---|
| `Setting` | Company payroll settings |
| `SalaryComponent`, `ComponentFormula` (versioned) | Components and formulas |
| `SalaryStructure`, `SalaryStructureLine` | Templates |
| `EmployeeRef`, `DepartmentRef`, `BranchRef` | Local read models from events |
| `EmployeeSalaryAssignment`, `AssignmentLine` | Effective-dated pay per employee |
| `SalaryRevision`, `RevisionLine` | Revision requests and bulk batches |
| `TaxProfile`, `TaxDeclaration` | Employee tax data |
| `TaxRuleSet`, `TaxSlab`, `TaxExemptionRule`, `TaxRebateRule` | Statutory tax data by assessment year |
| `PolicyPf`, `PolicyGratuity`, `PolicyBonus`, `PolicyOvertime` | Benefit policies (effective-dated) |
| `PayrollPeriod` | Months and status |
| `AttendanceSummary` | Snapshots from HR per employee/month/version |
| `PayrollInput` | Variable inputs (bonus, arrears, one-time, recoveries) |
| `PayrollRun` | Run header: type, period, status, snapshot id, totals, verification hash, approvals |
| `PayrollRunSnapshot` | Frozen inputs and rule version ids (JSON/relational) |
| `PayrollRunEmployee` | Per-employee result and flags |
| `PayrollRunLine` | Component results with `ExplainTrace` (JSON) |
| `PayrollException` | Exceptions and resolution |
| `SalaryHold` | Holds and releases |
| `Payslip` | Rendered data, PDF reference, publish state |
| `PaymentBatch`, `PaymentItem` | Payments |
| `PfAccount`, `PfLedgerEntry` | PF |
| `GratuityAccrual` | Monthly provisions |
| `FinalSettlement`, `FinalSettlementLine` | Exit pay |
| `LoanDeduction` | Schedule and status from HR |
| `SensitiveAccessLog` | Views/exports of salary data |
| `InboxMessage`, `OutboxMessage` | Reliable messaging |

### 9.3 Integrity Rules
```sql
-- One regular run per company and period (supplementary/bonus allowed with sequence)
CREATE UNIQUE INDEX UX_Run_Regular
  ON pay.PayrollRun (TenantId, CompanyId, PeriodId, RunType, Sequence);

-- One result row per employee per run
CREATE UNIQUE INDEX UX_RunEmployee ON pay.PayrollRunEmployee (TenantId, RunId, EmployeeId);

-- Net = Gross - Deductions
ALTER TABLE pay.PayrollRunEmployee ADD CONSTRAINT CK_Net
  CHECK (NetPay = GrossPay - TotalDeductions);

-- Salary assignment starts are unique per employee (non-overlap checked in domain and by trigger)
CREATE UNIQUE INDEX UX_Assignment_Start ON pay.EmployeeSalaryAssignment (TenantId, EmployeeId, EffectiveFrom);

-- Approved runs are immutable: application role has no UPDATE/DELETE on result tables after approval
-- (enforced by domain state check plus a trigger rejecting changes where Run.Status >= 'Approved')

-- PF ledger is append-only; balance = SUM(Amount)
-- Idempotent processing of HR/Accounting events
CREATE UNIQUE INDEX UX_Inbox_Message ON pay.InboxMessage (TenantId, MessageId);
```

### 9.4 Indexing and Performance
- `PayrollRunLine (TenantId, RunId, EmployeeId)` clustered for per-employee trace; `(TenantId, RunId, ComponentCode)` for totals.
- `EmployeeSalaryAssignment (TenantId, EmployeeId, EffectiveFrom DESC)` for "as of date" lookups.
- Calculation reads the **snapshot** only, with set-based loading; results inserted in batches of employees.
- Reports on aggregates use `PayrollRunLine` with covering indexes; year-to-date tax uses a materialized `EmployeeYtd` table updated at approval.

### 9.5 Concurrency and Consistency
- Run state transitions take an exclusive row lock on `PayrollRun`; the calculation job refuses to start if the state is not calculable.
- Job idempotency key `(RunId, EmployeeId, SnapshotVersion)` allows safe retry of failed employees.
- Optimistic concurrency on assignments, revisions, inputs; duplicate approve clicks idempotent.
- Outbox publishes `PayrollApproved` in the same transaction as the approval.

---

## 10. API Design (ERP.Api, controllers + Cortex.Mediator)

Base: `/api/v1/payroll`. JSON, JWT, tenant from the token. Errors as `ProblemDetails` with stable codes (for example `PAY.RUN_STALE`, `PAY.NEGATIVE_NET`, `PAY.SELF_APPROVAL`). `Idempotency-Key` supported on approve, pay and publish commands.

| Area | Endpoints (examples) |
|---|---|
| Setup | `GET/PUT /settings`, `GET/POST /components`, `POST /formulas/validate`, `POST /formulas/test`, `GET/POST /structures`, `GET/POST /tax-rule-sets`, `GET/PUT /policies/pf|gratuity|bonus|overtime` |
| Employee pay | `GET /employees/{id}/salary` (logged), `POST /employees/{id}/salary-assignments`, `POST /revisions`, `POST /revisions/bulk/preview`, `POST /revisions/bulk/submit`, `POST /holds`, `GET/PUT /employees/{id}/tax-profile` |
| Inputs | `GET/POST /inputs`, `POST /inputs/import` |
| Runs | `POST /runs`, `GET /runs/{id}`, `GET /runs/{id}/checks`, `POST /runs/{id}/lock-inputs`, `POST /runs/{id}/calculate`, `GET /runs/{id}/progress`, `GET /runs/{id}/variance`, `GET /runs/{id}/exceptions`, `POST /runs/{id}/exceptions/{exId}/resolve`, `GET /runs/{id}/employees/{empId}/trace`, `POST /runs/{id}/submit`, `/approve`, `/reject`, `/lock`, `/cancel`, `POST /runs/{id}/verify` |
| Payments | `POST /runs/{id}/payment-batches`, `GET /payment-batches/{id}`, `POST /payment-batches/{id}/export`, `POST /payment-batches/{id}/confirm`, `POST /payment-items/{id}/repay` |
| Payslips | `POST /runs/{id}/payslips/publish`, `GET /payslips?employeeId=`, `GET /payslips/{id}/pdf`, `POST /payslips/{id}/hold` |
| Statutory | `GET /tax/monthly-statement`, `GET /tax/annual-statement?employeeId=`, `GET /pf/ledger`, `GET /gratuity/provision` |
| Settlement | `GET /settlements`, `POST /settlements/{id}/calculate`, `/approve`, `/pay` |
| Reports/Dashboard | `GET /reports/register`, `/cost-summary`, `/variance`, `/ytd`; `GET /dashboard/summary` |
| Self-service | `GET /me/payslips`, `GET /me/payslips/{id}`, `GET /me/pay-changes?period=`, `GET /me/tax-statement` |

Cortex.Mediator naming: `CreatePayrollRunCommand`, `CalculatePayrollRunCommand`, `ApprovePayrollRunCommand`, `ReviseSalaryCommand`, `PublishPayslipsCommand`, `GetPayrollVarianceQuery`, `GetCalculationTraceQuery`.

---

## 11. Non-Functional Requirements (Payroll-Specific)

| Area | Requirement |
|---|---|
| Performance | Calculate 500 employees < 60 s, 1,000 < 2 min (background, parallel per employee); trace drawer < 500 ms; payslip PDF generation 500 in < 2 min; variance view < 2 s |
| Accuracy | Decimal arithmetic only; golden-case suite must pass in CI; hash verification of approved runs |
| Confidentiality | RLS, restricted roles, TDE; access log for salary views/exports; encrypted bank/wallet numbers; masked exports with watermark; no salary data in logs or emails |
| Integrity | Approved runs immutable; DB constraints; reconciliation of employee nets to batch totals and to Accounting journal |
| Auditability | Full history of salary changes, inputs, run actions, approvals, cancellations, file generations |
| Reliability | Idempotent jobs and events; resumable calculation; safe retries; outbox for publishing |
| Scalability | Sized for ~500 employees per tenant (master A8); jobs scheduled with per-tenant fairness |
| Retention | Payroll records retained per legal requirement (configurable, default 7+ years); archival, not deletion |
| Compatibility | Latest two versions of Chrome, Edge, Firefox, Safari; mobile Chrome/Safari for payslips; IIS and Docker/Azure hosting without code change |
| Maintainability | Architecture tests; Domain coverage >= 90% (calculation code), Application >= 75% |

### 11.1 Observability (Serilog)
- Log commands with name, tenant, user, duration, outcome; **never** log salary amounts, tax figures, bank numbers, or payslip content.
- Enrich with `TenantId`, `CompanyId`, `CorrelationId`, `RunId`, `EmployeeId` (ids only).
- Metrics: run duration, employees per second, failures per run, exception counts, outbox lag, payment failures, payslip generation time. Alerts: calculation job failure, outbox lag > 1 minute, run stale for > 24 h, unpaid salaries older than N days.

---

## 12. Testing Strategy

| Level | Focus |
|---|---|
| Domain unit tests | Proration, mid-month segments, formula evaluation, dependency ordering and cycle detection, rounding, overtime, loan caps, PF, gratuity, bonus eligibility |
| Tax tests | Table-driven tests per assessment year: slab boundaries, thresholds by category, exemptions, rebate, minimum tax, projection and true-up, mid-year joiner, bonus month |
| Golden cases | A library of hand-verified payroll cases (typical staff, leaver, joiner, revision, unpaid leave, loan, high earner) with expected net to the taka |
| Property-based tests | `Net = Gross - Deductions`; sum of employee nets = batch total = run net; scaling inputs behaves monotonically where expected; formula engine never loops or throws unhandled errors on random input |
| Determinism tests | Re-run from snapshot equals stored hashes; parallel and serial calculation produce identical results |
| Application tests | Permissions, maker-checker, stale detection, idempotent events, workflow routing |
| Integration tests | EF mappings, RLS, immutability triggers, concurrency (double approve, parallel calculate), outbox/inbox, job resume after failure |
| Contract tests | Events with HR (`EmployeeJoined`, `AttendanceFinalized`, `LoanApproved`, `EmployeeSeparated`) and Accounting (`PayrollApproved`, `PayrollPaid`, `JournalPosted`) |
| Scenario tests | Hire-to-Pay end-to-end: employee joins, attendance finalized, payroll approved, journal posted, payments confirmed, payslip published; offboarding to final settlement |
| Security tests | Role matrix for salary endpoints, access logging, export masking, tenant isolation |
| UI tests | Playwright: run wizard, exception resolution, bulk revision, payslip on mobile viewport, owner approval; axe checks |
| Performance | 1,000 employees calculation; 500 payslip PDFs; concurrent tenants |

---

## 13. Delivery Plan for the Payroll Module

| Release | Scope | Exit criteria |
|---|---|---|
| **P1 Foundation** | Module skeleton, settings, components, structures, assignments, local read models from HR events, calculation engine (fixed, percent, formula, proration), run create/calculate, trace drawer, basic payslip | AC-1, AC-2, AC-9 pass; golden cases for basic pay green |
| **P2 Bangladesh statutory** | Tax rule sets and projection, PF, overtime, festival bonus run, exceptions and variance review, stale detection | Tax test suite green; AC-3, AC-7 pass |
| **P3 Approval, accounting and payments** | Workflow approval, maker-checker, `PayrollApproved`/`PayrollPaid`, run lock/cancel, payment batches and bank/MFS files, ESS payslips with "why my pay changed", owner mobile approval | AC-4, AC-5, AC-6, AC-8 pass; Hire-to-Pay scenario green |
| **P4 Settlement and revisions** | Final settlement, gratuity, loans and recoveries, salary revisions (single/bulk) with arrears, tax statements, holds, supplementary runs | Settlement and arrears scenarios green |
| **P5 Hardening** | WPPF (optional), reports and dashboard, performance tuning, privacy mode polish, accessibility audit, country-pack documentation | Usability targets met; 1,000-employee performance target met |

---

## 14. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Wrong tax or statutory calculation | Critical | Data-driven rules, table-driven tests, golden cases, expert review each assessment year, clear disclaimer in templates |
| Rule or law changes mid-year | High | Effective-dated rule sets; update without redeploy; versioned in snapshots |
| Salary data leakage | Critical | Strict permissions, RLS, access logging, privacy mode, masked exports, no salary in logs/emails |
| Attendance or HR data changes after calculation | High | Versioned events, stale detection, snapshot locking, clear banner and recalculation |
| Formula errors by users | Medium | Sandbox testing, validation, cycle detection, versioned formulas, preview for sample employees |
| Posting mismatch with Accounting | High | Aggregated posting lines with logical keys, contract tests, reconciliation totals, exception queue on the Accounting side |
| Bank file format differences | Medium | Formatter plug-ins per bank, sample file validation, configurable columns |
| Payment failures and unpaid salaries | Medium | Item-level status, repay flow, unpaid list with ageing |
| Overly complex for SMEs | Medium | Default component/structure set, wizard, progressive disclosure, hide advanced settings until needed |

---

## 15. Definition of Done (Module)

- All **Must** requirements in section 5 implemented; AC-1 to AC-9 pass.
- Architecture tests prove Payroll references no other module's projects.
- Golden-case and tax suites pass; determinism and hash verification pass.
- HR and Accounting contract and scenario tests (Hire-to-Pay) green.
- Every screen in 8.4 implemented, accessible (axe clean, keyboard walkthrough) and verified on mobile (payslip, approval) and desktop (console).
- Confidentiality verified: role matrix, access logs, masked exports, no salary data in logs or emails.
- Bangladesh tax, PF, gratuity, bonus and overtime templates reviewed and signed off by a qualified tax/payroll expert before release.
- Usability targets in 8.10 met with real users.
- Documentation: API (Swagger), event contracts, calculation and formula guide, tax rule maintenance guide, user guides for payroll officers, managers and employees.

---

## 16. Open Questions

1. Payroll cycle: is a **calendar month** enough (aligned with HR attendance), or do customers need cut-off cycles such as 26th to 25th?
2. Who maintains the yearly tax rule data (platform team via country pack updates, or tenant payroll officers)? Assumed: platform team ships updates, tenants can override.
3. Which banks and MFS providers need file formats first (to prebuild formatters)?
4. Is a **PF trust** (recognized fund with trustees and interest rules) required, or a simple internal PF ledger?
5. Should payslips publish at approval or only after payment confirmation (company-level default)?
6. Are cash salary payments common enough to need a cash disbursement register with signatures?
7. Contract updates in `prd-hr.md` (v1.1): extend `EmployeeSeparated` with encashable leave and notice shortfall; add `LeaveEncashmentApproved`. Confirm and apply.
8. Expense reimbursements through payroll: include in a later release?

---

## 17. Glossary

| Term | Meaning |
|---|---|
| CTC | Cost to company: annual total employment cost |
| Gross / Net | Total earnings / pay after deductions |
| Proration | Adjusting pay for part of a period |
| Snapshot | Frozen inputs and rule versions used by a run |
| Trace | Step-by-step explanation of how an amount was calculated |
| Variance | Difference between this run and a comparison run |
| Supplementary run | Extra run for corrections or late items without changing the regular run |
| Arrears | Back pay due to retroactive changes |
| Projection method | Tax computed on estimated annual income, adjusted monthly |
| Assessment year | Tax year to which tax rules apply |
| PF / WPPF | Provident Fund / Workers' Profit Participation Fund |
| BEFTN | Bangladesh Electronic Funds Transfer Network |
| MFS | Mobile financial services (bKash, Nagad, Rocket) |
| Accounting key | Logical account identifier that Accounting maps to a real account |
| Outbox / Inbox | Tables ensuring reliable, idempotent messaging between modules |
