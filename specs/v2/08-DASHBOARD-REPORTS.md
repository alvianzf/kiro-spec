# 08 — Dashboard & Reports (V2 changes)

> Delta file. Baseline: `../v1/08-DASHBOARD-REPORTS.md`. Source: `cashier-business-owner` V2, Requirements 10.9 and 10.10 (design: Dashboard client).

## Behaviour change

- **10.9** WHILE a user is viewing the Dashboard, WHEN the user's Session expiry time is reached, THE Dashboard SHALL clear the Session and redirect the user to the Login_Page without waiting for further user input.
- **10.10** WHILE a user has no valid Session, WHEN the user requests any Dashboard view, THE System SHALL redirect the user to the Login_Page.

## Design notes

- The login response returns `expiresAt` and `serverNow`. The client stores `clockOffset = serverNow - Date.now()` so a wrong device clock cannot change the session length.
- A `useSessionExpiry` hook schedules a timer for the expiry and also re-checks on `visibilitychange`, `focus`, `online`, and every 30 seconds.
- A global HTTP interceptor treats `401 SESSION_EXPIRED` the same way. A route guard on every Dashboard route checks expiry before render.
- Session state is broadcast across tabs (`BroadcastChannel` / `storage`), and the client returns to `returnTo` after re-authentication when it is a same-origin Dashboard path.