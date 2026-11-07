# Implementation Plan: Employee Application

- [ ] 1. Set up project foundation and shared infrastructure
  - Create the employee app module structure (client feature modules + API/BFF entry points).
  - Configure routing, HTTP client, and a central place to read the session permission set.
  - _Requirements: 10.1, 10.3_

- [ ] 2. Authentication, session, and permissions
  - [ ] 2.1 Implement employee login and session bootstrap
    - Authenticate against the shared identity service and store employee id, role, outlet, and permission set in the session.
    - _Requirements: 4.1, 9.1_
  - [ ] 2.2 Implement the permission guard and per-employee scoping middleware
    - Reject cross-employee access on all `/me/*` endpoints and gate permission-controlled actions.
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5_
  - [ ] 2.3 Implement audit logging for sensitive actions
    - Write `AuditLogEntry` records for clock events, POS actions, refunds, shift open/close, profile/password changes, and request submissions.
    - _Requirements: 9.6_

- [ ] 3. Data models and persistence
  - Define and persist the core models (Employee, AttendanceRecord, Request, Shift, Product, Transaction, CashierShift, PayrollSummary, Payslip, SalaryPayment, LeaveRequest, LeaveBalance, Notification, AuditLogEntry).
  - _Requirements: 10.5_

- [ ] 4. Employee Dashboard
  - [ ] 4.1 Implement the `GET /me/dashboard` aggregation in the BFF
    - Compose profile, outlet/position, schedule/upcoming shifts, today's attendance, recent activity, notifications, and pending tasks/approvals.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6_
  - [ ] 4.2 Build the dashboard UI
    - Render the composed view with accessible, task-focused layout.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 10.1_

- [ ] 5. Attendance & Timesheet
  - [ ] 5.1 Implement clock in/out
    - `POST /me/attendance/clock`, validate sequence, record timestamps and audit.
    - _Requirements: 2.1, 9.6, 10.2_
  - [ ] 5.2 Implement attendance and timesheet reads
    - Daily records, total hours, late/absence/overtime, weekly/monthly timesheets.
    - _Requirements: 2.2, 2.3, 2.4, 2.5_
  - [ ] 5.3 Implement attendance correction and leave/absence requests with status tracking
    - `POST /me/attendance/corrections`, `POST /me/requests`, `GET /me/requests`.
    - _Requirements: 2.6, 2.7, 2.8_

- [ ] 6. Work Schedule
  - [ ] 6.1 Implement schedule reads
    - Assigned shifts and upcoming days via `GET /me/schedule`.
    - _Requirements: 3.1, 3.2_
  - [ ] 6.2 Implement schedule change notifications and approved swap display
    - Surface schedule-change notifications and show approved swaps/adjustments.
    - _Requirements: 3.3, 3.4_
  - [ ] 6.3 Implement permitted shift change requests
    - `POST /me/schedule/change-requests` gated by permission.
    - _Requirements: 3.5, 9.2_

- [ ] 7. Point of Sale (Cashier)
  - [ ] 7.1 Implement outlet catalogue and product search
    - List products for the assigned outlet and search by name, SKU, or barcode.
    - _Requirements: 4.2, 4.3, 10.2_
  - [ ] 7.2 Implement transaction creation and payments
    - Create transactions, accept supported payment methods, and generate shareable/printable receipts.
    - _Requirements: 4.4, 4.6, 4.7_
  - [ ] 7.3 Implement authorised discounts, refunds, and cancellations
    - Gate by permission and audit each action.
    - _Requirements: 4.5, 4.9, 9.2, 9.6_
  - [ ] 7.4 Implement personal transaction history
    - `GET /me/pos/transactions` scoped to the cashier.
    - _Requirements: 4.8, 9.1_
  - [ ] 7.5 Implement cashier shift open/close with cash balances
    - Record opening/closing balances and shift events.
    - _Requirements: 4.10, 4.11, 9.6_
  - [ ] 7.6 Implement offline queue-and-sync for POS actions
    - Queue actions during brief connectivity loss and reconcile on reconnect.
    - _Requirements: 10.4, 10.5_

- [ ] 8. Payroll & Salary Information (read-only)
  - [ ] 8.1 Implement salary summary and breakdown views
    - Current salary, monthly summaries, and breakdown (basic, allowances, bonuses, overtime, deductions).
    - _Requirements: 5.1, 5.2, 5.3_
  - [ ] 8.2 Implement payslip view/download and payment history
    - `GET /me/payslips`, download endpoint, and pending/completed payment history.
    - _Requirements: 5.4, 5.5, 5.6_

- [ ] 9. Leave Management
  - [ ] 9.1 Implement leave request submission with documents
    - Annual/sick/other requests with date selection and supporting document upload.
    - _Requirements: 6.1, 6.2, 6.3_
  - [ ] 9.2 Implement leave balances, status tracking, and history
    - `GET /me/leave/balances`, `GET /me/leave` with status and history.
    - _Requirements: 6.4, 6.5, 6.6_

- [ ] 10. Notifications
  - [ ] 10.1 Implement notification delivery and feed
    - Attendance reminders, schedule updates, leave status, payroll/payment notices, and announcements via `GET /me/notifications`.
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_
  - [ ] 10.2 Implement mark-as-read
    - `POST /me/notifications/{id}/read`.
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

- [ ] 11. Employee Profile
  - [ ] 11.1 Implement profile view (personal, employment, role/outlet, history)
    - `GET /me/profile` with employment history where available.
    - _Requirements: 8.1, 8.4, 8.5_
  - [ ] 11.2 Implement permitted profile updates and password change
    - `PATCH /me/profile` (permitted fields only) and `POST /me/password`, both audited.
    - _Requirements: 8.2, 8.3, 9.4, 9.6_

- [ ] 12. Cross-cutting hardening and verification
  - [ ] 12.1 Add access-control integration tests
    - Verify an employee cannot read another employee's data and that forbidden scopes are not exposed.
    - _Requirements: 9.1, 9.3, 9.4_
  - [ ] 12.2 Add performance checks for attendance and POS endpoints
    - Assert response-time thresholds for clock and POS operations.
    - _Requirements: 10.2_
  - [ ] 12.3 Verify central database synchronisation
    - Confirm queued employee activity reconciles with the central business database.
    - _Requirements: 10.5_
