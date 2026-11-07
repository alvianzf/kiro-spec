# Implementation Plan: Cashier / Point-of-Sale for Business Owner

**Version:** V4 (see CHANGELOG.md)

## Overview

Implementation is in TypeScript (NestJS + Prisma/PostgreSQL backend, React clients, Vitest + fast-check). The V3 brand theme (primary #990000, secondary #000000, accent #FFFFFF) is delivered as a shared `@app/theme` package created right after scaffolding, and all client tasks build on it. The theme is presentation only and adds no requirements. V4 adds a limit of 2 concurrent sessions per user and 1 session per device: a persisted `sessions` table with `deviceId` and a revocation reason (`CONCURRENT_LIMIT` or `DEVICE_REPLACED`), pure `applySessionLimit` and `parseDeviceId` cores (Property 16), an atomic login with device replacement first and oldest-first revocation second, a distinct `SESSION_REVOKED` error carrying the reason, status/event endpoints, a persisted client `deviceId`, and Dashboard and POS handling of revocation (criteria 10.14-10.29). The plan builds the pure domain core first (the main target of property-based tests), then the persistence layer and backend modules, then the Dashboard and POS clients, and ends with wiring.

The design document has no dedicated "Correctness Properties" section, so the properties below are derived from its Key Invariants and pure-function contracts. Each property is numbered here and lives in its own test file under `src/domain/__properties__/`. Run property tests with `fast-check` (minimum 100 runs each).

| Property | Statement | Requirements |
| --- | --- | --- |
| P1 | Cart totals: `total = subtotal - discountTotal + taxTotal` and `sum(lineTotal) = subtotal` | 4.2, 4.3 |
| P2 | Change due equals tendered minus total, never negative for sufficient tender | 4.4 |
| P3 | Split payment is valid iff the sum of payments equals the total | 4.5 |
| P4 | Stock ledger: initial quantity plus the sum of movement deltas equals current quantity, and quantity is never negative | 1.5, 2.6, 3.6 |
| P5 | Transfer conserves total quantity across source and destination | 1.6 |
| P6 | Alerts: low-stock iff quantity <= threshold, out-of-stock iff quantity = 0 | 1.8, 1.9 |
| P7 | Valuation equals the sum of quantity x unit cost | 1.11 |
| P8 | Attendance: late minutes and overtime minutes are non-negative and correctly classified against the scheduled shift | 5.6, 5.7 |
| P9 | Payroll net equals gross + overtime + allowances + bonuses - deductions | 6.3 |
| P10 | Payroll outstanding equals net - paid, and payments never produce a negative balance | 6.5 |
| P11 | Expected cash balance equals opening + cash sales - cash refunds; discrepancy iff closing differs | 4.8, 9.4 |
| P12 | Session expiry is a local 00:00 in the Business_Timezone, after issue time, with no local midnight in between | 10.7, 10.11, 10.13 |
| P13 | `isSessionExpired` is true iff `now >= expiresAt` (monotonic in `now`) | 10.8, 10.9, 10.10 |
| P14 | Authorization grants only permitted operations and outlet scope restricts non-owner roles to assigned outlets | 10.3, 10.4 |
| P15 | Idempotent recording: repeated submissions with one `transactionUid` yield exactly one record | 4.10, 12.2 |
| P16 | Concurrent session limit: after a new login at most 2 sessions are active and at most 1 per `deviceId`; all active same-device sessions are revoked as `DEVICE_REPLACED`; `CONCURRENT_LIMIT` revokes exactly the earliest remaining other-device sessions by `(issuedAt, id)` and only when needed; expired and revoked sessions are ignored; results are deterministic | 10.14, 10.15, 10.16, 10.17, 10.18, 10.21 |

## Tasks

- [ ] 1. Set up project structure and persistence
  - [ ] 1.1 Scaffold the project
    - Create a NestJS TypeScript workspace with Vitest and fast-check
    - Add shared types: `ID`, `Money` (integer minor units), `Quantity`, `Timestamp`, `Role`, `OutletScope`, `IanaTimezone`
    - Create directories `src/domain`, `src/modules`, `src/domain/__properties__`
    - _Requirements: 13.2_

  - [ ] 1.2 Define the Prisma schema and migrations
    - Model all entities from the design, including `Session` and `BusinessSettings` (default `Asia/Jakarta`)
    - Add unique constraints on `Transaction.transactionUid`, `Product.sku`, `Product.barcode`
    - Add check constraint `StockItem.quantity >= 0`
    - Add outlet-scoped foreign keys and audit log table
    - Add a `sessions` table (id, userId, role, outletIds, deviceId, deviceLabel, issuedAt, expiresAt, revokedAt, revokedReason, revokedBySessionId) with `revokedReason` as an enum `CONCURRENT_LIMIT | DEVICE_REPLACED` (null for silently closed expired rows)
    - Add a partial index on (userId, issuedAt, id) WHERE revokedAt IS NULL, and a partial unique index on (userId, deviceId) WHERE revokedAt IS NULL
    - _Requirements: 1.1, 2.1, 2.7, 4.10, 10.6, 10.7, 10.14, 10.16, 10.19, 10.22, 12.2, 13.1_

  - [ ] 1.3 Create the shared `@app/theme` package
    - Define tokens once from a single source and emit them as CSS custom properties on `:root` and as a typed `Theme` object (`as const`), using the token values in the design
    - Add the `PERMITTED_PAIRS` table of allowed foreground/background pairs with minimum ratios (4.5:1 normal text, 3:1 large text and UI boundaries); primary on secondary is not listed
    - Add the theme provider (CSS variables plus React context) and base styles, including the two-tone focus ring that is never removed
    - Presentation only; gives the Login_Page and both clients one consistent look
    - _Requirements: 10.9, 10.10_

  - [ ]* 1.4 Write the contrast unit test for the theme package
    - Vitest test computing WCAG contrast ratios for every `PERMITTED_PAIRS` entry against its minimum (4.5:1 normal text, 3:1 large text and UI boundaries)
    - Assert that the forbidden pair (primary on secondary/black) is not present in `PERMITTED_PAIRS`
    - _Requirements: none (presentation only, design V3 Testing Note)_

  - [ ] 1.5 Add the colour-literal lint rule
    - ESLint/stylelint rule banning hex, `rgb()`, and `hsl()` colour literals outside the theme package
    - Wire the rule into the workspace lint script
    - _Requirements: none (presentation only, design V3 Central Application)_

- [ ] 2. Implement the pure domain core
  - [ ] 2.1 Implement cart and payment calculations
    - `updateCart`, `recalculateCart`, `calculateChange`, `validateSplitPayment` as pure functions with descriptive validation errors
    - _Requirements: 4.2, 4.3, 4.4, 4.5, 4.6, 15.2_

  - [ ]* 2.2 Write property test for cart totals
    - **Property 1: Cart totals**
    - **Validates: Requirements 4.2, 4.3**

  - [ ]* 2.3 Write property test for change calculation
    - **Property 2: Change due**
    - **Validates: Requirements 4.4**

  - [ ]* 2.4 Write property test for split payments
    - **Property 3: Split payment validity**
    - **Validates: Requirements 4.5**

  - [ ] 2.5 Implement inventory calculations
    - `applyMovement`, `transferStock`, `evaluateStockAlerts`, `calculateValuation`
    - Reject movements that would make quantity negative
    - _Requirements: 1.5, 1.6, 1.8, 1.9, 1.11_

  - [ ]* 2.6 Write property test for the stock ledger
    - **Property 4: Stock ledger consistency**
    - **Validates: Requirements 1.5, 2.6, 3.6**

  - [ ]* 2.7 Write property test for stock transfers
    - **Property 5: Transfer conservation**
    - **Validates: Requirements 1.6**

  - [ ]* 2.8 Write property test for stock alerts
    - **Property 6: Low-stock and out-of-stock alerts**
    - **Validates: Requirements 1.8, 1.9**

  - [ ]* 2.9 Write property test for inventory valuation
    - **Property 7: Valuation**
    - **Validates: Requirements 1.11**

  - [ ] 2.10 Implement attendance and payroll calculations
    - `classifyAttendance`, `calculatePayrollRecord`, `applyPayrollPayment`
    - _Requirements: 5.6, 5.7, 6.3, 6.5_

  - [ ]* 2.11 Write property test for attendance classification
    - **Property 8: Attendance classification**
    - **Validates: Requirements 5.6, 5.7**

  - [ ]* 2.12 Write property test for payroll net pay
    - **Property 9: Payroll net**
    - **Validates: Requirements 6.3**

  - [ ]* 2.13 Write property test for payroll outstanding balance
    - **Property 10: Payroll outstanding**
    - **Validates: Requirements 6.5**

  - [ ] 2.14 Implement shift and notification helpers
    - `calculateExpectedBalance`, `detectDiscrepancy`, `resolveRecipients`
    - _Requirements: 4.7, 4.8, 9.1, 9.2, 9.4_

  - [ ]* 2.15 Write property test for expected balance and discrepancy
    - **Property 11: Cash balance and discrepancy**
    - **Validates: Requirements 4.8, 9.4**

  - [ ] 2.16 Implement session and access core
    - `computeSessionExpiry(issuedAt, tz)` using an IANA timezone library (no fixed offsets)
    - `isSessionExpired(session, now)`, `authorize`, `resolveOutletScope`
    - _Requirements: 10.3, 10.4, 10.7, 10.8, 10.11, 10.13_

  - [ ]* 2.17 Write property test for session expiry computation
    - **Property 12: Session expiry boundary**
    - Generate instants and timezones, including DST zones and instants at and near midnight
    - **Validates: Requirements 10.7, 10.11, 10.13**

  - [ ]* 2.18 Write property test for expiry detection
    - **Property 13: Expired-session detection**
    - **Validates: Requirements 10.8, 10.9, 10.10**

  - [ ]* 2.19 Write property test for authorization and outlet scope
    - **Property 14: Role authorization and outlet scope**
    - **Validates: Requirements 10.3, 10.4**

  - [ ] 2.20 Implement the session limit core
    - Pure `applySessionLimit(activeSessions, newSession, now, limit)` returning `{ kept, revoked: [{ session, reason }] }`, and `isSessionActive(session, now)`, with `MAX_CONCURRENT_SESSIONS = 2`
    - Drop expired and revoked sessions first
    - Device replacement next: revoke every remaining session with the new session's `deviceId` as `DEVICE_REPLACED`; these do not count toward the limit
    - Then the limit: sort the other remaining sessions by `(issuedAt asc, id asc)` and revoke from the front as `CONCURRENT_LIMIT` until `remaining + 1 <= limit`
    - Add pure `parseDeviceId(raw)` returning a UUID v4 string, or `undefined` when missing or malformed
    - _Requirements: 10.14, 10.15, 10.16, 10.17, 10.18, 10.21, 10.22_

  - [ ]* 2.21 Write property test for the concurrent session limit
    - **Property 16: Concurrent session limit**
    - Generate mixed active, expired, and revoked sessions with mixed `deviceId` values, including equal `issuedAt` values, and a new login; assert kept + new has at most 2 active sessions and at most 1 per `deviceId`
    - Assert every active session on the new `deviceId` is revoked as `DEVICE_REPLACED`, `CONCURRENT_LIMIT` revocations are exactly the earliest remaining other-device sessions by `(issuedAt, id)` and occur only when needed, expired and revoked inputs are never counted or revoked, and repeated calls on the same input give the same result
    - **Validates: Requirements 10.14, 10.15, 10.16, 10.17, 10.18, 10.21**

  - [ ]* 2.22 Write unit tests for `parseDeviceId`
    - Accepts UUID v4; rejects missing, empty, non-string, and malformed values
    - _Requirements: 10.22_

- [ ] 3. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement authentication, RBAC, and daily session expiry (server)
  - [ ] 4.1 Implement AccessService and the auth/RBAC guard
    - `authenticate` sets `expiresAt` via `computeSessionExpiry`, returns `expiresAt` and `serverNow`
    - Guard checks session expiry before permissions and returns `401 SESSION_EXPIRED`, distinct from `401 INVALID_CREDENTIALS` and `403`
    - No refresh or sliding renewal past the Day_Boundary
    - Inject outlet scope, apply role and per-user permissions, append Audit_Log for financial, payroll, and security operations
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.11_

  - [ ] 4.2 Implement Business_Timezone setting
    - `setBusinessTimezone` (Business_Owner only, audited), read at login so only new sessions are affected
    - _Requirements: 10.6, 10.13_

  - [ ]* 4.3 Write unit tests for the guard and login flow
    - Expired session rejected without performing the operation, invalid credentials, re-login yields new expiry, timezone change affects only new sessions
    - _Requirements: 10.2, 10.8, 10.11, 10.13_

  - [ ] 4.4 Implement atomic login with session limit and revoked-session guard
    - Extend `AccessService.authenticate(credentials, deviceId?)` (4.1), in this order:
      - Verify credentials and return `401 INVALID_CREDENTIALS` before reading or writing any session
      - Resolve `deviceId = parseDeviceId(input) ?? randomUUID()`
      - Open one database transaction and take a per-user lock (`SELECT ... FOR UPDATE` or `pg_advisory_xact_lock`)
      - Load all non-revoked rows for the user (active and expired) and call `applySessionLimit` (2.20) on the active ones
      - Close rows: mark revoked sessions with `revokedAt`, `revokedReason` (`DEVICE_REPLACED` or `CONCURRENT_LIMIT`), and `revokedBySessionId`; silently close expired non-revoked rows on the same `deviceId` (`revokedAt = now`, no reason, no event, no audit entry) so the unique index never blocks a login
      - Insert the new session with its `deviceId` and expiry
      - Append one Audit_Log entry per revocation (reason, user, `deviceId`, revoked session id, new session id, timestamp)
      - Commit, then publish `SESSION_REVOKED` with the reason for each revoked session
    - All session queries are scoped to the authenticated `userId`; a submitted `deviceId` is never used for authorisation
    - Extend the guard (4.1): after the expiry check, reject a revoked session with `401 { code: SESSION_REVOKED, reason }`, distinct from `SESSION_EXPIRED`, `INVALID_CREDENTIALS`, and `403`, without performing the operation
    - _Requirements: 10.14, 10.15, 10.16, 10.17, 10.18, 10.19, 10.20, 10.21, 10.22, 10.23, 10.24, 10.29_

  - [ ] 4.5 Implement session status and events endpoints
    - `GET /session/status` returning `{ status: 'ACTIVE' }`, `{ status: 'EXPIRED' }`, or `{ status: 'REVOKED', reason }` for the caller's session
    - `GET /session/events` Server-Sent Events stream that emits `{ type: 'SESSION_REVOKED', sessionId, reason, revokedAt }` when the session is revoked
    - _Requirements: 10.24, 10.25, 10.26_

  - [ ]* 4.6 Write tests for the session limit on the server
    - Integration tests against real PostgreSQL:
      - Concurrent logins by one user from the same device leave exactly 1 active session for that device
      - Concurrent logins from different devices never leave more than 2 active sessions
      - A missing or forged `deviceId` cannot push a user above 2 active sessions, and a `deviceId` held by another user's session does not touch that user's sessions
      - Device replacement revokes the old session with reason `DEVICE_REPLACED` and writes one Audit_Log entry with reason, `deviceId`, user, revoked and new session ids, and timestamp; `CONCURRENT_LIMIT` revocations are audited the same way
      - An expired non-revoked row on the same device is closed silently (no event, no audit entry), so the login is not blocked by the unique index
    - Failed login leaves all sessions unchanged; a revoked session gets `401 SESSION_REVOKED` with its reason, no operation runs, and `GET /session/status` returns `REVOKED` with the reason; expired sessions are not counted
    - _Requirements: 10.16, 10.17, 10.19, 10.20, 10.21, 10.22, 10.23, 10.24, 10.29_

- [ ] 5. Implement inventory module
  - [ ] 5.1 Implement InventoryService
    - Stock item CRUD, levels, movements with history, transfers in one database transaction, thresholds, valuation, outlet scoping
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.10, 1.11_

  - [ ] 5.2 Wire stock alerts to notifications
    - Call `evaluateStockAlerts` after every quantity change and emit alert events
    - _Requirements: 1.8, 1.9, 9.1, 9.2_

- [ ] 6. Implement product module
  - [ ] 6.1 Implement ProductService
    - Product CRUD, duplicate SKU/barcode rejection, outlet assignment, variations, sales performance, best sellers
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.7, 2.8, 2.9_

- [ ] 7. Implement sales and transactions
  - [ ] 7.1 Implement `recordTransaction`
    - Single database transaction: insert keyed on `transactionUid`, deduct linked stock with row locks, record income, commit
    - On unique violation return the original record; on partial failure roll back with an error and no stock or finance changes
    - _Requirements: 2.6, 4.10, 7.3, 11.1, 12.1, 12.2, 12.3_

  - [ ] 7.2 Implement transaction queries, refunds, shifts, and receipts
    - List and filter by outlet, date, product, employee; detail view; sales summary
    - Adjustments and refunds require Business_Owner or Outlet_Manager authorisation; refunds restore linked stock and trigger unusual-activity notification above the threshold
    - Shift open/close, receipt generation
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 4.7, 4.8, 4.9, 9.3, 9.4_

  - [ ]* 7.3 Write property test for idempotent recording
    - **Property 15: Idempotent transaction recording**
    - **Validates: Requirements 4.10, 12.2**

  - [ ]* 7.4 Write integration tests for atomicity and concurrency
    - Forced mid-transaction failure leaves stock and income unchanged; concurrent sales from multiple outlets lose no records and never oversell
    - _Requirements: 11.1, 12.1, 12.3_

- [ ] 8. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement remaining backend modules
  - [ ] 9.1 Implement EmployeeService
    - Account CRUD with roles, outlet assignment, schedules, clock in/out with late and overtime classification, audited attendance correction approval
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8_

  - [ ] 9.2 Implement PayrollService
    - Compensation and adjustments, payroll run from approved timesheets, summary and payslips, payments, history
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ] 9.3 Implement AccountingService
    - Income and expense entries with categories, liabilities, cash balances per outlet, income statement / P&L / cash flow reports, audited manual adjustments
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7_

  - [ ] 9.4 Implement NotificationService
    - Low-stock, out-of-stock, unusual activity, cash discrepancy, attendance issue, pending payment, backup failure, using `resolveRecipients`
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 14.4_

  - [ ] 9.5 Implement BackupService
    - Scheduled backup job to the owner-designated store, list, restore, failure notification
    - _Requirements: 14.1, 14.2, 14.3, 14.4_

  - [ ] 9.6 Implement DashboardService and exports
    - Overview, sales, product, inventory, workforce views with period and outlet filters, summary tables refreshed on transaction sync, PDF/Excel/CSV export
    - _Requirements: 3.2, 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 11.2_

- [ ] 10. Implement client applications
  - [ ] 10.1 Implement Dashboard session expiry handling
    - Store `clockOffset` from `serverNow`; `useSessionExpiry` timer redirects to `/login?returnTo=...`
    - Re-check on `visibilitychange`, `focus`, `online`, and every 30 seconds
    - HTTP interceptor for `401 SESSION_EXPIRED`, route guard on all Dashboard routes, cross-tab sync via `BroadcastChannel`/`storage`, same-origin `returnTo` after re-login
    - _Requirements: 10.9, 10.10, 10.11_

  - [ ]* 10.2 Write unit tests for Dashboard session handling
    - Fake timers for redirect at expiry, background-tab re-check, interceptor redirect, guard blocks routes without a valid session, cross-tab redirect
    - _Requirements: 10.9, 10.10_

  - [ ] 10.3 Implement Dashboard views
    - Overview, sales, products, inventory, workforce, transactions, employees, payroll, accounting, notifications, and settings (including Business_Timezone), with report export
    - Build on the themed Dashboard shell (10.8) and shared status components (10.7); no colour literals
    - _Requirements: 1.4, 3.1, 5.1, 6.6, 7.4, 8.1, 8.8, 9.1, 10.13_

  - [ ] 10.4 Implement POS interface
    - Single workflow for search, cart, payment (cash, bank transfer, debit, credit, QRIS, split), and receipt
    - Shift open/close, inline descriptive validation messages, local confirmation within 2 seconds
    - Use the shared status components (10.7) for validation messages; theme styling is applied in 10.9
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 11.3, 15.1, 15.2_

  - [ ] 10.5 Implement POS offline queue and sync
    - IndexedDB queue keyed by outlet and device, never cleared on expiry or logout
    - `401 SESSION_EXPIRED` and `401 SESSION_REVOKED` (either reason) leave items queued; sync resumes after re-login with `transactionUid` retries
    - _Requirements: 4.10, 10.12, 10.28, 12.2_

  - [ ]* 10.6 Write unit tests for the POS queue
    - Queue retained across expiry, revocation with both reasons, and logout; sync resumes after re-login; no duplicates on retry
    - _Requirements: 10.12, 10.28_

  - [ ] 10.7 Implement shared status components
    - Status component with icon + text label + border pattern: error, out-of-stock, discrepancy (2px strong border, solid left bar, primary-tint background), low-stock and warning (dashed warning border), success (check icon, success border)
    - Form field error component with icon, message linked through `aria-describedby`, and border pattern
    - Destructive action button with icon, verb label, and a confirmation dialog
    - Status never relies on colour alone; all colours come from `@app/theme` tokens
    - _Requirements: 15.2, 1.8, 1.9, 9.4_

  - [ ] 10.8 Apply the theme to the Dashboard shell and Login_Page
    - Black top bar and sidebar with white text and icons; active nav item shown as white label plus white indicator bar or primary-filled pill with white text, never red on black
    - Primary buttons (primary fill, white text, hover and pressed tokens) and white secondary buttons with strong border
    - Login_Page uses the same theme provider so expiry redirects land on a consistent page
    - _Requirements: 10.9, 10.10_

  - [ ] 10.9 Apply the theme to the POS interface
    - Primary action buttons (for example Complete Sale), white surfaces, black text, two-tone focus ring on all interactive elements
    - Replace any hard-coded colours; lint rule from 1.5 must pass
    - _Requirements: 4.1, 4.9, 15.1, 15.2_

  - [ ] 10.10 Implement themed charts, exports, and receipts
    - Charts: series palette from tokens (primary, black, greys, primary-tint) with pattern fills, direct labels or markers, and distinct line styles
    - PDF export: theme header band with white title text, token-based table styling read from the typed `Theme` object
    - Excel export: header row with primary fill and white bold text; CSV unstyled; data never encoded by colour alone
    - Receipts (printed and shared): black on white only, no tinted backgrounds; brand colour only in an optional on-screen preview header
    - _Requirements: 8.8, 3.8, 4.9_

  - [ ]* 10.11 Write component tests for status components and themed shell
    - Status components render icon, label, and border pattern for each state; field errors link via `aria-describedby`; destructive button requires confirmation
    - Active nav item on the black sidebar uses white text; printed receipt output contains no coloured backgrounds
    - _Requirements: 15.2, 1.8, 1.9, 9.4, 3.8, 4.9_

  - [ ] 10.12 Implement Dashboard handling of session revocation
    - Subscribe to `/session/events` (SSE); while the tab is visible, poll `GET /session/status` every 3 seconds as a fallback when the channel is closed or errors
    - Re-check on `visibilitychange`, `focus`, and `online`; the HTTP interceptor maps `401 SESSION_REVOKED` like `SESSION_EXPIRED`
    - Clear the session and redirect within 5 seconds by revocation reason, showing the message through the shared status component (10.7), with icon and text:
      - `CONCURRENT_LIMIT` goes to `/login?reason=signed_in_elsewhere` with "Your account was signed in on another device."
      - `DEVICE_REPLACED` goes to `/login?reason=signed_in_again` with "You were signed out because you signed in again on this device."
    - Own-login rule: store the new session id on login and ignore any revocation event or status whose `sessionId` is not the currently stored session id, so the tab that just logged in is not redirected; other tabs adopt the new session id through the broadcast
    - Route guard blocks revoked sessions on all Dashboard routes; revocation is broadcast across tabs via `BroadcastChannel`/`storage`
    - _Requirements: 10.24, 10.25, 10.26, 10.27_

  - [ ]* 10.13 Write unit tests for Dashboard revocation handling
    - Fake timers: redirect within 5 seconds with push up and with push down (polling fallback); interceptor redirect; route guard blocks a revoked session; cross-tab redirect
    - Both messages render with icon and text and route to the matching `reason` query value
    - A revocation event or status for a stale session id is ignored, so the tab that performed the new login is not redirected
    - _Requirements: 10.25, 10.26, 10.27_

  - [ ] 10.14 Generate, persist, and send the device identifier in both clients
    - Generate a random UUID v4 `deviceId` once and persist it: Dashboard in `localStorage` with a first-party cookie fallback when storage is unavailable; POS app in installation storage
    - Send `deviceId` in every login request; the server treats it as a hint and replaces a missing or malformed value (4.4)
    - Needed so a repeat login on the same device is recognised as a Device_Replacement
    - _Requirements: 10.16, 10.22_

- [ ] 11. Integration and wiring
  - [ ] 11.1 Wire all modules together
    - Register modules and guard globally, connect sales to inventory, accounting, and notifications, and clients to the API
    - Verify adding an Outlet, employees, products, or stock items needs no reconfiguration of existing records
    - _Requirements: 13.1, 13.2, 3.2, 10.3, 10.4_

  - [ ]* 11.2 Write end-to-end integration tests
    - Sale to stock deduction to income to dashboard; refund restores stock; role and outlet isolation; login, midnight expiry, redirect, re-login; login on devices A, B, then A again keeps B and replaces A's first session (`DEVICE_REPLACED`, signed-in-again message); a login on a third distinct device revokes the oldest session, which is redirected to the Login_Page with the signed-in-elsewhere message
    - _Requirements: 2.6, 3.6, 7.3, 8.1, 10.3, 10.4, 10.7, 10.8, 10.11, 10.16, 10.17, 10.25, 10.26_

- [ ] 12. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability, including V2 criteria 10.7-10.13 (tasks 2.16-2.19, 4.1-4.3, 10.1-10.2, 10.5-10.6, 11.2)
- Property tests validate universal correctness properties from the table above; unit and integration tests cover specific examples and edge cases
- V4 session-limit tasks cover criteria 10.14-10.29: 1.2 (sessions table with `deviceId`, reason, unique index), 2.20-2.22 (pure `applySessionLimit`, `parseDeviceId`, Property 16, unit tests), 4.4-4.6 (atomic login with device replacement, revoked guard with reason, status/events endpoints, tests), 10.5-10.6 (POS queue retention on revocation), 10.12-10.13 (Dashboard revocation handling for both reasons), 10.14 (client `deviceId`), 11.2 (device-sequence end-to-end test). Existing task numbers are unchanged; 2.22 and 10.14 are new
- Requirements 14.1 (owner-controlled hosting) is addressed by configurable database and backup targets in tasks 9.5 and 1.2
- V3 theme tasks (1.3-1.5, 10.7-10.11) are presentation only and add no requirements. The theme package (1.3) is built first; the Dashboard and POS tasks (10.3, 10.4, 10.8-10.10) depend on it and on the shared status components (10.7)
- Components must use `@app/theme` tokens only; the lint rule (1.5) and contrast test (1.4) guard this. Full accessibility validation still needs manual testing with assistive technologies

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "1.3", "2.1", "2.5", "2.10", "2.14", "2.16", "2.20"] },
    { "id": 2, "tasks": ["1.4", "1.5", "2.2", "2.3", "2.4", "2.6", "2.7", "2.8", "2.9", "2.11", "2.12", "2.13", "2.15", "2.17", "2.18", "2.19", "2.21", "2.22", "4.1"] },
    { "id": 3, "tasks": ["4.2", "4.3", "4.4", "5.1", "6.1", "9.1", "9.3", "9.4", "9.5", "10.7"] },
    { "id": 4, "tasks": ["4.5", "4.6", "5.2", "7.1", "9.2"] },
    { "id": 5, "tasks": ["7.2", "7.3", "7.4", "9.6"] },
    { "id": 6, "tasks": ["10.1", "10.4", "10.8", "10.14"] },
    { "id": 7, "tasks": ["10.2", "10.3", "10.5", "10.9", "10.12"] },
    { "id": 8, "tasks": ["10.6", "10.10", "10.11", "10.13"] },
    { "id": 9, "tasks": ["11.1"] },
    { "id": 10, "tasks": ["11.2"] }
  ]
}
```
