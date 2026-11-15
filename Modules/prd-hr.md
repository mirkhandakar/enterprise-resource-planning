# Human Resources (HR) Module PRD

**Product:** Integrated Multi-Tenant ERP Platform (SaaS)
**Module:** Human Resources (bounded context `HR`, schema `hr`)
**Parent document:** `prd.md` (Master PRD v0.5). Refines master requirements HR-01 to HR-16, BD-08, BD-09, BD-11, CMN-04, CMN-09. Sibling documents: `prd-accounting.md`, `prd-payroll.md` (planned).

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
HR is the **system of record for people and their working time**. It owns the employee master, organization structure, employment history, attendance, leave, loans/advances, offboarding and basic recruitment/performance/training. It supplies clean, trusted **employee and time data to Payroll**, and offers self-service so employees and managers handle their own requests without paper or phone calls.

### 1.2 Product Goals
1. **One truth about every person:** a single, effective-dated employee record with full history and documents.
2. **Time that adds up:** attendance and leave rules produce correct payable days with no spreadsheet work.
3. **Self-service first:** employees and managers complete 90% of routine requests themselves, mostly from a phone.
4. **Payroll-ready:** a locked monthly attendance summary flows to Payroll as an event, not as a file.
5. **Bangladesh-ready and configurable:** Labour Act-based policy templates, Eid and government holidays, weekly off, Friday/other weekend, overtime rules; everything editable per company.
6. **Privacy by design:** sensitive personal data is encrypted, masked and access-logged.

### 1.3 In Scope (v1)
Employee master and lifecycle, organization structure (designation, grade, reporting line), contracts, shifts and rosters, attendance (manual, import, web check-in), regularization, overtime, leave policies/balances/requests, holidays, loans and advances, offboarding, documents and letters, employee/manager self-service, notifications, HR reports and dashboard.

### 1.4 Out of Scope (v1)
Payroll calculation (Payroll module), biometric device drivers (import file/API only), applicant tracking beyond a light pipeline, 360-degree reviews, learning management system, benefits administration/insurance claims, native mobile apps (responsive PWA-ready web only), Bangla UI (later release).

### 1.5 Success Metrics

| Metric | Target |
|---|---|
| Leave request submission on phone | <= 45 seconds median |
| Manager approval | <= 20 seconds median |
| New employee record (full onboarding data entry) | <= 10 minutes |
| Monthly attendance finalization (500 employees) | <= 2 hours of HR effort, <= 30 seconds compute |
| Leave balance accuracy | 100% reproducible from the leave ledger |
| Self-service adoption | >= 80% of leave requests submitted by employees themselves |
| Employee data completeness (mandatory fields) | >= 98% |
| Usability | Task success >= 90%, SUS >= 75 (employees and HR officers) |

---

## 2. Users and Jobs-to-be-Done

| Persona | Context | Top jobs-to-be-done |
|---|---|---|
| **HR Officer / HR Manager** (primary) | Desktop, daily | "Onboard a new hire quickly", "Keep records correct", "Finalize attendance for payroll", "Answer any employee question from the record" |
| **Employee** | Mostly phone | "Check my leave balance", "Apply for leave", "See my attendance", "Download my payslip/letters", "Update my phone number" |
| **Line Manager** | Phone and desktop | "Approve leave/regularization fast", "See who is out this week", "Know my team's attendance" |
| **Business Owner (SME)** | Occasional | "Headcount and cost trend", "Who is absent today", "Contracts about to expire" |
| **HR Admin / System Admin** | Setup | "Configure leave, shifts, holidays, workflows once and correctly" |
| **Payroll Officer** | Month-end | "Get a final, locked attendance and leave summary" |
| **Auditor** (read-only) | Verification | "See who changed a record and when" |

---

## 3. Module Architecture (Modular Monolith)

### 3.1 Bounded Context and Data Ownership
- HR **owns** schema `hr`. No other module reads or writes `hr` tables, and HR does not read other modules' tables.
- Cross-module references are **soft references** (IDs and codes), never foreign keys.
- **Resolved ownership overlaps with the master PRD:**

| Data | Owner | HR's access |
|---|---|---|
| Company, Branch, **Department** (organization unit) | Common | Local read model `hr.DepartmentRef` fed by `DepartmentChanged` events |
| User accounts, roles | Common | `hr.UserRef`; HR emits `EmployeeOnboarded`, Common answers `UserLinked` |
| Employee master, designation, grade, reporting line, attendance, leave, loans | **HR** | Owner |
| Salary structure, employee salary, payroll runs, payslips | Payroll | HR has **no** salary data; salary visibility links to Payroll through UI permission only |
| Cost centers (accounting) | Accounting | HR sends `DepartmentId`; Payroll/Accounting map it |
| Approval workflow engine | Common | HR defines workflow *types* and uses contract `IReportingLineQuery` so Common can resolve "reporting manager" |

### 3.2 Projects (one per layer)

```
Modules/HR/
  ERP.HR.Domain/          # Aggregates, value objects, domain services (leave calc, attendance calc), domain events
  ERP.HR.Application/     # Commands, queries, handlers, validators, DTOs, ports, event handlers
  ERP.HR.Infrastructure/  # HRDbContext, EF configs, migrations, repositories, device-import parsers, PDF letter templates
BuildingBlocks/ERP.Contracts/
  HR/        # Events HR publishes + IReportingLineQuery
  Common/    # DepartmentChanged, UserLinked ... (consumed)
  Payroll/   # LoanInstallmentDeducted, FinalSettlementCompleted (consumed)
```
Allowed references: `Domain` -> SharedKernel; `Application` -> own Domain, SharedKernel, Contracts; `Infrastructure` -> own Application/Domain. Architecture tests forbid references from `ERP.HR.*` to Accounting, Payroll, Inventory, Procurement and Common projects.

### 3.3 Request Flow
```
React screen -> HTTPS JSON (+JWT) -> HR controller (thin, ERP.Api)
  -> Cortex.Mediator command/query
  -> pipeline: Logging -> Authorization -> Validation -> Transaction -> Performance
  -> Handler -> Aggregate (domain rules) -> Repository -> SQL Server
  -> Domain events -> Outbox -> Integration events to other modules
```

### 3.4 Integration Strategy
- Inbox/Outbox tables for reliable, idempotent messaging (same pattern as Accounting).
- HR keeps minimal local read models (`DepartmentRef`, `UserRef`, `BranchRef`).
- Payroll never queries HR: it receives **employee lifecycle events** and an **attendance summary event** and keeps its own employee read model.
- Sensitive payloads (bank account, national ID) are sent **only** to modules that need them (Payroll: bank account; never NID).

### 3.5 Integration Contracts

**Published by HR (`ERP.Contracts.HR`)**
```csharp
public sealed record EmployeeJoined(Guid TenantId, Guid CompanyId, Guid EmployeeId, string EmployeeNo, string FullName,
    DateOnly JoinDate, Guid DepartmentId, Guid BranchId, Guid DesignationId, Guid GradeId, string EmploymentType,
    EmployeeBankInfo? Bank, int Version);

public sealed record EmployeeChanged(Guid TenantId, Guid EmployeeId, DateOnly EffectiveFrom, string ChangeType, // Transfer, Promotion, BankUpdate, Status...
    Guid? DepartmentId, Guid? BranchId, Guid? DesignationId, Guid? GradeId, EmployeeBankInfo? Bank, int Version);

public sealed record EmployeeSeparated(Guid TenantId, Guid EmployeeId, DateOnly LastWorkingDay, string Reason, bool EligibleForSettlement);

public sealed record AttendanceFinalized(Guid TenantId, Guid CompanyId, int Year, int Month, int Version,
    IReadOnlyList<EmployeeAttendanceSummary> Employees);
// EmployeeAttendanceSummary: EmployeeId, PayableDays, PresentDays, AbsentDays, PaidLeaveDays, UnpaidLeaveDays,
//   WeeklyOffDays, HolidayDays, LateCount, OvertimeHoursNormal, OvertimeHoursHoliday

public sealed record LoanApproved(Guid TenantId, Guid LoanId, Guid EmployeeId, decimal Amount,
    IReadOnlyList<LoanInstallment> Schedule); // DueMonth, Amount
public sealed record LoanScheduleChanged(Guid TenantId, Guid LoanId, IReadOnlyList<LoanInstallment> Schedule);

// Synchronous query contract implemented by HR, called by Common's workflow engine
public interface IReportingLineQuery
{
    Task<IReadOnlyList<Guid>> GetApproverUserIdsAsync(Guid tenantId, Guid requesterUserId, ApproverRole role, CancellationToken ct);
}
```

**Consumed by HR**

| Event (publisher) | HR reaction |
|---|---|
| `DepartmentChanged`, `BranchChanged` (Common) | Upsert `DepartmentRef`, `BranchRef` |
| `UserLinked` (Common) | Store `UserId` on employee, enable self-service |
| `LoanInstallmentDeducted` (Payroll) | Reduce loan balance, mark installment paid |
| `FinalSettlementCompleted` (Payroll) | Complete the payroll clearance task in the offboarding case |
| `PayrollPeriodLocked` (Payroll) | Reject back-dated attendance/leave changes for that month unless an authorized correction flow is used |

### 3.6 Architecture Decision Records (HR)
- HR-ADR-001: Effective-dated employment records (no overwriting history).
- HR-ADR-002: Leave balances as an append-only **ledger**; balance = sum of entries.
- HR-ADR-003: Raw punches stored separately from computed attendance days (recompute is always possible).
- HR-ADR-004: Attendance month finalized by event; Payroll consumes a versioned snapshot.
- HR-ADR-005: No salary data in HR; clear separation of HR and Payroll authorization.
- HR-ADR-006: Department master in Common, read model in HR.
- HR-ADR-007: Field-level encryption and access logging for sensitive personal data.

---

## 4. Domain Model

### 4.1 Aggregates

| Aggregate root | Contains | Key invariants |
|---|---|---|
| `Employee` | Personal profile, contacts, emergency contacts, bank accounts, documents, `EmploymentRecord[]` (effective-dated), status | Employee number unique per company; exactly one active employment record at any date; join date <= any later effective date; national ID unique per tenant (when provided); cannot be hard-deleted |
| `ReportingLine` (inside Employee) | Manager reference, effective dates | No cycles; manager must be active on the effective date |
| `Designation`, `Grade`, `EmployeeType` | Master data | Codes unique; cannot delete when used |
| `Shift` | Start/end, break, grace, flexible window, night-shift flag | End after start (or crosses midnight flagged); grace >= 0 |
| `Roster` | Employee-to-shift assignment per date range | No overlapping assignments per employee |
| `AttendanceDay` | Employee, date, first in, last out, status, late/early minutes, work hours, overtime, source flags | One per employee per date; locked when the period is finalized |
| `AttendancePeriod` | Company, month, status (Open, Finalized), version | Only Open periods accept changes; finalize publishes the summary event |
| `LeaveType`, `LeavePolicy` | Entitlement, accrual, carry-forward, expiry, rules | Rules validated on save (for example max carry-forward <= entitlement unless allowed) |
| `LeaveAccount` (per employee, per leave type, per year) | `LeaveLedgerEntry[]` | Balance = sum(entries); balance below zero only if the type allows negative |
| `LeaveRequest` | Days (per-date lines), status, approvals, attachment | No overlap with other active requests; days computed by the work calendar; cannot exceed balance (unless allowed) |
| `HolidayCalendar` | Holidays, weekly-off pattern, assignment to company/branch | No duplicate holiday dates in a calendar |
| `EmployeeLoan` | Terms, schedule, installments, status | Sum of installments = principal (+ interest if any); balance never negative |
| `OffboardingCase` | Separation details, clearance tasks, exit interview | Last working day >= notice start; all mandatory tasks done before closure |
| `Vacancy` / `Candidate` (light) | Pipeline and stages | Convert to employee only from "Offer accepted" |
| `AppraisalCycle` (light) | Goals, ratings | One review per employee per cycle |
| `TrainingRecord` | Course, dates, certificate, expiry | Expiry >= completion date |

### 4.2 State Machines

**Employee status:** `Candidate/Pre-joining -> Probation -> Confirmed -> (OnNotice) -> Separated`; side states: `Suspended`, `OnLongLeave`. Rehire creates a new employment record on the same person with the same employee number policy (configurable).

**Leave request:** `Draft -> Submitted -> Approved | Rejected | Cancelled`; `Approved -> CancellationRequested -> Cancelled` (after approval, cancellation may need approval). Pending requests **reserve** balance.

**Attendance period:** `Open -> Finalized` (re-open allowed before Payroll locks the same month, by permission, producing a new `Version` of the event).

**Loan:** `Requested -> Approved -> Disbursed -> Repaying -> Closed | WrittenOff`.

**Offboarding:** `Initiated -> Clearance in progress -> Final settlement pending -> Closed`.

### 4.3 Calculation Rules

**Leave day counting**
- Days = working days in the range per the employee's **work calendar** (weekly off + holidays excluded). Optional **sandwich rule**: weekly offs/holidays between two leave days count as leave (configurable per leave type).
- Half-day supported (first half / second half); counted as 0.5.
- Pending requests reserve balance; approval converts reservation to consumption; rejection/cancellation releases it.

**Accrual and carry-forward (via ledger entries)**
- Accrual methods: annual grant (1 Jan or join date), monthly accrual, or per days worked (for example earned leave accruing from worked days).
- Pro-rata for new joiners; probation restriction configurable (for example earned leave only after confirmation).
- Year-end job writes `CarryForward` and `Expire` entries; encashment writes `Encashment` entries consumed by final settlement/annual payout.

**Attendance status per day** (first match):
1. Approved leave (full or half) -> `LV` (or `HD` + P)
2. Public holiday -> `H`; weekly off -> `WO`
3. No punches -> `A` (Absent), unless regularized or on duty-travel
4. Late beyond grace -> `P-L` (Present, late); early out -> `P-E`
5. Otherwise `P`

**Late policy (configurable):** after N late arrivals in a month, deduct 0.5 or 1 day from a chosen leave type or mark as unpaid. **Missing punch** -> status `Incomplete` until regularized.

**Overtime:** hours beyond shift end (or beyond daily/weekly standard hours) after a minimum threshold; rounding rule (for example round down to 15 min); separate buckets for normal-day and holiday/weekly-off work; requires approval when pre-approval is enabled; daily/weekly caps with warnings. The pay multiplier is applied in Payroll, not HR.

**Night shifts:** punches crossing midnight are assigned to the shift start date.

**Time zone:** punches stored in UTC with the device/source; attendance date derived in the **company time zone** (default Asia/Dhaka).

---

## 5. Functional Requirements

Priority: **M** Must, **S** Should, **C** Could. "Master" shows the traced master requirement.

### 5.1 Organization Setup

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-001 | Manage designations (title, level, job description), grades (code, rank), employee types (permanent, contract, probation, intern, part-time, daily worker) | M | HR-02 |
| HR-FR-002 | View departments and branches (from Common) and assign employees; **org chart** view with reporting lines (zoom, search, export) | M | HR-02 |
| HR-FR-003 | Employee number sequence per company with prefix and format (for example `EMP-0001`), editable pattern | M | HR-01 |
| HR-FR-004 | Company HR settings: work week (weekly off days), standard hours, time zone, probation length, notice periods, retirement age, required employee fields | M | CMN-10 |
| HR-FR-005 | Custom fields (text, number, date, list) on the employee profile, with section placement and visibility by role | S | HR-01 |
| HR-FR-006 | Setup wizard that creates sensible defaults (work week, one shift, Bangladesh leave policy template, holiday calendar for the year) | M | BD-08 |

### 5.2 Employee Master and Lifecycle

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-010 | **Add employee** wizard: personal (name, DOB, gender, marital status, blood group, photo), contact, addresses (present/permanent), national ID/passport, TIN, emergency contacts, bank account (and mobile wallet), education, previous employment, documents | M | HR-01 |
| HR-FR-011 | Save as **draft** at any step; resume later; duplicate check on name+DOB+phone/NID before saving | M | HR-01 |
| HR-FR-012 | **Employment record (effective-dated):** join date, type, department, branch, designation, grade, manager, work location, shift/roster, work calendar; every change creates a new dated record, history visible in a timeline | M | HR-03 |
| HR-FR-013 | Lifecycle actions: confirmation (with probation review), **transfer**, **promotion**, status change, suspension, reinstatement, each with reason, effective date, optional approval and generated letter | M | HR-03 |
| HR-FR-014 | **Contracts:** type, start/end, renewal; alerts 60/30/7 days before expiry to HR and the manager | S | HR-04 |
| HR-FR-015 | **Documents:** upload (NID, certificates, contract, photos), type, expiry date, verification status (Pending, Verified), version history; files via `IFileStorage` | M | HR-01 |
| HR-FR-016 | Expiry alerts for documents (work permit, license, certification) | S | HR-10 |
| HR-FR-017 | **Self-service profile update:** employee edits a safe subset (phone, address, emergency contact, photo); sensitive fields (bank, name, NID) go through HR approval | S | HR-07 |
| HR-FR-018 | Employee search and directory: by name, number, department, designation, status, phone; saved filters; export (permission-controlled) | M | HR-13 |
| HR-FR-019 | **Bulk import** employees from Excel/CSV with validation preview and error report; bulk update of selected fields | S | CMN-11 |
| HR-FR-020 | Field-level permissions: sensitive fields (NID, bank, DOB, medical, disciplinary notes) hidden or masked by role; every reveal is logged | M | HR-14 |
| HR-FR-021 | Linking to a system user: invite employee to create an account (email/phone); one user per employee | M | HR-07 |
| HR-FR-022 | Rehire: reactivate a former employee with a new employment record; history preserved | S | HR-03 |
| HR-FR-023 | Employee timeline: chronological feed of all events (joined, promoted, leave highlights, documents, letters) | S | HR-03 |

### 5.3 Shifts, Rosters and Attendance

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-030 | Shift master: fixed and flexible, break, grace for late/early, night shifts; multiple shifts per company | M | HR-05 |
| HR-FR-031 | **Roster**: assign shifts by employee/department and date range; weekly rotation patterns; bulk assignment grid | S | HR-05 |
| HR-FR-032 | **Punch ingestion:** (a) manual entry, (b) CSV/Excel import with column mapping per device, (c) REST endpoint for device/middleware push with API key per tenant, (d) **web check-in/out** with optional IP/geofence rules | M | HR-05 |
| HR-FR-033 | Idempotent ingestion: duplicate punches (same employee, timestamp, device) are ignored; unknown employee codes go to an **unmatched punches** list | M | HR-05 |
| HR-FR-034 | Daily attendance computation job (and on-demand recompute) producing `AttendanceDay` per 4.3 | M | HR-05 |
| HR-FR-035 | **Attendance regularization** request: missing punch, wrong punch, on-duty/field visit, work-from-home; with reason and manager approval | M | HR-05 |
| HR-FR-036 | **Overtime**: calculation per rules, optional pre-approval request, approval, monthly totals | M | HR-05 |
| HR-FR-037 | **Late/early policy** automation (deductions, warnings, notifications) | S | HR-05 |
| HR-FR-038 | Daily attendance board (who is present, late, absent, on leave, not yet in) with filters by department/branch | M | HR-13 |
| HR-FR-039 | **Monthly attendance sheet** (grid: employees x days with codes), inline correction with reason (audited), bulk fill for a range | M | HR-05 |
| HR-FR-040 | **Finalize attendance period:** pre-checks (incomplete days, pending regularizations, pending leave, unmatched punches), then lock and publish `AttendanceFinalized` with a version | M | PAY-05 |
| HR-FR-041 | Re-open a finalized period (permission and reason) if Payroll has not locked it; republish new version | S | PAY-05 |

### 5.4 Leave and Holidays

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-050 | **Leave types:** paid/unpaid, half-day allowed, attachment required after N days, gender-specific, requires confirmation, negative balance allowed, sandwich rule, max consecutive days, encashable | M | HR-06 |
| HR-FR-051 | **Leave policies:** entitlement per type and per employee group (type, grade, location), accrual method, pro-rata, carry-forward limit, expiry; assignment effective-dated | M | HR-06, HR-15 |
| HR-FR-052 | **Leave ledger:** every grant, accrual, consumption, adjustment, carry-forward, expiry, encashment is an immutable entry with reason and actor; balances derived | M | HR-06 |
| HR-FR-053 | **Apply for leave** (employee or HR on behalf): dates, half-day option, reason, attachment, handover note; live preview of days, balance after, and conflicts | M | HR-06 |
| HR-FR-054 | Validation: balance, overlap, minimum notice, maximum consecutive days, blackout periods, probation restrictions, with clear messages | M | HR-06 |
| HR-FR-055 | **Approval workflow** via Common's engine: reporting manager, then HR (configurable by type/days); **delegation** when approver is on leave; auto-escalation after N days; reminders | M | CMN-04 |
| HR-FR-056 | Cancel or modify an approved leave (before/after start) with rules and approval; ledger is corrected through reversal entries | M | HR-06 |
| HR-FR-057 | **Holiday calendars:** government/Eid/company holidays per year; assign by company/branch; weekly-off patterns; copy last year; (lunar dates entered manually each year) | M | BD-11, HR-16 |
| HR-FR-058 | **Team leave calendar** (month/week) with filters, "who is out" widget, clash warning when too many team members are off | S | HR-06 |
| HR-FR-059 | Leave **encashment** and year-end carry-forward/expiry jobs with preview and approval | S | BD-08 |
| HR-FR-060 | Leave balance and history for each employee, downloadable statement | M | HR-06 |
| HR-FR-061 | Approved paid/unpaid leave feeds attendance status and the payroll summary (4.3, 5.3) | M | PAY-05 |

### 5.5 Employee and Manager Self-Service (ESS / MSS)

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-070 | **ESS home:** leave balances, quick "Apply leave", today's attendance/check-in button, pending requests, announcements, holidays ahead | M | HR-07 |
| HR-FR-071 | View own attendance calendar and request regularization | M | HR-07 |
| HR-FR-072 | View and download own letters, certificates and (via Payroll link) payslips | S | HR-07 |
| HR-FR-073 | **Manager inbox:** all pending approvals (leave, regularization, overtime, loan, profile change) with approve/reject/comment, bulk approve | M | HR-07 |
| HR-FR-074 | **Team view:** direct reports, today's attendance, who's on leave, upcoming leaves, contract/probation dates | S | HR-07 |
| HR-FR-075 | Notifications (in-app and email; SMS optional later) for submissions, approvals, reminders, expiring documents | M | CMN-09 |
| HR-FR-076 | Company announcements and policy documents with acknowledgement tracking | C | HR-07 |

### 5.6 Loans and Advances

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-080 | Loan/advance types (salary advance, festival advance, personal loan) with max amount rules, tenure, interest (optional), eligibility (for example after confirmation) | S | HR-11 |
| HR-FR-081 | Request -> approval -> disbursement record; schedule generated automatically; edit schedule before approval | S | HR-11 |
| HR-FR-082 | Publish `LoanApproved` and `LoanScheduleChanged` so Payroll deducts installments; consume `LoanInstallmentDeducted` to update balances | S | PAY-08 |
| HR-FR-083 | Early settlement, reschedule, write-off (with approval); outstanding balance shown on separation | S | HR-12 |
| HR-FR-084 | Loan register and per-employee loan statement | S | HR-13 |

### 5.7 Offboarding and Exit

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-090 | Resignation submission (employee) and termination initiation (HR) with notice period calculation and last working day | M | HR-12 |
| HR-FR-091 | **Clearance checklist** template per company (IT, assets, finance/loans, department handover, library/ID card) with task owners and due dates; progress visible | S | HR-12 |
| HR-FR-092 | Exit interview form and reasons for attrition analytics | C | HR-12 |
| HR-FR-093 | Publish `EmployeeSeparated`; Payroll calculates final settlement; HR receives `FinalSettlementCompleted` | M | PAY-11 |
| HR-FR-094 | Generate experience certificate, release letter, NOC from templates | S | HR-12 |
| HR-FR-095 | Access disabling: on last working day, request Common to deactivate the user account (event) | M | HR-12 |

### 5.8 Documents and Letters

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-100 | Letter templates (appointment, confirmation, promotion, transfer, warning, experience, salary-certificate placeholder) with merge fields and tenant branding | S | HR-03 |
| HR-FR-101 | Generate letters as PDF server-side (QuestPDF), store in the employee's documents, and make available in ESS after release | S | CMN-16 |
| HR-FR-102 | Disciplinary notes and warnings recorded with restricted visibility | C | HR-14 |

### 5.9 Recruitment (Light)

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-110 | Vacancy request with approval, then published vacancy record | C | HR-08 |
| HR-FR-111 | Candidate pipeline (Applied, Screened, Interview, Offer, Accepted, Rejected), CV upload, interview notes and scores | C | HR-08 |
| HR-FR-112 | **Convert accepted candidate to employee** pre-filling the add-employee wizard; onboarding checklist created | C | HR-08 |

### 5.10 Performance and Training (Light)

| ID | Requirement | Pri | Master |
|---|---|---|---|
| HR-FR-120 | Appraisal cycles with goals, self and manager ratings, final rating, history | C | HR-09 |
| HR-FR-121 | Training records, certificates, expiry alerts | C | HR-10 |
| HR-FR-122 | Probation review workflow ahead of confirmation date | S | HR-03 |

### 5.11 Reports and Analytics

All reports: filter by company, branch, department, status, date; export PDF/Excel; saved views; scheduled email (S); respect field-level permissions.

| ID | Report | Pri |
|---|---|---|
| HR-FR-130 | Headcount (by department, branch, grade, type, gender), joiners/leavers trend, **attrition rate** | M |
| HR-FR-131 | Employee master list and profile print | M |
| HR-FR-132 | Daily attendance, monthly attendance summary, late/absent analysis, overtime | M |
| HR-FR-133 | Leave balance, leave taken, leave liability (days outstanding), leave calendar | M |
| HR-FR-134 | Contract, probation, document-expiry and birthday/anniversary lists | M |
| HR-FR-135 | Loan register and outstanding | S |
| HR-FR-136 | Offboarding status and exit reasons | S |
| HR-FR-137 | HR dashboard: headcount, today's absence, pending approvals, expiring items, turnover trend | M |
| HR-FR-138 | Audit reports: record changes, sensitive-data views, export history | M |

### 5.12 Key Acceptance Criteria (Given / When / Then)

**AC-1 Leave overlap**
- Given employee E has approved leave 10-12 Nov, when E applies for 12-14 Nov, then submission is blocked with "You already have leave on 12 Nov" and a link to the existing request.

**AC-2 Day counting**
- Given a 5-day work week (Fri/Sat weekly off) and a holiday on Tue, when E applies Sun-Thu (5 calendar days with 1 holiday), then the preview shows 4 leave days and the balance after is calculated from 4.

**AC-3 Ledger accuracy**
- Given any employee and leave type, when all ledger entries are summed, then the result equals the displayed balance, and each entry shows its source (request id, accrual job, adjustment reason).

**AC-4 Effective-dated change**
- Given E is in Sales until 30 Nov, when HR records a transfer to Marketing effective 1 Dec, then reports for November show Sales, December shows Marketing, and the timeline shows both with the transfer letter.

**AC-5 Attendance finalization**
- Given 3 incomplete days remain in November, when HR clicks Finalize, then the system lists them with fix links and blocks finalization unless the user has permission to override with a reason; after finalization, `AttendanceFinalized` v1 is published once with correct totals.

**AC-6 Punch idempotency**
- Given a device file is imported twice, then the second import adds zero new punches and reports "N duplicates ignored".

**AC-7 Privacy**
- Given a Line Manager, when viewing a direct report's profile, then NID, bank and personal salary-related fields are not visible, and an HR Officer's reveal of a masked NID is recorded in the access log.

**AC-8 Tenant isolation**
- Given tenants A and B, when a user of A requests an employee id from B, then the API returns 404 and logs the attempt.

---

## 6. Permissions

| Permission | Employee | Line Manager | HR Officer | HR Manager | Payroll Officer | Owner | Auditor |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `HR.Employee.ViewSelf` | Yes | Yes | Yes | Yes | Yes | Yes | - |
| `HR.Employee.ViewTeam` | - | Yes | - | - | - | - | - |
| `HR.Employee.ViewAll` | - | - | Yes | Yes | Limited | Yes | Yes |
| `HR.Employee.Create/Edit` | - | - | Yes | Yes | - | - | - |
| `HR.Employee.ViewSensitive` (NID, bank) | - | - | Yes | Yes | Bank only | - | Logged |
| `HR.Lifecycle.Execute` (transfer, promote, separate) | - | - | Prepare | Approve | - | Yes | - |
| `HR.Leave.Request` | Yes | Yes | Yes | Yes | Yes | Yes | - |
| `HR.Leave.Approve` | - | Team | Yes | Yes | - | Yes | - |
| `HR.Leave.Adjust` (ledger adjustments) | - | - | - | Yes | - | - | - |
| `HR.Attendance.Manage` | - | Team view | Yes | Yes | - | - | - |
| `HR.Attendance.Finalize/Reopen` | - | - | Finalize | Both | - | - | - |
| `HR.Loan.Approve` | - | - | - | Yes | - | Yes | - |
| `HR.Setup.Manage` | - | - | - | Yes | - | - | - |
| `HR.Report.View` | Self | Team | Yes | Yes | Limited | Yes | Yes |
| `HR.Audit.View` | - | - | - | Yes | - | - | Yes |

Rules: nobody approves their own request (the next approver in line is used); managers see only their reporting subtree (direct and indirect configurable); HR cannot see salary (Payroll module permission); every sensitive-field reveal and export is logged.

---

## 7. Bangladesh Labour Act Policy Template

> **Starting values only.** They must be verified against the current Bangladesh Labour Act, Labour Rules and the customer's own policy before release, then stored as **editable, effective-dated policy data** (master BD-08). The sector (factory, shop/commercial, other) changes some values.

| Policy | Template default (to verify) | Configurable |
|---|---|---|
| Weekly off | 1 day per week (commonly Friday); second weekly off optional | Yes |
| Standard hours | 8 hours/day, 48 hours/week | Yes |
| Casual leave | 10 days/year | Yes |
| Sick leave | 14 days/year (medical certificate after N days) | Yes |
| Annual/earned leave | Accrues by days worked (about 1 day per 18 days worked in many cases) | Yes, with accrual formula |
| Festival leave | 11 days/year on declared festival days | Yes, via holiday calendar |
| Maternity leave | 16 weeks (8 before and 8 after delivery), eligibility conditions | Yes |
| Probation | Configurable (commonly 3 to 6 months) | Yes |
| Overtime | Hours beyond standard hours; pay rule in Payroll (commonly twice ordinary rate) | Yes |
| Notice period | By employment type and cause | Yes |
| Holidays | Government gazette and Eid holidays; lunar dates entered yearly | Yes |

---

## 8. UI/UX Design

> Stack: React + TypeScript + MUI (shared design system with Accounting). English v1, text externalized for Bangla later. **Two experiences:** HR Admin (desktop-first, dense) and Self-Service (mobile-first, simple).

### 8.1 Design Principles
1. **People first, data second.** Show faces, names and status before fields; make records feel human but professional.
2. **Mobile-first for employees and managers.** Most self-service happens on a phone: one-handed use, large touch targets (>= 44 px), short flows, no horizontal scrolling.
3. **Desktop efficiency for HR.** Dense grids, keyboard shortcuts, bulk actions, split panes.
4. **Privacy visible and respectful.** Mask sensitive data by default; "Reveal" is deliberate and logged; show why something is hidden.
5. **Guide, don't gatekeep.** Wizards with progress, drafts, smart defaults and plain-language validation.
6. **Clarity of status.** Every request shows where it is, who must act next, and what happens after. No black holes.
7. **Accessible and inclusive.** WCAG 2.2 AA; no color-only meaning; names of any length and script handled without truncation problems.

### 8.2 Information Architecture

```
HR (HR Admin experience)
  Dashboard
  People            Directory, Org Chart, Add Employee, Import
  Attendance        Today, Monthly Sheet, Regularizations, Overtime, Punches (import/unmatched), Finalize
  Leave             Requests, Team Calendar, Balances, Holidays, Encashment
  Lifecycle         Probation & Confirmation, Transfers & Promotions, Contracts
  Loans & Advances
  Offboarding       Cases, Clearance, Exit
  Recruitment       Vacancies, Candidates   (light)
  Performance & Training                    (light)
  Documents & Letters
  Reports
  Setup             Org (designations, grades, types), Shifts & Rosters, Leave Types & Policies, Work Calendar, Workflows, Templates, Settings

My Space (Self-Service, mobile-first)
  Home, My Leave, My Attendance, My Profile, My Documents, Requests, Team (if manager), Approvals (if manager)
```
Role-aware navigation: users see only areas they can use. Mobile uses a **bottom tab bar** (Home, Leave, Attendance, Approvals, Me); desktop uses the left rail.

### 8.3 Visual System (extends the shared MUI theme)

| Token | Rule |
|---|---|
| Typeface | Inter (fallback Roboto); tabular numerals for counts, hours, balances |
| Primary | Deep blue `#1F4FD8`; secondary teal accent `#0F766E` for HR-positive actions (approve, present) |
| Neutrals | Slate scale (as Accounting) |
| Attendance/leave codes | Every status = **color + letter + icon**: P Present, A Absent, L Late, LV Leave, H Holiday, WO Weekly off, HD Half day, INC Incomplete. Colors from a color-blind-safe palette; also patterned fills in the monthly grid |
| Avatars | Photo or initials on a deterministic neutral-tint background; 24/32/40/64 px sizes |
| Status chips | Draft, Submitted, Approved, Rejected, Cancelled, Finalized with icon + text |
| Spacing | 4 px grid; mobile screen padding 16 px; desktop 24 px |
| Density | HR grids default compact; self-service comfortable |
| Motion | 150 to 200 ms; reduced-motion respected |
| Dark mode | Tokenized; later phase |

### 8.4 Key Screens and Wireframes

#### 8.4.1 HR Dashboard (desktop)
```
+--------------------------------------------------------------------------------+
| HR > Dashboard                       [Company v] [Branch: All v]  [Nov 2026 v]  |
+--------------------------------------------------------------------------------+
| Headcount 248 | Present today 211 | On leave 17 | Absent 14 (unplanned) | Late 9 |
+--------------------------------------------------------------------------------+
| Needs your attention                    | Headcount by department (bar)         |
| ! 6 leave requests pending (2 > 3 days) |                                       |
| ! 3 regularizations pending             | Joiners vs leavers, 12 months (line)  |
| ! 4 contracts expire in 30 days         |                                       |
| ! Attendance Oct: ready to finalize     | Upcoming: birthdays, anniversaries,   |
| ! 2 probations end this week            | holidays (list)                       |
+--------------------------------------------------------------------------------+
```
"Needs your attention" is the primary element; each row links to a pre-filtered screen.

#### 8.4.2 Employee Directory (desktop and mobile)
```
+--------------------------------------------------------------------------------+
| People  [Search name, no., phone]  [Dept v][Status v][Type v]   [Cards|Table] [+ Add]|
+--------------------------------------------------------------------------------+
| (photo) Nusrat Jahan        Sr. Accountant   Finance   Dhaka   Active   [...]  |
| (photo) Rahim Uddin         Store Keeper     Stores    Dhaka   Probation [...] |
| (AB)    Abdul Karim         Driver           Admin     Ctg     Active   [...] |
+--------------------------------------------------------------------------------+
```
Cards view for browsing (mobile default); table with column chooser and bulk actions for HR. Row click opens the profile; quick actions: call, email, view leave.

#### 8.4.3 Employee Profile
```
+--------------------------------------------------------------------------------+
| (photo) Nusrat Jahan  EMP-0042   Active   Confirmed   [Edit] [Actions v]        |
| Sr. Accountant | Finance | Dhaka | Manager: Mahmud Hasan | Joined 12 Mar 2021   |
+--------------------------------------------------------------------------------+
| Overview | Job & History | Personal | Documents | Leave | Attendance | Timeline  |
+--------------------------------------------------------------------------------+
| Contact: +880 17XX-XXXXXX   nusrat@...      Emergency: Rafiq (Husband) ...      |
| NID: ********4821 [Reveal]  (logged)         Bank: **** 7712 [Reveal]            |
| Leave balances: Casual 6.0 | Sick 11.0 | Earned 14.5       Contract: Permanent    |
| Job history (timeline): 2021 Joined -> 2022 Confirmed -> 2024 Promoted (Grade 4) |
+--------------------------------------------------------------------------------+
```
Behavior: sensitive fields masked with explicit **Reveal** (logged, auto-hide after 30 s); tabs deep-linkable; **Actions** menu (Transfer, Promote, Confirm, Issue letter, Start offboarding) opens a side drawer with effective date and reason; effective-dated history shown as a timeline, never as edits that erase the past.

#### 8.4.4 Add Employee Wizard (desktop; works on tablet)
```
Step: [1 Personal] [2 Contact] [3 Job] [4 Documents] [5 Bank] [6 Review]   Draft saved 10:42
+--------------------------------------------------------------------------------+
| Full name [_________________]   Date of birth [__/__/____]   Gender [v]        |
| Photo [Upload]   National ID [_____________]  (validates format, checks duplicates)|
|                                                                                  |
| ! Possible duplicate: "Nusrat Jahan" born 12 Apr 1993 exists (EMP-0042) [View]   |
+--------------------------------------------------------------------------------+
| [Back]                                           [Save draft]   [Next: Contact]  |
+--------------------------------------------------------------------------------+
```
Behavior: autosave draft every 30 s; required fields clearly marked; the Review step lists missing items with jump links; the final action "Create employee and invite to system" is explicit about the invitation email/SMS.

#### 8.4.5 Apply for Leave (mobile-first)
```
+----------------------------------+
| <  Apply for leave               |
|                                  |
| Leave type                       |
| [ Casual leave            v ]    |
| Balance: 6.0 days                |
|                                  |
| From [ 12 Nov 2026 ]             |
| To   [ 14 Nov 2026 ]             |
| [ ] Half day                     |
|                                  |
| 2.0 days (Fri weekly off skipped)|
| Balance after: 4.0               |
|                                  |
| Reason [______________________]  |
| Attachment [+ Add photo/file]    |
|                                  |
| Approver: Mahmud Hasan (Manager) |
| [        Submit request        ] |
+----------------------------------+
```
Behavior: live **day count and balance preview** before submit; calendar picker shows holidays and weekly off; conflicts and rule messages appear inline in plain words ("Sick leave over 3 days needs a medical certificate"); after submit, a confirmation shows status, next approver and expected response, with a link to cancel while pending.

#### 8.4.6 Manager Approval Inbox (mobile and desktop)
```
+----------------------------------+
| Approvals (5)         [Select]   |
| ---------------------------------|
| (photo) Rahim Uddin              |
| Casual leave  12-14 Nov (2 days) |
| 3 teammates also off on 13 Nov!  |
| [Reject]            [Approve]    |
| ---------------------------------|
| (photo) Nusrat Jahan             |
| Regularization: missing out-punch|
| 09 Nov  Reason: client visit     |
| [Reject]            [Approve]    |
+----------------------------------+
```
Behavior: key context shown without opening (days, balance impact, team clash); reject requires a short reason; bulk approve for selected items; undo for 5 seconds after action; large tap targets.

#### 8.4.7 Monthly Attendance Sheet (HR, desktop)
```
+--------------------------------------------------------------------------------+
| Attendance: Nov 2026  [Dept v]  Status: Open   Issues: 3 incomplete   [Finalize]|
+--------------------------------------------------------------------------------+
| Employee        | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | ... | P  | A | LV | OT(h)   |
| Nusrat Jahan    | P | P | WO| P | P | L | P | P | ... | 21 | 0 | 2  | 6.5     |
| Rahim Uddin     | P | A | WO| P |INC| P | P | LV| ... | 19 | 1 | 1  | 0       |
+--------------------------------------------------------------------------------+
| Legend: P Present  L Late  A Absent  LV Leave  H Holiday  WO Weekly off  INC Incomplete |
+--------------------------------------------------------------------------------+
```
Behavior: sticky header and first column; click a cell to see punches, shift, and edit with a mandatory reason (audited); filter "Show issues only"; keyboard navigation across cells; the Finalize dialog lists blockers with fix links and states what will be sent to Payroll.

#### 8.4.8 Team Leave Calendar
Month/week view with person rows and colored leave bars (type color + pattern + label), holiday bands, and a clash indicator when more than the allowed number of people in a team are off. Click a bar to open the request. Mobile: agenda list grouped by date.

#### 8.4.9 Offboarding Case
Header with employee, last working day and a progress ring; checklist grouped by owner (IT, Finance, HR, Department) with due dates and status; loans and settlement block showing outstanding items; right rail with documents (resignation letter, clearance form, experience letter draft). Closing the case is disabled until mandatory tasks are done, with the reason shown.

### 8.5 Interaction Patterns

| Pattern | Standard |
|---|---|
| Lists | MUI X Data Grid: server-side paging/sort/filter, column chooser, saved views, density toggle, Excel export (permission-controlled) |
| Forms | Labels above fields; one column on mobile, up to two on desktop; inline validation on blur; error summary with anchors; dates use locale format with a calendar and keyboard entry |
| Wizards | Progress stepper, autosave drafts, review step with missing-item links, safe exit ("Draft saved") |
| Requests | Consistent request card: type, dates, status chip, current approver, history timeline, actions |
| Confirmations | Only for consequential actions (finalize attendance, separate employee, reject); consequences stated in the dialog |
| Undo | Short undo window after approvals; otherwise correction through explicit "Cancel/Reverse" flows, never silent edits |
| Notifications | In-app bell plus email; actionable (approve from the notification on desktop); quiet hours; per-user preferences |
| Search | Global Ctrl+K: employee by name/number/phone, request id; recent items first |
| Empty states | Teach the first step ("No employees yet. Add your first employee or import from Excel") |
| Privacy | Masked values, Reveal with logging, watermark on exported sensitive reports |

### 8.6 States and Microcopy
- **Plain, kind, specific:** "You have 4.0 casual leave days left. This request needs 5.0." plus a suggested action ("Request 4 days, or choose Earned leave").
- **Errors never blame:** "We couldn't find that employee number. Check the number or search by name."
- **Loading:** skeletons shaped like content; optimistic UI for approvals with rollback on failure.
- **Offline/poor network (mobile):** clear banner, queued draft for leave requests, retry button; no data loss.
- **Names:** support long names and non-Latin scripts, never truncate without a tooltip.

### 8.7 Responsive Strategy
- **Mobile (< 768 px):** Self-service and approvals fully supported; HR directory and profile read/update basics; heavy grids (monthly sheet) read-only with a horizontal-scroll-safe layout.
- **Tablet (768 to 1279 px):** all screens usable; two-pane layouts collapse to tabs.
- **Desktop (>= 1280 px):** full HR admin experience.
- Installable **PWA** (add to home screen), camera upload for documents and medical certificates, web check-in/out button on the ESS home (optional per tenant).

### 8.8 Accessibility (WCAG 2.2 AA)
Contrast >= 4.5:1; visible focus; full keyboard operation including grids and calendars; screen-reader labels ("Casual leave, 12 to 14 November, approved"); status never conveyed by color alone; touch targets >= 44 px on mobile (min 24 px per WCAG); announcements for validation and status changes; respects zoom to 200% and reduced motion.

### 8.9 Keyboard Shortcuts (HR Admin)

| Keys | Action |
|---|---|
| Ctrl+K | Global search |
| Alt+N | New employee / new request (context) |
| Arrow keys | Move across the attendance grid |
| Enter | Open cell/row details |
| A / R | Approve / Reject focused request (in inbox) |
| / | Focus filter |
| ? | Shortcut help |

### 8.10 UX Research and Success Measures
- Test with 5 HR officers, 5 employees (phone, mid-range Android), 3 line managers.
- Tasks: add an employee; apply for leave; approve with a team clash; correct a missing punch; finalize a month; start offboarding.
- Targets: task success >= 90%; leave application <= 45 s; approval <= 20 s; SUS >= 75; zero privacy-leak findings.
- Instrumentation (privacy-safe): form drop-off by step, validation errors by field, time to approve, notification-to-action time.
- Deliverables: Figma flows and prototype for 8.4, mobile-first component specs, content style guide, accessibility audit.

---

## 9. Data Design (SQL Server, schema `hr`)

### 9.1 Conventions
Standard columns (`Id`, `TenantId`, `CompanyId`, audit fields, `RowVersion`), soft delete, Row-Level Security on `TenantId`, unique indexes starting with `TenantId`, dates as `date` for business dates and `datetime2` (UTC) for timestamps. Sensitive columns (national ID, bank account, passport) use **application-level encryption** (or Always Encrypted) with a deterministic blind index for duplicate checks.

### 9.2 Main Tables

| Table | Purpose |
|---|---|
| `Employee` | Person and current-state summary: EmployeeNo, names, DOB, gender, marital status, photo ref, contact, addresses, NID (encrypted), TIN, UserId, Status |
| `EmploymentRecord` | **Effective-dated** job data: EmployeeId, EffectiveFrom, EffectiveTo, TypeId, DepartmentId, BranchId, DesignationId, GradeId, ManagerEmployeeId, ShiftId, WorkCalendarId, ChangeReason |
| `EmployeeBankAccount`, `EmergencyContact`, `Education`, `PreviousEmployment` | Profile sub-data |
| `EmployeeDocument` | Type, file ref, ExpiryDate, VerificationStatus, Version |
| `Contract` | Type, start, end, renewal |
| `Designation`, `Grade`, `EmployeeType` | Master data |
| `CustomFieldDefinition`, `CustomFieldValue` | Extensible profile |
| `Shift`, `ShiftRule`, `RosterAssignment` | Shifts and rosters |
| `PunchLog` | Raw punches: EmployeeId (nullable), EmployeeCode, PunchTimeUtc, Source, DeviceId, Direction, IsMatched |
| `AttendanceDay` | Computed day: EmployeeId, Date, ShiftId, FirstIn, LastOut, WorkMinutes, LateMinutes, EarlyMinutes, OvertimeMinutes, StatusCode, Flags |
| `AttendanceRegularization` | Request, reason, approval state |
| `AttendancePeriod` | CompanyId, Year, Month, Status, Version, FinalizedBy/At |
| `OvertimeRule`, `OvertimeRequest` | OT configuration and approvals |
| `LeaveType`, `LeavePolicy`, `LeavePolicyRule`, `LeavePolicyAssignment` | Leave configuration (effective-dated) |
| `LeaveLedger` | Append-only entries: EmployeeId, LeaveTypeId, Year, EntryType, Days, ReferenceType/Id, Reason, CreatedBy |
| `LeaveRequest`, `LeaveRequestDay` | Requests with per-date lines (full/half) |
| `WorkCalendar`, `HolidayCalendar`, `Holiday` | Weekly off and holidays |
| `Loan`, `LoanInstallment` | Loans and schedules |
| `OffboardingCase`, `ClearanceTemplate`, `ClearanceTask` | Exit management |
| `Vacancy`, `Candidate`, `Interview`, `AppraisalCycle`, `Appraisal`, `TrainingRecord` | Light modules |
| `LetterTemplate`, `GeneratedLetter` | Letters |
| `SensitiveAccessLog` | Who revealed or exported what and when |
| `DepartmentRef`, `BranchRef`, `UserRef` | Local read models |
| `InboxMessage`, `OutboxMessage` | Reliable messaging |

### 9.3 Integrity Rules
```sql
-- One punch per employee code, device and timestamp (idempotent imports)
CREATE UNIQUE INDEX UX_Punch_Dedup
  ON hr.PunchLog (TenantId, CompanyId, EmployeeCode, DeviceId, PunchTimeUtc);

-- One attendance row per employee per day
CREATE UNIQUE INDEX UX_AttendanceDay ON hr.AttendanceDay (TenantId, EmployeeId, [Date]);

-- Employment records must not start twice on the same date (non-overlap checked in domain + trigger)
CREATE UNIQUE INDEX UX_Employment_Start ON hr.EmploymentRecord (TenantId, EmployeeId, EffectiveFrom);

-- Leave ledger is append-only: no update/delete permission for the application role
-- Balance view (conceptual)
-- SELECT EmployeeId, LeaveTypeId, Year, SUM(Days) AS Balance FROM hr.LeaveLedger GROUP BY ...

-- Unique business keys per tenant
CREATE UNIQUE INDEX UX_Employee_No ON hr.Employee (TenantId, CompanyId, EmployeeNo);
CREATE UNIQUE INDEX UX_Employee_NidHash ON hr.Employee (TenantId, NidBlindIndex) WHERE NidBlindIndex IS NOT NULL;
```

### 9.4 Indexing and Performance
- `EmploymentRecord (TenantId, EmployeeId, EffectiveFrom DESC)` for "current record" lookups; a computed "current employment" view or cached current columns on `Employee` for list screens.
- `AttendanceDay (TenantId, Date, DepartmentId)` for the daily board; monthly sheet pages by employee.
- `LeaveRequest (TenantId, EmployeeId, Status)` and `(TenantId, StartDate, EndDate)` for overlap and calendar queries.
- Attendance computation and imports run as **background jobs**, batched by company and date range.

### 9.5 Concurrency and Consistency
- Optimistic concurrency on employee, request and policy rows.
- Leave submission/approval writes ledger reservation and request state in one transaction; duplicate approval clicks are idempotent.
- Attendance finalization takes an exclusive lock on the period row; edits check the lock.
- Event processing is idempotent via the inbox.

---

## 10. API Design (ERP.Api, controllers + Cortex.Mediator)

Base: `/api/v1/hr`. JSON, JWT, tenant from the token. Errors as `ProblemDetails` with stable codes (for example `HR.LEAVE_OVERLAP`, `HR.INSUFFICIENT_BALANCE`, `HR.PERIOD_FINALIZED`). `Idempotency-Key` supported on submit/approve commands.

| Area | Endpoints (examples) |
|---|---|
| Setup | `GET/POST /designations`, `/grades`, `/employee-types`, `/shifts`, `/work-calendars`, `/holiday-calendars`, `/settings` |
| Employees | `GET /employees`, `GET /employees/{id}`, `POST /employees` (draft/create), `PUT /employees/{id}`, `POST /employees/import`, `GET /employees/{id}/timeline`, `POST /employees/{id}/reveal` (sensitive field, logged) |
| Lifecycle | `POST /employees/{id}/transfer`, `/promote`, `/confirm`, `/suspend`, `/reinstate`, `/separate`, `/rehire` |
| Documents/Letters | `POST /employees/{id}/documents`, `GET /employees/{id}/letters`, `POST /letters/generate` |
| Attendance | `POST /punches/import`, `POST /punches` (device push, API key), `POST /attendance/check-in`, `/check-out`, `GET /attendance/day?date=`, `GET /attendance/monthly?month=`, `PUT /attendance/{id}` (with reason), `POST /attendance/recompute`, `POST /attendance-periods/{id}/finalize`, `/reopen` |
| Regularization/OT | `POST /regularizations`, `POST /overtime-requests`, `GET /approvals/inbox` |
| Leave | `GET /leave/types`, `GET /leave/balances?employeeId=`, `POST /leave/requests/preview`, `POST /leave/requests`, `POST /leave/requests/{id}/cancel`, `/approve`, `/reject`, `GET /leave/calendar`, `POST /leave/adjustments`, `POST /leave/year-end/preview`, `/execute` |
| Loans | `POST /loans`, `/loans/{id}/approve`, `/reschedule`, `/settle`, `GET /loans` |
| Offboarding | `POST /offboarding`, `GET /offboarding/{id}`, `PUT /offboarding/tasks/{taskId}` |
| Light modules | `/vacancies`, `/candidates`, `/appraisal-cycles`, `/trainings` |
| Reports/Dashboard | `GET /reports/headcount`, `/attendance-summary`, `/leave-balance`, `/attrition`; `GET /dashboard/summary` |
| Self-service | `GET /me`, `PUT /me/profile`, `GET /me/leave`, `GET /me/attendance`, `GET /me/approvals` |

Cortex.Mediator naming: `CreateEmployeeCommand`, `TransferEmployeeCommand`, `SubmitLeaveRequestCommand`, `ApproveLeaveRequestCommand`, `FinalizeAttendancePeriodCommand`, `GetLeaveBalanceQuery`, `GetMonthlyAttendanceQuery`.

---

## 11. Non-Functional Requirements (HR-Specific)

| Area | Requirement |
|---|---|
| Performance | Employee list (50 rows) < 500 ms; leave preview < 300 ms; leave submit < 400 ms; daily attendance board < 1 s for 500 employees; import 50,000 punches < 60 s (background); monthly compute for 500 employees < 30 s |
| Scalability | Sized for up to ~500 employees per tenant and ~200 tenants in year one (master A8); background jobs per tenant with fair scheduling |
| Privacy and data protection | Encrypt sensitive columns; mask by default; access log for reveals/exports; retention policy and data export/erasure procedure per applicable data-protection law; consent record for photo/biometric use |
| Security | Permission checks in `AuthorizationBehavior`; manager visibility limited to reporting subtree; API keys for device push scoped per tenant, rotatable; no PII in logs |
| Auditability | Every change recorded (who, when, before/after); view-log for sensitive fields; immutable leave ledger |
| Reliability | Idempotent imports and events; job retries with backoff; finalization is transactional |
| Availability | Business hours target aligned with master NFR; self-service degrades gracefully (read-only) during maintenance |
| Accessibility and UX | WCAG 2.2 AA; supported browsers: latest two versions of Chrome, Edge, Firefox, Safari; mobile Chrome/Safari |
| Maintainability | Architecture tests; Domain coverage >= 85%, Application >= 75% |

### 11.1 Observability (Serilog)
- Log commands with name, tenant, user, duration, outcome; **never** log NID, bank numbers, DOB, medical data or reasons for leave at `Information` level.
- Enrich with `TenantId`, `CompanyId`, `CorrelationId`, `EmployeeId` (id only, not name).
- Metrics: punches ingested, unmatched punches, attendance job duration, pending approvals age, outbox lag, event failures. Alerts: attendance job failure, outbox lag > 1 minute, unmatched punches spike.

---

## 12. Testing Strategy

| Level | Focus |
|---|---|
| Domain unit tests | Leave day counting (weekly off, holidays, half-day, sandwich), accrual and pro-rata, carry-forward, attendance status resolution, late/overtime rules, effective-dated employment, reporting-line cycle detection |
| Property-based tests | "Ledger sum always equals reported balance after any sequence of grant/request/cancel/approve"; "no overlapping approved leave for one employee"; "employment records never overlap" |
| Application tests | Handlers with fakes: permissions, workflow routing, idempotency, event handling |
| Integration tests | EF mappings, RLS, unique indexes, concurrency (double-approve, parallel finalize), outbox/inbox, import dedup |
| Contract tests | Event schemas for `EmployeeJoined`, `AttendanceFinalized`, `LoanApproved` consumed by Payroll tests |
| Scenario tests | Hire-to-Pay (HR to Payroll), leave to attendance to payroll days, offboarding to final settlement |
| Golden dataset | Known month of punches, leaves and holidays with expected attendance summary |
| UI tests | Playwright: apply leave on a mobile viewport, approve from inbox, add employee wizard, finalize attendance; axe accessibility checks |
| Privacy tests | Role matrix against sensitive fields; reveal logging; export masking |
| Tenant isolation | Two tenants; cross-tenant read/write attempts via API, jobs, files |
| Performance | 500 employees x 30 days attendance; 50k punch import; concurrent leave submissions |

---

## 13. Delivery Plan for the HR Module

| Release | Scope | Exit criteria |
|---|---|---|
| **H1 Foundation** | Module skeleton, setup (designations, grades, types), employee master, effective-dated employment, documents, directory, profile, add-employee wizard, audit, field-level privacy, `EmployeeJoined/Changed` events | AC-4, AC-7, AC-8 pass; Payroll can build its employee read model |
| **H2 Leave** | Holidays and work calendar, leave types/policies/ledger, apply/approve with workflow, team calendar, ESS home and manager inbox (mobile-first) | AC-1, AC-2, AC-3 pass; usability targets for leave met |
| **H3 Attendance** | Shifts, punches (import, API, web check-in), daily computation, regularization, overtime, monthly sheet, finalize and `AttendanceFinalized` | AC-5, AC-6 pass; Hire-to-Pay scenario with Payroll green |
| **H4 Lifecycle and exit** | Confirmation/transfer/promotion flows, contracts and expiry alerts, letters, loans, offboarding, `EmployeeSeparated`, loan events | Loan and final-settlement scenarios green |
| **H5 Light modules and polish** | Recruitment, appraisal, training, reports and dashboards, bulk import, PWA polish, accessibility audit | SUS >= 75; accessibility audit passed |

---

## 14. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Wrong leave/attendance rules cause payroll errors | High | Policy templates verified with HR, golden datasets, preview screens, ledger transparency |
| Labour law and company policy variations | High | Everything configurable and effective-dated; templates labeled "verify" |
| Biometric device formats vary widely | Medium | Flexible import mapping per device, REST push endpoint, unmatched punches workflow |
| Privacy breach of personal data | High | Encryption, masking, access logs, role matrix tests, no PII in logs |
| HR-Payroll data mismatch | High | Versioned events, idempotent consumers, payroll blocked until attendance finalized |
| Low employee adoption of self-service | Medium | Mobile-first UX, simple invitation flow, SMS/email reminders, onboarding tips |
| Scope creep into full HCM (recruitment, performance) | Medium | Keep those as light "Could" items after core releases |

---

## 15. Definition of Done (Module)

- All **Must** requirements in section 5 implemented; AC-1 to AC-8 pass.
- Architecture tests prove HR references no other module's projects.
- HR to Payroll contracts verified with contract and scenario tests.
- Every screen in 8.4 implemented, accessible (axe clean, keyboard walkthrough) and verified on mobile and desktop.
- Leave ledger reproducibility verified; attendance golden dataset passes.
- Sensitive data encrypted, masked and access-logged; no PII in logs.
- Bangladesh Labour Act template reviewed and signed off by a domain expert (HR/legal) before release.
- Usability targets in 8.10 met with real users.
- Documentation: API (Swagger), event contracts, policy configuration guide, user guides for HR, managers and employees.

---

## 16. Open Questions

1. Which sector(s) are first customers in (factory/RMG, shop/commercial, services)? Labour Act rules differ.
2. Typical attendance source: biometric devices (which brands/formats), card, or web/mobile check-in only?
3. Should managers see all indirect reports or only direct reports by default?
4. Is employee ID card printing or access-card integration required in v1?
5. SMS/WhatsApp notifications needed in v1, or email and in-app only?
6. Do customers need multiple leave policies by employee group at launch (for example workers vs staff)?
7. Retention periods for resigned employee records and documents (policy/legal requirement)?

---

## 17. Glossary

| Term | Meaning |
|---|---|
| ESS / MSS | Employee / Manager Self-Service |
| Effective-dated | A record valid from a start date until the next change, preserving history |
| Leave ledger | Append-only list of leave grants and usage; balance is the sum |
| Regularization | Employee request to correct or explain attendance |
| Roster | Assignment of shifts to employees over time |
| Sandwich rule | Weekly offs/holidays between leave days count as leave |
| Finalization | Locking a month's attendance and publishing it to Payroll |
| Clearance | Checklist of returned assets and handovers at exit |
| Punch | A raw in/out time record from a device or check-in |
| WPPF | Workers' Profit Participation Fund (handled in Payroll) |
| Outbox / Inbox | Tables ensuring reliable, idempotent messaging between modules |
