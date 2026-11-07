# 06 — Payroll Management

**Requirement 6** — covers all 6 acceptance criteria for compensation configuration, adjustments, payroll calculation from approved timesheets, payslips, payments, and history.

> **User Story:** As a Business Owner, I want payroll calculated from attendance and approved timesheets, so that I can pay employees accurately and keep payroll records.

## Acceptance Criteria

1. WHEN the Business_Owner configures an employee's salary, hourly rate, or daily wage, THE System SHALL store the configured compensation for that employee.
2. WHEN the Business_Owner configures allowances, bonuses, overtime rates, or deductions for an employee, THE System SHALL store those values for payroll calculation.
3. WHEN the Business_Owner initiates payroll calculation for a pay period, THE System SHALL calculate each employee's Payroll_Record from the configured compensation and the approved Timesheet for that period.
4. WHEN payroll calculation for a pay period completes, THE System SHALL generate a monthly payroll summary and an individual payslip for each included employee.
5. WHEN the Business_Owner records a salary payment against a Payroll_Record, THE System SHALL mark the paid amount and recalculate the outstanding payroll balance.
6. WHEN a user with payroll access requests historical payroll, THE System SHALL display the Payroll_Records for the selected pay periods.

## Module Interface

```typescript
interface PayrollService {
  setCompensation(employeeId: ID, comp: Compensation): Promise<void>; // 6.1
  setAdjustments(employeeId: ID, adj: PayrollAdjustments): Promise<void>; // 6.2
  runPayroll(period: PayPeriod): Promise<PayrollBatch>; // 6.3 from comp + approved timesheets
  generateOutputs(batchId: ID): Promise<{ summary: PayrollSummary; payslips: Payslip[] }>; // 6.4
  recordPayment(recordId: ID, amount: Money): Promise<PayrollRecord>; // 6.5 recalcs outstanding
  getHistory(filter: PeriodFilter): Promise<PayrollRecord[]>; // 6.6
}
```

`calculatePayrollRecord(compensation, adjustments, timesheet)` is a pure function — the core of property testing for payroll.

## Data Model Entities

Monetary fields are integers in minor currency units.

```typescript
interface Compensation { employeeId: ID; basis: 'SALARY' | 'HOURLY' | 'DAILY'; baseAmount: Money; }
interface PayrollAdjustments { employeeId: ID; allowances: Money; bonuses: Money; overtimeRate: Money; deductions: Money; }

interface PayrollRecord {
  id: ID; employeeId: ID; period: PayPeriod;
  gross: Money; overtimePay: Money; allowances: Money; bonuses: Money; deductions: Money;
  net: Money;                 // gross + overtime + allowances + bonuses - deductions
  paidAmount: Money;          // (6.5)
  outstanding: Money;         // net - paidAmount (6.5)
}
```

## Invariants

- `PayrollRecord.net == gross + overtimePay + allowances + bonuses - deductions`.
- `PayrollRecord.outstanding == net - paidAmount`.

## Notes

- **`calculatePayrollRecord` (6.3):** A pure function taking configured compensation, adjustments, and the approved timesheet for the period. It is side-effect-free and the primary property-test target for payroll correctness.
- Payroll calculation consumes **approved** timesheets only — see `05-EMPLOYEES.md` for attendance derivation (late/overtime minutes).

## Cross-References

- Outstanding balances on or after their due date trigger pending-payment notifications (9.6) — see `09-NOTIFICATIONS.md`.
- Payroll summaries appear in the workforce dashboard view (8.5) — see `08-DASHBOARD-REPORTS.md`.
- Payroll record operations are recorded in the Audit_Log (10.6) — see `10-SECURITY-ACCESS.md`.
