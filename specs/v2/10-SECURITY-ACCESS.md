# 10 — User Access & Security (V2 changes)

> Delta file. Baseline: `../v1/10-SECURITY-ACCESS.md` (criteria 10.1–10.6, unchanged). Source: `cashier-business-owner` V2.

## New acceptance criteria

7. WHEN the System establishes a Session, THE System SHALL set the Session expiry to the next Day_Boundary after the Session issue time.
8. IF a request carries a Session whose expiry time has been reached, THEN THE System SHALL reject the request with a session-expired authentication error and SHALL NOT perform the requested operation.
9. WHILE a user is viewing the Dashboard, WHEN the user's Session expiry time is reached, THE Dashboard SHALL clear the Session and redirect the user to the Login_Page without waiting for further user input.
10. WHILE a user has no valid Session, WHEN the user requests any Dashboard view, THE System SHALL redirect the user to the Login_Page.
11. WHEN a user re-authenticates after Session expiry, THE System SHALL establish a new Session with an expiry set to the next Day_Boundary.
12. IF a Session expires while the POS_Interface holds Transactions that are not yet synchronised, THEN THE System SHALL retain those Transactions and synchronise them after the user re-authenticates.
13. WHEN the Business_Owner configures the Business_Timezone, THE System SHALL use the configured Business_Timezone to determine the Day_Boundary for Sessions established after the change.

## Module interface additions

```typescript
interface AccessService {
  setBusinessTimezone(tz: IanaTimezone): Promise<void>;        // 10.13
}

// Returns the next 00:00 in `tz` strictly after `issuedAt`, as a UTC instant.
function computeSessionExpiry(issuedAt: Timestamp, tz: IanaTimezone): Timestamp; // 10.7, 10.11
function isSessionExpired(session: Session, now: Timestamp): boolean;            // 10.8, 10.9, 10.10
```

## Design notes

- `authenticate` sets `issuedAt = now` and `expiresAt = computeSessionExpiry(now, businessTimezone)`. The timezone is read at login time, so a change only affects new sessions.
- `computeSessionExpiry` uses an IANA timezone database, not fixed UTC offsets.
- The guard checks `isSessionExpired(session, serverNow)` before any permission check and returns `401 { code: "SESSION_EXPIRED" }`.
- There is no sliding renewal or refresh token that extends a session past the Day_Boundary.
- Client behaviour is described in `08-DASHBOARD-REPORTS.md` (Dashboard) and `04-POINT-OF-SALE.md` (POS) of this version folder.

## Correctness property

- **P12** Session expiry is a local 00:00 in the Business_Timezone, after the issue time, with no local midnight in between (10.7, 10.11, 10.13).
- **P13** `isSessionExpired` is true iff `now >= expiresAt` (10.8, 10.9, 10.10).