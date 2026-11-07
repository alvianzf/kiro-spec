# Design Document: Employee Application

**Version:** V3 (see CHANGELOG.md)

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
- (V2) The shared theme package `@app/theme` (CSS custom properties plus a typed `Theme` object) is the
  single source of truth for brand colours, and the employee client consumes it. If the package is not
  available, the employee client must not define its own hex values; theme work is blocked until the
  package is provided.
- (V3) The shared identity/authorisation service enforces the session policy defined in the
  cashier-business-owner spec (V4, requirements 10.14-10.29 and the design section "Concurrent Session
  Limit and Device Identity (V4, Req 10.14-10.29)"): at most 2 Active_Sessions per user across all roles
  and clients, and at most 1 Active_Session per Device_Identifier. A repeat login from the same device
  replaces that device's session (reason `DEVICE_REPLACED`). Otherwise, a new login at the limit revokes
  the other-device session with the earliest issue time (reason `CONCURRENT_LIMIT`). Employee accounts are
  users subject to the same policy. The employee app only consumes this behaviour. It does not count,
  create, replace, or revoke sessions itself. The exact endpoints and payloads (`GET /session/status`,
  `/session/events` SSE, `401 SESSION_REVOKED`) are owned by the shared service and are referenced from the
  cashier spec. The shapes the client relies on are summarised below under Session Revocation Handling.
- (V3) Clock events and POS submissions carry client-generated idempotency keys so that queued actions can
  be replayed safely. The V1/V2 design relied on queue-and-sync without stating this, so it is recorded
  here as an explicit assumption. If the POS or attendance services do not already accept idempotency keys,
  that support must be provided before the retention behaviour below can be relied on.

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

#### Session Revocation Handling (V3)

The shared identity service allows at most 2 concurrent Active_Sessions per user and 1 per device (see
Assumptions). A session can be revoked for one of two reasons, and the employee client must notice
revocation quickly and leave the app cleanly with the matching message.

**Device identifier (client side):** on first run the client generates a UUID v4 `deviceId`, persists it in
browser storage (`localStorage`, with a cookie fallback when storage is unavailable) or, for an installed
app, in installation storage, and sends it with every login request. The id is reused across logins, tabs,
and revocations, and is not cleared by the revocation flow. The employee client does not use it for any
decision: it does not count sessions, pick which session to revoke, or infer a reason. The server decides,
and the client only reports the id and reacts to the result. If a valid id cannot be persisted, the client
still sends the one it generated for this run, and the server applies its own fallback.

**Payload shapes consumed (owned by the shared service):**
- `401` body: `{ code: 'SESSION_REVOKED', reason: 'CONCURRENT_LIMIT' | 'DEVICE_REPLACED' }`.
- `GET /session/status`: `{ status: 'REVOKED', reason }` when revoked (other statuses are unchanged).
- SSE events on `/session/events`: carry `sessionId` and `reason`.

**Detection (any of the following triggers the same revocation flow):**
- SSE subscription to `/session/events` while a session is active (push).
- Fallback polling of `GET /session/status` every 3 seconds while the page is visible, used when SSE is
  unavailable or disconnected. Polling pauses while the page is hidden.
- Immediate status check on `visibilitychange` (becoming visible), window `focus`, and `online` events.
- A global HTTP 401 interceptor that maps the error code `SESSION_REVOKED` to the revocation flow and reads
  `reason` from the body. It treats `SESSION_EXPIRED` and `INVALID_CREDENTIALS` as separate cases with their
  own handling and messages.

**Own-login (stale sessionId) rule:** the tab that performs a new login replaces its stored session and
ignores any revocation event or status response whose `sessionId` is not the current session's id. This
prevents a device-replacement revocation of the old session, which the server emits as part of that same
login, from signing out the tab that just logged in. Events and statuses without a `sessionId` are
evaluated against the current session as before. The same check applies to the SSE handler, the polling
result, and the cross-tab broadcast. The 401 interceptor ignores `SESSION_REVOKED` for a request that was
sent with an older session token than the current one, and retries it once with the current session
instead of redirecting.

**Revocation flow:**
1. Stop SSE and polling, and clear the in-memory session, tokens, and cached per-employee data. The
   persisted `deviceId` is kept.
2. Redirect to the login screen within 5 seconds of the server revoking the session, using the variant for
   the reason:

   | Reason | Redirect | Message |
   | --- | --- | --- |
   | `CONCURRENT_LIMIT` | `/login?reason=signed_in_elsewhere` | "Your account was signed in on another device." |
   | `DEVICE_REPLACED` | `/login?reason=signed_in_again` | "You were signed out because you signed in again on this device." |

   An unknown or missing reason falls back to the `CONCURRENT_LIMIT` variant.
3. Show the message using the shared status component from the Visual Theme section (icon + text, with
   `aria-describedby` linkage), not colour alone.
4. Broadcast the revocation, including `sessionId` and `reason`, to other tabs of the same browser (for
   example via `BroadcastChannel`) so all open tabs redirect together to the same variant. Receiving tabs
   apply the own-login rule above before acting.

**Route guard:** the client route guard blocks navigation to any authenticated route once the session is
marked revoked, so a stale tab cannot render employee screens or call `/me/*` endpoints.

**Retention of queued work:** revocation does not discard offline work.
- The POS offline queue and queued attendance actions (clock in/out events) stay in local storage, keyed
  by employee id, and are not cleared by the revocation flow.
- After the employee logs in again, the client syncs the queue using the original client-generated
  idempotency keys, so actions already received by the server are not duplicated.
- Queued actions are only replayed under the re-authenticated account that owns them. If a different user
  logs in on the device, the queue is kept untouched and not submitted under that user.
- Server-side authorisation still applies on replay. Rejected actions surface for review (see Error
  Handling).

**Auditing:** revocations are audited server-side by the shared service. The employee client writes no
audit entry for them.

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
  accessible components and labels. (V2) Colours, contrast, and status cues follow the
  "Visual Theme and Design Tokens (V2)" section.
- **Performance:** Attendance and POS endpoints prioritised; product search uses indexed lookups on
  name/SKU/barcode; receipts generated asynchronously where possible.
- **Security:** Enforced AuthN/AuthZ, least-privilege permission checks, encrypted transport, and no
  exposure of out-of-scope data.
- **Availability:** Schedule/attendance/employment reads served from cache-friendly endpoints; POS
  tolerates brief connectivity loss and queues actions for sync.
- **Data Synchronisation:** Sync workers reconcile queued POS/attendance actions and propagate changes
  to the central business database.

## Visual Theme and Design Tokens (V2)

The Employee Application uses the same brand theme as the Business Owner application: primary `#990000`
(deep red), secondary `#000000` (black), accent `#FFFFFF` (white). This is a presentation-only change. It
adds no requirements and does not alter any domain, API, or data behaviour.

### Single Source of Truth

Tokens come from the shared `@app/theme` package, as CSS custom properties on `:root` and as a typed
`Theme` object. Components reference tokens only. Colour literals in components are not allowed and are
flagged by a lint rule. If the package is unavailable, the client must not redefine hex values (see
Assumptions).

| Token | Value | Purpose |
| --- | --- | --- |
| `--color-primary` | `#990000` | Brand colour for primary actions |
| `--color-primary-hover` | `#7A0000` | Hover state |
| `--color-primary-pressed` | `#5C0000` | Pressed/active state |
| `--color-primary-tint` | `#FBEAEA` | Subtle tinted background (selected row, error/authorisation notices) |
| `--color-secondary` | `#000000` | Text, top bar, navigation, strong borders |
| `--color-accent` / `--color-surface` | `#FFFFFF` | Surfaces, and text/icons on dark backgrounds |
| `--color-surface-muted` | `#F5F5F5` | Page background, zebra rows |
| `--color-border` | `#D4D4D4` | Dividers and input borders |
| `--color-text-muted` | `#595959` | Secondary text on light surfaces |
| `--color-success` | `#1B5E20` | Success state |
| `--color-warning` | `#8A5A00` | Warning state |
| `--focus-ring-inner` / `--focus-ring-outer` | `#FFFFFF` / `#000000` | Two-tone focus ring |

Derived values are fixed explicit tokens, not computed at runtime, so they can be contrast-tested.

### Role Assignment

| Colour | Used for |
| --- | --- |
| Primary | Primary actions such as Clock in/out, Open/Close shift, Complete Sale; links; active navigation pill |
| Secondary (black) | Text, top bar, navigation, strong borders |
| Accent (white) | Surfaces, and text/icons on primary or black backgrounds |

Primary buttons use white text with hover and pressed tokens. Secondary buttons are white with a strong
black border. Primary actions (clock in/out, open/close shift, complete sale) are large, in line with the
Usability goal.

### Contrast and Accessibility

| Foreground on background | Ratio (approx.) | Result |
| --- | --- | --- |
| `#990000` on `#FFFFFF` | 8.9:1 | Passes AA and AAA |
| `#FFFFFF` on `#990000` | 8.9:1 | Passes AA and AAA |
| `#000000` on `#FFFFFF` | 21:1 | Passes AA and AAA |
| `#990000` on `#000000` | 2.4:1 | Fails AA |

- Primary text, icons, or borders must never be placed on black. On the black top bar and navigation, use
  white labels with a white indicator bar or a primary-filled pill with white text.
- Only pairs in the theme package's permitted-pairs table may be used (4.5:1 for normal text, 3:1 for
  large text and UI boundaries).
- Focus rings are two-tone and never removed.
- Automated checks cover token pairs only. Full WCAG validation needs manual testing with assistive
  technologies and expert accessibility review.

### Status Never Relies on Colour Alone

Every status renders as icon + text label + border pattern.

| Status | Rendering |
| --- | --- |
| Request pending (correction, leave, absence, shift change) | Clock icon, "Pending" label, dashed neutral border |
| Request approved | Check icon, "Approved" label, `--color-success` border |
| Request rejected | Error icon, "Rejected" label, strong 2px border with left bar on primary tint |
| Attendance late / absent / overtime | Distinct icon and text label each ("Late", "Absent", "Overtime"); absent uses the strong border pattern, late and overtime use the dashed `--color-warning` border |
| Salary payment pending / completed | "Pending" with clock icon and dashed border; "Completed" with check icon and success border |
| POS errors and authorisation errors | Error icon, explicit message (for example "Refund not permitted for your role"), strong border with left bar on primary tint, message linked via `aria-describedby` |
| Offline / queued POS state | Offline icon, "Offline, N actions queued" label, dashed warning border; changes to a success cue with "Synced" when the queue clears |

Destructive actions (refund, cancel) carry an icon and a verb label and require confirmation.

### Payslips and Receipts

- Payslips, on-screen and downloaded, use black on white with a plain header. The brand colour may appear
  only in an optional on-screen header band, never in the downloaded file.
- Printed and shared receipts are black on white only, with no tinted backgrounds, for thermal-printer and
  print legibility.

### Central Application

The theme is applied through the shared theme provider. Per-component colour overrides are not allowed.
Changing a brand value means editing the token source only.

### Testing Note

- A Vitest contrast test computes WCAG ratios for every permitted pair and asserts that primary-on-black is
  absent.
- A lint check asserts that no colour literals appear outside the theme package.
- Component tests assert that each status in the table renders an icon, a text label, and its border pattern.
- Visual and assistive-technology checks are manual.

## Error Handling

- Permission failures return a standard authorisation error without leaking data existence.
- Validation errors (e.g., invalid clock sequence, overlapping leave) return field-level messages.
- POS actions performed offline are queued and retried; conflicts surface for review rather than
  silently overwriting.
- All write failures on sensitive actions are logged.
- (V3) `401 { code: 'SESSION_REVOKED', reason }` means the session was ended by the shared service. The
  client runs the Session Revocation Handling flow: it clears the session, keeps the `deviceId`, retains
  queued POS and attendance actions, and redirects by reason:
  - `CONCURRENT_LIMIT`: `/login?reason=signed_in_elsewhere`, "Your account was signed in on another
    device."
  - `DEVICE_REPLACED`: `/login?reason=signed_in_again`, "You were signed out because you signed in again
    on this device."

  It is distinct from `SESSION_EXPIRED` (timeout) and `INVALID_CREDENTIALS` (failed login), and must not
  be shown as either. A revocation for a stale `sessionId` (the tab's own earlier session) is ignored. In-flight
  writes that fail with `SESSION_REVOKED` for the current session are put back in the queue rather than
  dropped.

## Testing Strategy

- **Unit tests:** Permission guard logic, attendance totals/overtime calculation, payroll breakdown
  rendering, leave balance computation.
- **Integration tests:** `/me/*` scoping (employee cannot read another employee's data), POS
  transaction + shift open/close flow, leave request lifecycle, audit log writes for sensitive actions.
- **End-to-end tests:** Clock in/out, process a sale with discount and refund (authorised vs
  unauthorised), submit and track a leave request, view and download a payslip.
- **Non-functional checks:** Response-time thresholds for attendance and POS endpoints; offline POS
  queue-and-sync behaviour.
- **(V3) Session revocation tests:** Vitest with fake timers.
  - Push path, both reasons: a simulated SSE event with `reason: 'CONCURRENT_LIMIT'` redirects to
    `/login?reason=signed_in_elsewhere` with "Your account was signed in on another device.", and one
    with `reason: 'DEVICE_REPLACED'` redirects to `/login?reason=signed_in_again` with "You were signed
    out because you signed in again on this device.", each within 5 seconds.
  - Polling path, both reasons: with SSE down, a `/session/status` response `{ status: 'REVOKED', reason }`
    is detected on the 3-second poll and redirects to the matching variant within 5 seconds. Polling
    pauses while hidden, and `visibilitychange`, `focus`, and `online` trigger an immediate check.
  - Interceptor: a 401 `{ code: 'SESSION_REVOKED', reason }` response triggers the flow with the matching
    variant for each reason, a missing or unknown reason falls back to the `CONCURRENT_LIMIT` variant, and
    `SESSION_EXPIRED` and `INVALID_CREDENTIALS` do not trigger it.
  - Stale-session rule: after a new login in a tab, a revocation event, status response, or broadcast
    carrying the old `sessionId` does not clear the new session or redirect, while one carrying the
    current `sessionId` does. A 401 `SESSION_REVOKED` for a request sent with the old token is retried
    once with the current session and does not redirect.
  - Device id: the first run generates a UUID v4, persists it (localStorage, with cookie fallback), and
    sends the same id on every login. It survives the revocation flow, and the client makes no
    session-counting decision with it.
  - Route guard and cross-tab: a revoked session cannot reach authenticated routes, and a broadcast
    redirects other tabs to the same variant.
  - Retention: queued POS and clock actions survive revocation, replay with the same idempotency keys
    after re-login without duplicates, and are not replayed under a different user.
  - The message renders through the shared status component with icon and text.
