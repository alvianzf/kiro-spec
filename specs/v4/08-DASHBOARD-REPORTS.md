# 08 — Dashboard & Reports (V4 changes)

> Delta file. Baseline: `../v1/08-DASHBOARD-REPORTS.md` plus V2 and V3 deltas. Source: `cashier-business-owner` V4, Requirements 10.25, 10.26, 10.27.

- **10.25** WHILE a user is viewing the Dashboard, WHEN the user's Session is revoked with reason `CONCURRENT_LIMIT`, THE Dashboard SHALL clear the Session, redirect the user to the Login_Page within 5 seconds under normal operating load without waiting for further user input, and display a message that the account was signed in on another device.
- **10.26** WHILE a user is viewing the Dashboard, WHEN the user's Session is revoked with reason `DEVICE_REPLACED`, THE Dashboard SHALL clear the Session, redirect the user to the Login_Page within 5 seconds under normal operating load without waiting for further user input, and display a message that the user signed in again on this device.
- **10.27** WHILE a user has a Revoked_Session, WHEN the user requests any Dashboard view, THE System SHALL redirect the user to the Login_Page.

## Design notes

- Push: Server-Sent Events at `/session/events`. Polling fallback: `GET /session/status` every 3 seconds while the tab is visible. Extra checks on `visibilitychange`, `focus`, and `online`. The 401 interceptor maps `SESSION_REVOKED` like `SESSION_EXPIRED`, keeping the reason.
- Redirects: `CONCURRENT_LIMIT` goes to `/login?reason=signed_in_elsewhere` ("Your account was signed in on another device."). `DEVICE_REPLACED` goes to `/login?reason=signed_in_again` ("You were signed out because you signed in again on this device."). Messages use the shared status component (icon + text).
- Own-login rule: the tab that performs a new login ignores revocation events or status for a session id that is not the currently stored one. Other tabs adopt the new session id through the cross-tab broadcast.
- The Dashboard generates a UUID v4 `deviceId` once (`localStorage`, cookie fallback) and sends it with every login.