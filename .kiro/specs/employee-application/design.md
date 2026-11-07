# Design Document: Employee Application

## Overview

The Employee Application is the employee-facing portion of the Cashier & Business Management
Application. It exposes a role-aware set of features — dashboard, attendance/timesheet, work schedule,
point of sale (POS), payroll, leave management, notifications, and profile — all scoped so that an
employee can only access their own data and the functions their role permits.

This design describes the architecture, module boundaries, data models, key interactions, and
cross-cutting concerns (permissions, auditing, synchronisation) needed to satisfy the requirements.

### Goals

- Deliver a simple, fast, role-aware employee experience.
- Enforce strict data scoping: an employee sees only their own records.
- Keep POS operations responsive and resilient to brief connectivity loss.
- Record all sensitive actions in a central activity log.
- Keep employee activity synchronised with the central business database.

### Assumptions

- A shared backend and identity/authorisation service already exist for the business/admin side.
- Roles and permissions are defined centrally; the employee app consumes them.
- The POS catalogue, pricing, and payment integrations are owned by shared services.

## Architecture

```
+-------------------------------------------------------------+
|                     Employee Client (UI)                    |
|  Dashboard | Attendance | Schedule | POS | Payroll | Leave  |
|          Notifications | Profile | Auth/Session             |
+-------------------------------+-----------------------------+
                                | HTTPS / REST (JSON)
                                v
+-------------------------------------------------------------+
|                      API / BFF Layer                        |
|   AuthN/AuthZ middleware  |  Permission guard  |  Audit log  |
+---+------------+----------+-----------+------------+---------+
    |            |          |           |            |
    v            v          v           v            v
+--------+  +---------+  +--------+  +---------+  +-----------+
|Employee|  |Attendance| | Schedule| |  POS    |  |  Payroll  |
|Service |  | Service  | | Service | | Service |  |  Service  |
+--------+  +---------+  +--------+  +---------+  +-----------+
    |            |          |           |            |
    +------------+----------+-----+-----+------------+
                                  v
                    +-----------------------------+
                    | Central Business Database   |
                    +-----------------------------+
                    | Notification | Sync workers |
                    +-----------------------------+
```

### Layers

- **Employee Client (UI):** Feature modules, each mapping to a requirement area. Keeps minimal local
  state; relies on the API layer for data.
- **API / BFF Layer:** Authenticates requests, enforces per-employee scoping and role permissions,
  and writes audit entries for sensitive actions. Aggregates data for the dashboard.
- **Domain Services:** Employee, Attendance, Schedule, POS, Payroll, Leave, Notification. Each owns
  its data and business rules.
- **Central Business Database:** System of record shared with the business/admin side.
- **Sync & Notification Workers:** Propagate employee activity and deliver notifications.

## Components and Interfaces

### Authentication & Session
- Employee logs in with an assigned account; session carries employee id, role, outlet, and
  permission set.
- All feature modules read the permission set to show/hide and enable/disable actions.

### Dashboard Module (Req 1)
- Aggregates read-only summaries: profile, outlet/position, schedule/upcoming shifts, today's
  attendance status, recent activity, notifications, and pending tasks/approvals.
- Interface: `GET /me/dashboard` returns a composed view assembled by the BFF from the relevant
  services.

### Attendance & Timesheet Module (Req 2)
- Clock in/out, daily records, totals, late/absence/overtime, weekly/monthly timesheets, correction
  requests, leave/absence requests, and request status.
- Interfaces:
  - `POST /me/attendance/clock` (in/out)
  - `GET /me/attendance?range=...`
  - `GET /me/timesheets?period=weekly|monthly`
  - `POST /me/attendance/corrections`
  - `POST /me/requests` (leave/absence)
  - `GET /me/requests`

### Work Schedule Module (Req 3)
- Assigned shifts, upcoming days, change notifications, approved swaps/adjustments, and permitted
  shift change requests.
- Interfaces:
  - `GET /me/schedule?range=...`
  - `POST /me/schedule/change-requests` (gated by permission)

### POS Module (Req 4)
- Outlet product catalogue, search (name/SKU/barcode), transaction creation, authorised discounts,
  supported payments, receipt generation/share, personal transaction history, authorised
  refunds/cancellations, and cashier shift open/close with cash balances.
- Interfaces:
  - `GET /outlets/{id}/products`, `GET /outlets/{id}/products/search?q=...`
  - `POST /pos/transactions`
  - `POST /pos/transactions/{id}/refund` (gated)
  - `GET /me/pos/transactions`
  - `POST /pos/shifts/open`, `POST /pos/shifts/close`
- Discounts, refunds, and cancellations are permission-gated and audited.

### Payroll Module (Req 5)
- Current salary, monthly summaries, breakdown (basic, allowances, bonuses, overtime, deductions),
  payslip view/download, and payment history (pending/completed). Read-only.
- Interfaces:
  - `GET /me/payroll/summary?month=...`
  - `GET /me/payslips`, `GET /me/payslips/{id}/download`
  - `GET /me/payroll/payments`

### Leave Management Module (Req 6)
- Annual/sick/other requests, date selection, supporting document upload, leave balances, status
  tracking, and history.
- Interfaces:
  - `POST /me/leave` (with optional document attachments)
  - `GET /me/leave/balances`
  - `GET /me/leave` (list with status)

### Notifications Module (Req 7)
- Attendance reminders, schedule updates, leave status, payroll/payment notices, and announcements.
- Interfaces:
  - `GET /me/notifications`
  - `POST /me/notifications/{id}/read`
- Delivered via notification workers; the client subscribes/polls for updates.

### Profile Module (Req 8)
- View personal/employment info, update permitted fields, change password, view role/outlet, and
  employment history.
- Interfaces:
  - `GET /me/profile`
  - `PATCH /me/profile` (permitted fields only)
  - `POST /me/password`

## Data Models

```
Employee        { id, name, email, role, outletId, position, status, hireDate }
EmploymentHistory { id, employeeId, position, outletId, startDate, endDate }
AttendanceRecord{ id, employeeId, date, clockIn, clockOut, workedHours, status(late|absent|present|overtime) }
Timesheet       { employeeId, period, totals, records[] }
Request         { id, employeeId, type(correction|leave|absence|shiftChange), payload, status(pending|approved|rejected), createdAt, decidedAt }
Shift           { id, employeeId, outletId, start, end, status, swapInfo }
Product         { id, outletId, name, sku, barcode, price, available }
Transaction     { id, cashierId, outletId, lines[], discounts[], payments[], total, status, createdAt }
CashierShift    { id, cashierId, outletId, openedAt, openingBalance, closedAt, closingBalance }
PayrollSummary  { employeeId, month, basic, allowances, bonuses, overtime, deductions, net }
Payslip         { id, employeeId, month, fileRef }
SalaryPayment   { id, employeeId, month, amount, status(pending|completed), paidAt }
LeaveRequest    { id, employeeId, type, startDate, endDate, documents[], status }
LeaveBalance    { employeeId, type, entitlement, used, remaining }
Notification    { id, employeeId, type, message, read, createdAt }
AuditLogEntry   { id, actorId, action, target, metadata, createdAt }
```

## Access Control & Permissions (Req 9)

- **Scoping:** Every `/me/*` endpoint resolves records by the authenticated employee id; cross-employee
  access is rejected at the API layer.
- **Permission guard:** POS discounts/refunds/cancellations, shift change requests, and profile edits
  are gated by the permission set on the session. Missing permission yields an authorisation error.
- **Forbidden scopes:** Business-wide sales, accounting, financial reports, and other employees'
  payroll are not exposed to the employee app at all.
- **Immutability:** Approved attendance and payroll records are read-only for employees; changes go
  through correction/request workflows.
- **Manager elevation:** Manager role grants additional permissions via the central permission set;
  the app reacts to the permission set rather than hard-coding role checks.
- **Auditing:** The audit middleware records sensitive actions (clock events, POS transactions,
  refunds, shift open/close, profile/password changes, request submissions) to `AuditLogEntry`.

## Cross-Cutting Concerns (Req 10)

- **Usability:** Clear, task-focused screens; large primary actions (clock in/out, open/close shift);
  accessible components and labels.
- **Performance:** Attendance and POS endpoints prioritised; product search uses indexed lookups on
  name/SKU/barcode; receipts generated asynchronously where possible.
- **Security:** Enforced AuthN/AuthZ, least-privilege permission checks, encrypted transport, and no
  exposure of out-of-scope data.
- **Availability:** Schedule/attendance/employment reads served from cache-friendly endpoints; POS
  tolerates brief connectivity loss and queues actions for sync.
- **Data Synchronisation:** Sync workers reconcile queued POS/attendance actions and propagate changes
  to the central business database.

## Error Handling

- Permission failures return a standard authorisation error without leaking data existence.
- Validation errors (e.g., invalid clock sequence, overlapping leave) return field-level messages.
- POS actions performed offline are queued and retried; conflicts surface for review rather than
  silently overwriting.
- All write failures on sensitive actions are logged.

## Testing Strategy

- **Unit tests:** Permission guard logic, attendance totals/overtime calculation, payroll breakdown
  rendering, leave balance computation.
- **Integration tests:** `/me/*` scoping (employee cannot read another employee's data), POS
  transaction + shift open/close flow, leave request lifecycle, audit log writes for sensitive actions.
- **End-to-end tests:** Clock in/out, process a sale with discount and refund (authorised vs
  unauthorised), submit and track a leave request, view and download a payslip.
- **Non-functional checks:** Response-time thresholds for attendance and POS endpoints; offline POS
  queue-and-sync behaviour.
