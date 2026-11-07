# 00 — Overview (V2 changes)

> Delta file. Baseline: `../v1/00-OVERVIEW.md`. Source spec: `cashier-business-owner` V2.

## Glossary additions

- **Session**: The authenticated state established for a user after successful login, carrying an issue time and an expiry time.
- **Business_Timezone**: The single IANA timezone configured by the Business_Owner (for example, `Asia/Jakarta`) that defines when a calendar day starts and ends for session expiry.
- **Day_Boundary**: 00:00 (midnight) at the start of a calendar day in the Business_Timezone.
- **Login_Page**: The System page on which a user submits credentials to establish a Session.

## Request lifecycle change

The central authorization guard now checks session expiry before resolving permissions. An expired session returns `401 SESSION_EXPIRED`, distinct from `401 INVALID_CREDENTIALS` and `403 FORBIDDEN`. See `10-SECURITY-ACCESS.md` in this version folder.

## Data model additions

- `Session.expiresAt` is derived as the next Day_Boundary after `issuedAt` in the Business_Timezone.
- New single-row `BusinessSettings { businessTimezone: IanaTimezone (default 'Asia/Jakarta'); updatedAt; updatedBy }`.
- New invariant: for every Session, `issuedAt < expiresAt`, `expiresAt` is a local 00:00 in the Business_Timezone, and no local 00:00 lies strictly between them.

## Affected domain files in this version

`00-OVERVIEW.md`, `04-POINT-OF-SALE.md`, `08-DASHBOARD-REPORTS.md`, `10-SECURITY-ACCESS.md`.