# 05 — Employee Management

**Requirement 5** — covers all 8 acceptance criteria for employee accounts, outlet assignment, schedules, attendance (clock in/out, late, overtime), and audited corrections.

> **User Story:** As a Business Owner, I want to manage employee accounts, schedules, and attendance, so that I can oversee my workforce across outlets.

## Acceptance Criteria

1. WHEN the Business_Owner submits a valid new employee account, THE System SHALL create the employee account with the assigned Role.
2. WHEN the Business_Owner submits an edit to an employee account, THE System SHALL update the account with the submitted values.
3. WHEN the Business_Owner or Outlet_Manager assigns an employee to one or more Outlets, THE System SHALL record the Outlet assignment for that employee.
4. WHEN the Business_Owner or Outlet_Manager configures a working schedule or Shift for an employee, THE System SHALL store the schedule for that employee.
5. WHEN an employee clocks in or clocks out, THE System SHALL record the event with its timestamp in the employee's Timesheet.
6. WHEN an employee clocks in after the scheduled Shift start time, THE System SHALL record the event as a late arrival.
7. WHEN an employee works beyond the scheduled Shift end time, THE System SHALL record the additional duration as overtime.
8. WHEN an Outlet_Manager approves an attendance correction, THE System SHALL update the affected Timesheet with the corrected values and record the approving user in the Audit_Log.

## Module Interface

```typescript
interface EmployeeService {
  createEmployee(input: EmployeeInput): Promise<Employee>;   // 5.1
  updateEmployee(id: ID, input: EmployeeInput): Promise<Employee>; // 5.2
  assignOutlets(id: ID, outletIds: ID[]): Promise<void>;     // 5.3
  setSchedule(id: ID, schedule: ScheduleInput): Promise<Schedule>; // 5.4
  clock(id: ID, event: 'in' | 'out', at: Timestamp): Promise<TimesheetEntry>; // 5.5, late 5.6, overtime 5.7
  approveAttendanceCorrection(entryId: ID, correction: Correction, approver: ID): Promise<TimesheetEntry>; // 5.8 (audited)
}
```

Late arrival (5.6) and overtime (5.7) are derived by a pure function `classifyAttendance(scheduledShift, clockIn, clockOut)` so the classification can be property-tested.

## Data Model Entities

```typescript
interface Employee { id: ID; name: string; role: Role; active: boolean; }
type Role = 'BUSINESS_OWNER' | 'OUTLET_MANAGER' | 'CASHIER';
interface EmployeeOutlet { employeeId: ID; outletId: ID; } // (5.3)

interface Schedule { id: ID; employeeId: ID; shiftStart: TimeOfDay; shiftEnd: TimeOfDay; days: Weekday[]; }

interface TimesheetEntry {
  id: ID; employeeId: ID; date: DateOnly;
  clockIn?: Timestamp; clockOut?: Timestamp;
  lateMinutes: number;      // derived (5.6)
  overtimeMinutes: number;  // derived (5.7)
  absent: boolean;
  approvedBy?: ID;          // (5.8)
}
```

## Notes

- **`classifyAttendance` (5.6, 5.7):** A pure function that derives `lateMinutes` and `overtimeMinutes` from the scheduled shift and actual clock-in/clock-out times. Being side-effect-free, it is a primary property-test target.
- **Audited corrections (5.8):** When an Outlet Manager approves an attendance correction, the affected `TimesheetEntry` is updated and the approving user is recorded in the Audit_Log. See `10-SECURITY-ACCESS.md` for the `AuditLog` entity and `appendAuditLog`.

## Cross-References

- Derived attendance data feeds payroll calculation (6.3) — see `06-PAYROLL.md`.
- Absences and late arrivals trigger attendance-issue notifications (9.5) — see `09-NOTIFICATIONS.md`.
- Outlet assignment (5.3) interacts with RBAC outlet scope (10.4) — see `10-SECURITY-ACCESS.md`.
