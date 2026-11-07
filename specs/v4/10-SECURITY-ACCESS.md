# 10 — User Access & Security (V4 changes)

> Delta file. Baseline: `../v1/10-SECURITY-ACCESS.md` (10.1–10.6) and `../v2/10-SECURITY-ACCESS.md` (10.7–10.13), both unchanged. Source: `cashier-business-owner` V4.

## New acceptance criteria

14. THE System SHALL allow at most 2 Active_Sessions per user and at most 1 Active_Session per Device_Identifier per user at any time, regardless of the user's Role, counted across all devices and clients, including the Dashboard and the POS_Interface.
15. WHEN a user authenticates successfully with a Device_Identifier for which the user holds no Active_Session, and the user holds fewer than 2 Active_Sessions, THE System SHALL establish the new Session without revoking any Active_Session.
16. WHEN a user authenticates successfully with a Device_Identifier for which the user already holds an Active_Session, THE System SHALL revoke that Active_Session as a Device_Replacement and establish the new Session, without counting the replaced Session toward the Concurrent_Session_Limit.
17. WHEN a user authenticates successfully and, after any Device_Replacement, the user still holds 2 Active_Sessions on other Devices, THE System SHALL revoke the Active_Session with the earliest issue time and establish the new Session, so that the user holds exactly 2 Active_Sessions after login completes.
18. IF the 2 Active_Sessions of a user have the same issue time when Session_Revocation under criterion 17 is required, THEN THE System SHALL revoke the Active_Session with the lowest Session identifier.
19. WHEN Session_Revocation or Device_Replacement and establishment of the new Session occur, THE System SHALL perform them as a single atomic operation, so that concurrent logins by the same user never leave the user with more than 2 Active_Sessions or more than 1 Active_Session per Device_Identifier.
20. IF a user submits invalid credentials, THEN THE System SHALL leave all Active_Sessions of that user unchanged.
21. THE System SHALL exclude Sessions that have reached their expiry time under criterion 7 from the count of Active_Sessions.
22. IF a login does not carry a valid Device_Identifier, THEN THE System SHALL treat the login as coming from a new Device with a System-generated Device_Identifier, so that the Concurrent_Session_Limit still holds.
23. THE System SHALL ensure that a submitted Device_Identifier cannot cause a user to hold more than 2 Active_Sessions and cannot affect the Sessions of any other user.
24. IF a request carries a Revoked_Session, THEN THE System SHALL reject the request with a session-revoked authentication error that carries the revocation reason (`CONCURRENT_LIMIT` or `DEVICE_REPLACED`), distinct from the session-expired error in criterion 8 and from the invalid-credentials error in criterion 2, and SHALL NOT perform the requested operation.
25–27. Dashboard redirect, messages, and view blocking: see `08-DASHBOARD-REPORTS.md` in this version folder.
28. POS transaction retention on revocation: see `04-POINT-OF-SALE.md` in this version folder.
29. WHEN the System revokes a Session because of the Concurrent_Session_Limit or a Device_Replacement, THE System SHALL record the revocation in the Audit_Log with the reason, the user, the revoked Session identifier, the new Session identifier, the Device_Identifier, and the timestamp.

## Module interface changes

```typescript
interface AccessService {
  authenticate(credentials, deviceId?: string): Promise<Session>;  // 10.1, 10.2, 10.15–10.23
  getSessionStatus(sessionId: ID): Promise<SessionStatusResult>;   // GET /session/status
  subscribeSessionEvents(sessionId: ID): EventStream<SessionEvent>; // /session/events (SSE)
}

type RevocationReason = 'CONCURRENT_LIMIT' | 'DEVICE_REPLACED';
type SessionStatusResult =
  | { status: 'ACTIVE' } | { status: 'EXPIRED' }
  | { status: 'REVOKED'; reason: RevocationReason };
type SessionEvent = { type: 'SESSION_REVOKED'; sessionId: ID; reason: RevocationReason; revokedAt: Timestamp };

const MAX_CONCURRENT_SESSIONS = 2; // fixed, all roles
function applySessionLimit(activeSessions: Session[], newSession: Session, now: Timestamp, limit = MAX_CONCURRENT_SESSIONS):
  { kept: Session[]; revoked: Array<{ session: Session; reason: RevocationReason }> };
function isSessionActive(session: Session, now: Timestamp): boolean;  // revokedAt null AND now < expiresAt
function parseDeviceId(raw: unknown): string | undefined;             // UUID v4 or undefined
```

## Design notes

- **Pure decision (`applySessionLimit`):** drop expired/revoked sessions; revoke every active session with the new session's `deviceId` as `DEVICE_REPLACED`; then sort the other sessions by `(issuedAt asc, id asc)` and revoke from the front as `CONCURRENT_LIMIT` until `remaining + 1 <= 2`.
- **Atomic login:** verify credentials first (`401 INVALID_CREDENTIALS` touches no session); resolve `deviceId = parseDeviceId(input) ?? randomUUID()`; open one transaction under a per-user lock (`SELECT ... FOR UPDATE` or `pg_advisory_xact_lock`); load the user's non-revoked rows; apply the limit; close rows (expired same-device rows are closed silently with no event or audit entry, so the unique index never blocks a login); insert the new session; append one Audit_Log entry per revocation; commit; then publish `SESSION_REVOKED` events with the reason.
- **Sessions table:** partial index on `(userId, issuedAt, id) WHERE revokedAt IS NULL` and partial unique index on `(userId, deviceId) WHERE revokedAt IS NULL`. The token carries only the session id; the guard validates it against the table on every request.
- **Guard:** after the expiry check, a revoked session returns `401 { code: "SESSION_REVOKED", reason }`.
- **Device identity:** a client-generated UUID v4 hint, always scoped to the authenticated user and never used for authorisation. Clearing storage only creates a new Device, and the cap of 2 Active_Sessions still holds because it is evaluated on sessions. Known limitation: identity is per client installation, not per physical machine. A different browser or profile is a different Device.
- **Edge cases:** login on A, B, then A again replaces A's first session and B stays. Ping-pong is only possible across 3 or more distinct Devices. Revocation under the limit is based on `issuedAt`, not last activity.

## Correctness property

- **P16** After a new login at most 2 sessions are active and at most 1 per `deviceId`; all active same-device sessions are revoked as `DEVICE_REPLACED`; `CONCURRENT_LIMIT` revokes exactly the earliest remaining other-device sessions by `(issuedAt, id)` and only when needed; expired and revoked sessions are ignored; results are deterministic (10.14–10.18, 10.21).