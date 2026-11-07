# Implementation Plan: Employee Application

**Version:** V3 (see CHANGELOG.md)

## Overview

Implementation plan for the employee-facing application, including the V2 brand theme work and the V3 session revocation handling.

## Tasks

- [ ] 1. Set up project foundation and shared infrastructure
  - Create the employee app module structure (client feature modules + API/BFF entry points).
  - Configure routing, HTTP client, and a central place to read the session permission set.
  - _Requirements: 10.1, 10.3_
  - [ ] 1.1 Integrate the shared `@app/theme` package
    - Wrap the client in the theme provider and consume CSS custom properties and the typed `Theme` object; do not define any hex values in the employee client.
    - If the package is unavailable, theme work is blocked until it is provided.
    - _Requirements: 10.1_
  - [ ] 1.2 Add a colour-literal lint rule
    - Flag colour literals in components and styles outside the theme package.
    - _Requirements: 10.1_
  - [ ]* 1.3 Write Vitest contrast test for theme token pairs
    - Compute WCAG ratios for every permitted pair (4.5:1 normal text, 3:1 large text and UI boundaries).
    - Assert primary-on-black is forbidden and absent from the permitted pairs.
    - _Requirements: 10.1_

- [ ] 2. Authentication, session, and permissions
  - [ ] 2.1 Implement employee login and session bootstrap
    - Authenticate against the shared identity service and store employee id, role, outlet, and permission set in the session.
    - (V3) Expose hooks only: the login call accepts an optional `deviceId` argument and includes it in the request when provided, and the session bootstrap exposes a post-login hook that stores the new session id (replacing any earlier one) and lets later code run after login.
    - (V3) The deviceId generation and sending is done in task 16.6, and the detector registration through the post-login hook is done in task 16.2. Task 2.1 does not depend on either.
    - _Requirements: 4.1, 9.1, 10.3_
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
    - Render the composed view with accessible, task-focused layout using theme tokens and the shared status components.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 10.1_

- [ ] 5. Attendance & Timesheet
  - [ ] 5.1 Implement clock in/out
    - `POST /me/attendance/clock`, validate sequence, record timestamps and audit.
    - Present clock in/out as a large primary action using theme tokens.
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
    - Use theme tokens and the shared status components for POS and authorisation errors; receipts follow task 15.2.
    - _Requirements: 4.4, 4.6, 4.7_
  - [ ] 7.3 Implement authorised discounts, refunds, and cancellations
    - Gate by permission and audit each action.
    - _Requirements: 4.5, 4.9, 9.2, 9.6_
  - [ ] 7.4 Implement personal transaction history
    - `GET /me/pos/transactions` scoped to the cashier.
    - _Requirements: 4.8, 9.1_
  - [ ] 7.5 Implement cashier shift open/close with cash balances
    - Record opening/closing balances and shift events.
    - Present open/close shift as large primary actions using theme tokens.
    - _Requirements: 4.10, 4.11, 9.6_
  - [ ] 7.6 Implement offline queue-and-sync for POS actions
    - Queue actions during brief connectivity loss and reconcile on reconnect.
    - (V3) Keep the queue keyed by employee id and retain it when the session is revoked; see task 16.4.
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

- [ ] 13. Shared status components
  - [ ] 13.1 Build status components (icon + text label + border pattern)
    - Request pending, approved, rejected (correction, leave, absence, shift change).
    - Attendance late, absent, overtime; salary payment pending, completed.
    - Status never relies on colour alone.
    - _Requirements: 2.5, 2.8, 5.6, 10.1_
  - [ ] 13.2 Build error, offline, and confirmation components
    - POS and authorisation errors with explicit message, error icon, and strong border with left bar, linked via `aria-describedby`; field-level errors use the same linkage.
    - Offline/queued POS state ("Offline, N actions queued") switching to "Synced" when the queue clears.
    - Destructive confirmation (refund, cancel) with icon and verb label.
    - Adopt these components in the attendance, requests, payroll, and POS screens.
    - (V3) Also use the error/status component for both session revocation messages ("Your account was signed in on another device." and "You were signed out because you signed in again on this device."), see task 16.3.
    - _Requirements: 4.9, 9.2, 10.1, 10.4_
  - [ ]* 13.3 Write component tests for status rendering
    - Assert each status renders its icon, text label, and border pattern, and that errors are linked via `aria-describedby`.
    - _Requirements: 2.5, 2.8, 4.9, 5.6, 9.2, 10.1_

- [ ] 14. Apply theme to the app shell
  - [ ] 14.1 Style the top bar and navigation
    - Black top bar and navigation with white labels; active item as a primary-filled pill with white text; never primary text, icons, or borders on black.
    - Two-tone focus rings are never removed.
    - _Requirements: 10.1_
  - [ ] 14.2 Style primary actions
    - Large primary buttons for clock in/out and open/close shift, with hover and pressed tokens; secondary buttons white with a strong black border.
    - _Requirements: 10.1, 2.1, 4.10, 4.11_

- [ ] 15. Black-on-white payslips and receipts
  - [ ] 15.1 Render payslips black on white
    - On-screen and downloaded payslips use a plain header; brand colour only in an optional on-screen header band, never in the downloaded file.
    - _Requirements: 5.4_
  - [ ] 15.2 Render receipts black on white
    - Printed and shared receipts have no tinted backgrounds, for thermal-printer legibility.
    - _Requirements: 4.7_

- [ ] 16. Session revocation handling
  - [ ] 16.1 Confirm or provide idempotency-key support for clock events and POS submissions
    - The client generates an idempotency key for each clock event and POS submission and keeps it with the queued action.
    - Verify that the attendance and POS services accept and honour the keys. If they do not, raise it as a blocker for task 16.4.
    - _Requirements: 2.1, 4.4, 10.4, 10.5_
  - [ ] 16.2 Implement the revocation detector
    - Subscribe to SSE `/session/events` while a session is active; events carry `sessionId` and `reason`.
    - Fall back to polling `GET /session/status` every 3 seconds while the page is visible; pause while hidden. A response `{ status: 'REVOKED', reason }` signals revocation.
    - Run an immediate status check on `visibilitychange` (visible), `focus`, and `online`.
    - Add a global 401 interceptor that maps `{ code: 'SESSION_REVOKED', reason }` to the revocation flow, separate from `SESSION_EXPIRED` and `INVALID_CREDENTIALS`.
    - Pass `reason` and `sessionId` (when present) from every source to the revocation flow in task 16.3.
    - Register the detector through the post-login hook exposed by task 2.1, so it starts after each successful login and uses the session id stored by that hook.
    - Depends on 2.1 (post-login hook) and 16.6 (login sends the `deviceId`).
    - _Requirements: 9.1, 10.3, 10.1_
  - [ ] 16.3 Implement the revocation flow
    - Own-login rule: ignore any revocation event, status response, or broadcast whose `sessionId` is not the current session's id; evaluate those without a `sessionId` against the current session. For a 401 `SESSION_REVOKED` on a request sent with an older session token, retry it once with the current session instead of redirecting.
    - Otherwise stop SSE and polling, and clear the session, tokens, and cached per-employee data. Keep the persisted `deviceId`.
    - Redirect to login within 5 seconds of the server revoking the session, by reason:
      - `CONCURRENT_LIMIT`: `/login?reason=signed_in_elsewhere`, "Your account was signed in on another device."
      - `DEVICE_REPLACED`: `/login?reason=signed_in_again`, "You were signed out because you signed in again on this device."
      - An unknown or missing reason falls back to the `CONCURRENT_LIMIT` variant.
    - Show the message through the shared status component (task 13.2) with icon, text, and `aria-describedby` linkage.
    - Add a route guard that blocks authenticated routes and `/me/*` calls for a revoked session.
    - Broadcast the revocation, including `sessionId` and `reason`, to other tabs with `BroadcastChannel` so they redirect together to the same variant; receiving tabs apply the own-login rule first.
    - The client writes no audit entry for revocations.
    - _Requirements: 9.1, 10.1, 10.3_
  - [ ] 16.4 Retain and replay queued work on revocation
    - Keep queued POS and clock actions in local storage keyed by employee id; the revocation flow does not clear them.
    - After re-login, replay them with their original idempotency keys, only under the owning account; leave them untouched if a different user logs in.
    - Put in-flight writes that fail with `SESSION_REVOKED` for the current session back in the queue. Surface server-rejected actions for review.
    - Extends task 7.6.
    - _Requirements: 10.4, 10.5, 2.1_
  - [ ] 16.6 Generate, persist, and send a deviceId on login
    - On first run generate a UUID v4 `deviceId` and persist it in `localStorage`, with a cookie fallback when storage is unavailable, or in installation storage for an installed app.
    - Wire into the optional `deviceId` argument of the login call exposed by task 2.1 and pass it with every login request. Reuse it across logins, tabs, and revocations; the revocation flow does not clear it.
    - If a valid id cannot be persisted, still send the one generated for this run.
    - The client makes no decision with it: it does not count sessions, choose which to revoke, or infer a reason.
    - _Requirements: 10.3_
  - [ ]* 16.5 Write tests for session revocation
    - Use Vitest fake timers.
    - Push path (SSE) for both reasons: `CONCURRENT_LIMIT` redirects to `/login?reason=signed_in_elsewhere` and `DEVICE_REPLACED` to `/login?reason=signed_in_again`, each within 5 seconds with the matching message.
    - Polling path with SSE down, both reasons: a `{ status: 'REVOKED', reason }` response redirects to the matching variant within 5 seconds. Polling pauses while hidden, and `visibilitychange`, `focus`, and `online` trigger an immediate check.
    - Interceptor, both reasons: 401 `SESSION_REVOKED` triggers the flow; a missing or unknown reason falls back to the `CONCURRENT_LIMIT` variant; `SESSION_EXPIRED` and `INVALID_CREDENTIALS` do not trigger it.
    - Stale-session rule: after a new login, an event, status response, or broadcast with the old `sessionId` does not clear the new session or redirect, while one with the current `sessionId` does. A 401 `SESSION_REVOKED` for a request sent with the old token is retried once with the current session and does not redirect.
    - deviceId lifecycle: first run generates a UUID v4, persists it (localStorage with cookie fallback), sends the same id on every login, keeps it through the revocation flow, and makes no session-counting decision.
    - Route guard blocks a revoked session; the cross-tab broadcast (with `sessionId` and `reason`) redirects other tabs to the same variant.
    - Queue retention: queued actions survive revocation, replay with the same idempotency keys without duplicates, and are not replayed under another user.
    - Messages render through the status component with icon and text.
    - _Requirements: 9.1, 10.1, 10.3, 10.4, 10.5_

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP.
- Each task references specific requirements for traceability.
- V2 adds the theme tasks 1.1-1.3 and 13-15; existing task numbering is unchanged.
- V3 adds task 16 (16.1-16.6) and extends the text of 2.1, 7.6, and 13.2; existing numbering is unchanged.
- V3 session revocation is design-only behaviour. The employee requirements have no criterion for concurrent sessions, so tasks 16.x cite the closest existing requirements (9.1 scoping, 10.1 usability, 10.3 security, 10.4 availability, 10.5 synchronisation, 2.1 and 4.4 for the queued actions). The session limit itself is enforced by the shared identity service, not this app.
- The shared service decides which session is revoked and why (`CONCURRENT_LIMIT` or `DEVICE_REPLACED`). The client only sends the `deviceId` (16.6), reacts to the reason, and ignores revocations for its own earlier session (16.3).
- Task 16.6 comes before 16.2 and 16.3 in the dependency graph. Task 2.1 (wave 0) only exposes the hooks (optional `deviceId` login argument and post-login hook); 16.6 supplies the `deviceId` and 16.2 registers the detector through those hooks, both after 2.1.
- Task 16.4 depends on 16.1. If the attendance or POS services do not accept idempotency keys, 16.4 is blocked until they do.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "2.1"] },
    { "id": 1, "tasks": ["1.2", "2.2", "14.1"] },
    { "id": 2, "tasks": ["1.3", "2.3", "4.1", "5.1", "6.1", "7.1", "8.1", "9.1", "10.1", "11.1", "13.1"] },
    { "id": 3, "tasks": ["4.2", "5.2", "6.2", "7.2", "8.2", "9.2", "10.2", "11.2", "13.2", "14.2"] },
    { "id": 4, "tasks": ["5.3", "6.3", "7.3", "13.3", "15.1", "15.2"] },
    { "id": 5, "tasks": ["7.4", "12.1"] },
    { "id": 6, "tasks": ["7.5"] },
    { "id": 7, "tasks": ["7.6"] },
    { "id": 8, "tasks": ["12.2", "12.3", "16.1", "16.6"] },
    { "id": 9, "tasks": ["16.2"] },
    { "id": 10, "tasks": ["16.3", "16.4"] },
    { "id": 11, "tasks": ["16.5"] }
  ]
}
```
