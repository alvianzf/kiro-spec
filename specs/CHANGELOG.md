# Changelog

Version numbers match `.kiro/specs/cashier-business-owner/CHANGELOG.md`. Each version folder holds only the domain files changed in that version, except `v1`, which is the full baseline. Earlier version folders are never edited or removed.

Scope: this folder translates the `cashier-business-owner` spec only. The `employee-application` spec (own versions V1–V3, tracked in `.kiro/specs/employee-application/CHANGELOG.md`) is not translated here yet.

## V1 - Initial Release
- Full baseline, 12 domain files (`00-OVERVIEW` to `11-NONFUNCTIONAL`), covering Requirements 1–15.
- Moved from the flat `specs/` layout into `v1/` without content changes.

## V2 - 2026-11-07
### Changes
- Sessions expire at the next 00:00 (Day_Boundary) in a configurable Business_Timezone; `401 SESSION_EXPIRED`; Dashboard redirects to the Login_Page on expiry; POS keeps unsynchronised transactions.
### Changed domain files
- `00-OVERVIEW`, `04-POINT-OF-SALE`, `08-DASHBOARD-REPORTS`, `10-SECURITY-ACCESS`
### Requirements
- Requirement 10: criteria 10.7–10.13 added.

## V3 - 2026-11-07
### Changes
- Brand theme (primary `#990000`, secondary `#000000`, accent `#FFFFFF`), shared `@app/theme` package, contrast and status-cue rules. Presentation only; no requirement changes.
### Changed domain files
- `00-OVERVIEW`, `03-SALES-TRANSACTIONS`, `04-POINT-OF-SALE`, `08-DASHBOARD-REPORTS`, `09-NOTIFICATIONS`
### Requirements
- None changed.

## V4 - 2026-11-07
### Changes
- At most 2 Active_Sessions per user and 1 per Device_Identifier. A repeat login on the same device replaces that device's session (`DEVICE_REPLACED`). Otherwise the oldest other-device session is revoked (`CONCURRENT_LIMIT`). Revoked sessions get `401 SESSION_REVOKED` with a reason, and the Dashboard redirects within 5 seconds with a reason-specific message.
- Breaking change for multi-device behaviour.
### Changed domain files
- `00-OVERVIEW`, `04-POINT-OF-SALE`, `08-DASHBOARD-REPORTS`, `10-SECURITY-ACCESS`
### Requirements
- Requirement 10: criteria 10.14–10.29 added; glossary gains Active_Session, Concurrent_Session_Limit, Session_Revocation, Revoked_Session, Device_Identifier, Device, Device_Replacement.