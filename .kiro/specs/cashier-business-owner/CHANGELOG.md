# Changelog

## V1 - Initial Release
- Initial requirements generated (Requirements 1–15).
- Initial design drafted (architecture, components and interfaces, data models, key invariants).
- Implementation scope not yet established: no tasks.md exists and no implementation has been verified.

## V2 - 2026-11-07
### Type
- Specification change
- Design change
- Breaking change for session behaviour: sessions no longer stay valid indefinitely. They now end at the next business-day boundary.

### Changes
- Sessions expire at the next 00:00 (Day_Boundary) in a configurable Business_Timezone. Users must log in again every calendar day.
- Expired sessions are rejected server-side with a distinct `401 SESSION_EXPIRED` error.
- The Dashboard redirects to the Login_Page as soon as the session expires, with no user action needed. Dashboard routes stay blocked until the user re-authenticates.
- The POS keeps unsynchronised transactions across session expiry and syncs them after re-login.
- Added the Business_Timezone setting, configured by the Business_Owner (default `Asia/Jakarta`).

### Reason
- The Business Owner asked for authentication to reset every day, and for users on the Dashboard to be sent to the login page immediately when their auth expires.

### Impact
- Requirements:
  - Glossary: added Session, Business_Timezone, Day_Boundary, Login_Page.
  - Requirement 10: added acceptance criteria 10.7–10.13. Criteria 10.1–10.6 are unchanged.
- Design components:
  - Request Lifecycle and RBAC: the guard checks session expiry before permissions.
  - Security / Access Module: added `setBusinessTimezone`, pure `computeSessionExpiry` / `isSessionExpired`, and the "Daily Session Expiry" subsection (server guard, Dashboard timer + visibility/focus re-checks + 401 interceptor + route guard + cross-tab sync, POS queue retention).
  - Data Models: documented `Session.expiresAt` derivation and added `BusinessSettings`.
  - Key Invariants: added the session-expiry invariant.
- Implementation tasks:
  - None affected yet (no tasks.md exists). Tasks generated from this design must cover 10.7–10.13.
  - Update (tasks.md created, still V2): tasks.md now exists and is versioned V2. It covers Requirements 1–15, including 10.7–10.13 (tasks 2.16–2.19, 4.1–4.3, 10.1–10.2, 10.5–10.6, 11.2). It adds property-based tests P1–P15 for the pure domain core. No requirements or design content changed, so the version is not incremented. No tasks are implemented or verified yet.

## V3 - 2026-11-07
### Type
- Design change
- Non-breaking: presentation only. No behaviour, API, or data changes.

### Changes
- Added a brand theme for the Dashboard, POS_Interface, and Login_Page: primary `#990000`, secondary `#000000`, accent `#FFFFFF`.
- Added the "Visual Theme and Design Tokens" section: shared design tokens (CSS custom properties plus a typed TypeScript theme object), explicit derived tokens, role assignment, contrast and accessibility rules, status colours with non-colour cues, chart/export/receipt guidance, central theming, and a testing note.
- Added a client styling note to the Technology Stack (CSS variables plus a shared theme package).

### Reason
- The Business Owner asked for a brand theme using the given primary, secondary, and accent colours.

### Impact
- Requirements: none changed. requirements.md is untouched.
- Design components:
  - Technology Stack: client styling approach.
  - New section Visual Theme and Design Tokens (tokens, role assignment, contrast, status colours, charts/exports/receipts, central application, testing note).
  - Dashboard / Reports Module: chart palette and PDF/Excel export styling follow the theme.
  - POS Module and Login_Page: use the shared theme. Receipts stay black on white.
  - Notification and status displays (low-stock, out-of-stock, discrepancy): icon + text + border pattern.
- Implementation tasks: tasks.md is updated next, in a separate step, to cover the theme package, token lint rule, contrast test, and themed components. Nothing in this version is implemented or verified yet.
  - Update (tasks.md now V3): added tasks 1.3 (`@app/theme` package), 1.4* (contrast test), 1.5 (colour-literal lint rule), 10.7 (shared status components), 10.8 (themed Dashboard shell and Login_Page), 10.9 (themed POS), 10.10 (themed charts, PDF/Excel exports, receipts), and 10.11* (component tests). Changed: 10.3 and 10.4 now reference the theme and status components; the Notes and Task Dependency Graph are extended (11.1 moved to wave 9 and 11.2 to wave 10). Existing task numbers are unchanged. Nothing is implemented or verified yet.

## V4 - 2026-11-07
### Type
- Specification change
- Design change
- Breaking change for multi-device behaviour: a user can no longer hold more than 2 simultaneous sessions. A third login signs out the oldest session.

### Changes
- Added a Concurrent_Session_Limit of 2 Active_Sessions per user, regardless of Role, counted across all devices and clients (Dashboard and POS_Interface).
- When a user logs in while already holding 2 Active_Sessions, the Active_Session with the earliest issue time is revoked (ties broken by lowest Session identifier) and the new Session is established.
- Revocation and creation of the new Session are atomic. Concurrent logins never exceed the limit, and failed logins revoke nothing.
- Sessions expired under the Day_Boundary rule (V2) do not count toward the limit. A repeated login from the same device counts as a new Session.
- Revoked sessions are rejected with a distinct session-revoked error, separate from session-expired and invalid-credentials errors.
- The Dashboard redirects to the Login_Page within 5 seconds of revocation, with a message that the account was signed in on another device. Dashboard routes stay blocked until re-authentication.
- The POS keeps unsynchronised transactions across revocation and syncs them after re-login.
- Each revocation is recorded in the Audit_Log.
- Added a one-line note to the Introduction.
- Clarification (same V4 revision, no new version): a repeat login from the same Device replaces that Device's existing Session as a Device_Replacement and does not use a new slot. Criterion 10.21 was revised after clarification and is now 10.16. The limit is now at most 2 Active_Sessions per user and at most 1 Active_Session per Device_Identifier per user. Device replacement is evaluated first, then the 2-session limit applies to the remaining Active_Sessions (earliest issue time revoked, ties by lowest Session identifier).
- Clarification: a login without a valid Device_Identifier is treated as a new Device with a System-generated Device_Identifier, and a submitted Device_Identifier can never raise a user above 2 Active_Sessions or affect another user's Sessions.
- Clarification: session-revoked errors carry a reason (CONCURRENT_LIMIT or DEVICE_REPLACED). The Dashboard shows the "another device" message for CONCURRENT_LIMIT and a "signed in again on this device" message for DEVICE_REPLACED. Audit_Log entries for both kinds of revocation record the reason and Device_Identifier.

### Reason
- The Business Owner asked to limit concurrent logins per user to 2 devices and to sign out the oldest session automatically when another login occurs.

### Impact
- Requirements:
  - Glossary: added Active_Session, Concurrent_Session_Limit, Session_Revocation, Revoked_Session, Device_Identifier, Device, Device_Replacement.
  - Introduction: added a note on the 2-device limit.
  - Requirement 10: added acceptance criteria 10.14–10.29 (the last number moved from 10.26 to 10.29). Criteria 10.1–10.13 are unchanged. Final V4 numbering, with former numbers where they changed:
    - 10.14 limit (revised: 2 per user, 1 per Device_Identifier)
    - 10.15 login below the limit (revised, was 10.15)
    - 10.16 device replacement (was 10.21, revised)
    - 10.17 concurrent-limit revocation after device replacement (revised, was 10.16)
    - 10.18 tie-break (was 10.17)
    - 10.19 atomicity (was 10.18, extended to device replacement)
    - 10.20 invalid credentials leave Sessions unchanged (was 10.19)
    - 10.21 expired Sessions excluded from the count (was 10.20)
    - 10.22 missing Device_Identifier (new)
    - 10.23 Device_Identifier cannot exceed the limit or affect other users (new)
    - 10.24 revoked request rejected, error carries the reason (was 10.22)
    - 10.25 Dashboard redirect and message for CONCURRENT_LIMIT (was 10.23)
    - 10.26 Dashboard redirect and message for DEVICE_REPLACED (new)
    - 10.27 revoked Session blocks Dashboard views (was 10.24)
    - 10.28 POS queue retained on revocation (was 10.25)
    - 10.29 Audit_Log for both revocation kinds (was 10.26)
  - design.md and tasks.md both reference the final numbering 10.14–10.29.
- Design components (design.md now V4, aligned with criteria 10.14–10.29):
  - Request Lifecycle and RBAC: the sequence diagram loads the session row and adds a `401 SESSION_REVOKED` branch (with reason) after the expiry check (10.24).
  - Security / Access Module: added `MAX_CONCURRENT_SESSIONS = 2`, pure `applySessionLimit` (returns revocations with reason `DEVICE_REPLACED` or `CONCURRENT_LIMIT`), `isSessionActive`, `parseDeviceId`, `authenticate(credentials, deviceId?)`, `getSessionStatus` (returns `{ status: 'REVOKED', reason }`), and `subscribeSessionEvents` (payload carries session id and reason).
  - Subsection renamed "Concurrent Session Limit and Device Identity (V4, Req 10.14-10.29)":
    - Device identity: client-generated UUID v4 `deviceId` persisted per client installation (Dashboard localStorage with cookie fallback, POS installation storage), treated as a hint scoped to the authenticated user, never used for authorisation. Missing or invalid values get a server-generated id (10.22). The hard cap of 2 is evaluated on sessions (10.23). Known limitation: one browser profile is one Device, and a different browser or profile is a different Device.
    - Sessions table: partial unique index on `(userId, deviceId)` where `revokedAt IS NULL`. The login transaction closes any non-revoked row for the same device first: active rows as `DEVICE_REPLACED` (with event and audit), already-expired rows silently (no event, no audit).
    - Atomic login rewritten: credential check, device id resolution, per-user lock, `applySessionLimit` (device replacement first, then oldest-first `CONCURRENT_LIMIT`), audit entries with reason and deviceId, `SESSION_REVOKED` events with reason (10.16–10.20, 10.29).
    - Dashboard: reason-specific redirect and message (`signed_in_elsewhere` for CONCURRENT_LIMIT, `signed_in_again` for DEVICE_REPLACED), route guard, cross-tab broadcast, and a rule that revocation events for a stale session id are ignored so the tab that logged in again is not redirected (10.25–10.27). POS queue retention (10.28).
    - Edge cases updated: same-device re-login never evicts another device's session, ping-pong only across 3 or more Devices, cleared storage creates a new Device.
  - Data Models: `Session` gains `id`, `deviceId`, `deviceLabel`, `revokedAt`, `revokedReason` (`CONCURRENT_LIMIT` or `DEVICE_REPLACED`), `revokedBySessionId`. ER diagram adds `USER ||--o{ SESSION`.
  - Key Invariants: at most 2 active sessions per user and at most 1 per deviceId, new session active after login, same-device sessions always revoked as DEVICE_REPLACED, CONCURRENT_LIMIT revokes only the earliest other-device session when 2 exist, failed login leaves sessions unchanged, a deviceId never affects other users.
  - "Session Limit Testing Note" revised: property test P16 covers device replacement and the limit, plus `parseDeviceId` unit tests, same-device and cross-device concurrency tests, device id abuse tests, guard test with reason, Dashboard tests for both messages and stale session id handling, and POS queue test.
- Implementation tasks (tasks.md now V4):
  - Added: 2.20 (pure `applySessionLimit` with device replacement first and reasons, `isSessionActive`, `parseDeviceId`), 2.21* (Property 16), 2.22* (`parseDeviceId` unit tests), 4.4 (atomic login with `deviceId`, per-user lock, silent close of expired same-device rows, audit entry per revocation with reason and deviceId, `SESSION_REVOKED` guard with reason), 4.5 (`GET /session/status` and `/session/events` carrying the reason), 4.6* (server tests: same-device and cross-device concurrency, forged or missing deviceId, audited device replacement, silent expired-row close), 10.12 (Dashboard revocation handling with reason-specific redirect and message, stale-session-id rule), 10.13* (Dashboard tests), 10.14 (client `deviceId` generation, persistence, and sending in Dashboard and POS).
  - Changed: 1.2 (`sessions` table gains `deviceId`, reason enum, and partial unique index on `(userId, deviceId)`), 10.5 and 10.6* (POS queue retained on `SESSION_REVOKED` for both reasons), 11.2* (A, B, A device sequence keeps B; a third distinct device revokes the oldest and shows the elsewhere message). Property P16 row updated; Overview, Notes, and Task Dependency Graph (2.22 in wave 2, 10.14 in wave 6) are extended.
  - Existing task numbers are unchanged; 2.22 and 10.14 are new ids.
- Nothing in this version is implemented or verified yet.
