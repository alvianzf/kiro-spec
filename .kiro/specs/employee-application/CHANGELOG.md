# Changelog

## V1 - Initial Release
- Initial requirements (Requirements 1–10), design, and tasks already present in the spec folder.
- No implementation has been verified.

## V2 - 2026-11-07
### Type
- Design change
- Non-breaking, presentation only. No behaviour, API, or data changes.

### Changes
- Added the "Visual Theme and Design Tokens (V2)" section to design.md: brand tokens, role assignment, contrast rules, status cues that never rely on colour alone, and receipt/payslip rendering.
- Added the `@app/theme` dependency to the design Assumptions.
- Added a theme reference to Cross-Cutting Concerns -> Usability.

### Reason
- The Business Owner asked for a brand theme (primary #990000, secondary #000000, accent #FFFFFF) applied to all relevant designs and tasks.

### Impact
- Requirements: none changed.
- Design components: Assumptions, Cross-Cutting Concerns (Usability), new Visual Theme and Design Tokens section.
- Implementation tasks: added 1.1, 1.2, 1.3 (optional), 13.1, 13.2, 13.3 (optional), 14.1, 14.2, 15.1, 15.2; extended the text of 4.2, 5.1, 7.2, 7.5. Existing numbering unchanged.

## V3 - 2026-11-07
### Type
- Design change
- Behavioural for session handling. No API or data model changes in this app; the client reacts to session revocation from the shared service.

### Changes
- Added a V3 Assumption that the shared identity service enforces the concurrent session limit (max 2 Active_Sessions per user, oldest revoked) as defined in the cashier-business-owner spec (V4, requirements 10.14-10.26 and "Concurrent Session Limit (V4)"). The employee app consumes it and does not count or revoke sessions.
- Added a V3 Assumption that clock and POS submissions carry client-generated idempotency keys (not previously stated).
- Added the "Session Revocation Handling (V3)" subsection under Authentication & Session: SSE subscription, 3s polling fallback while visible, visibilitychange/focus/online checks, global 401 interceptor for SESSION_REVOKED, redirect to login with reason=signed_in_elsewhere within 5s, route guard, cross-tab broadcast, and retention and idempotent replay of queued POS and attendance actions.
- Extended Error Handling with the SESSION_REVOKED case.
- Extended Testing Strategy with revocation redirect tests (fake timers, push and polling), interceptor, route guard, cross-tab, and queued-action retention tests.
- Clarification within the same revision (still V3, no new version): aligned the design with the clarified cashier-business-owner V4 session policy (requirements 10.14-10.29). The Assumption now references 10.14-10.29 and the "Concurrent Session Limit and Device Identity (V4, Req 10.14-10.29)" section, and describes the added rule of at most 1 Active_Session per Device_Identifier.
- Clarification: Session Revocation Handling now covers two reasons. `CONCURRENT_LIMIT` redirects to `/login?reason=signed_in_elsewhere` ("Your account was signed in on another device."). `DEVICE_REPLACED` redirects to `/login?reason=signed_in_again` ("You were signed out because you signed in again on this device."). Unknown or missing reasons fall back to the `CONCURRENT_LIMIT` variant.
- Clarification: documented the client-generated, persisted UUID v4 `deviceId` (localStorage with cookie fallback, or installation storage), sent on login and kept across revocation. The employee client makes no session decisions with it.
- Clarification: documented the payload shapes consumed: `401 { code: 'SESSION_REVOKED', reason }`, `GET /session/status` returning `{ status: 'REVOKED', reason }`, and SSE events carrying `sessionId` and `reason`.
- Clarification: added the own-login (stale sessionId) rule. The tab that performs a new login ignores revocation events, status responses, broadcasts, and 401s that refer to its earlier session.
- Clarification: Error Handling and Testing Strategy updated. Tests now cover both reasons across push, polling, and interceptor paths, plus the stale-session rule and the device id lifecycle.

### Reason
- The Business Owner asked to limit each user to 2 concurrent logins and sign out the oldest session.

### Impact
- Requirements: none changed, because authentication is owned by the shared identity service.
- Design components: Assumptions (reference to cashier 10.14-10.29, per-device rule), Authentication & Session (new subsection with device identifier, payload shapes, own-login rule, and two reason variants), Error Handling (both reasons, stale-session rule), Testing Strategy (both reasons, stale-session, device id tests). The message uses the shared status component from the Visual Theme section.
- Implementation tasks: added 16.1-16.6 (16.5 optional) under new top-level task 16 "Session revocation handling": idempotency keys (16.1), detector (16.2), revocation flow with both reason variants, own-login rule, and cross-tab broadcast (16.3), queue retention and replay (16.4), tests (16.5), and deviceId generation/persistence (16.6). Extended the text of 2.1 (send the deviceId at login, store the new session id, register the detector), 7.6 (queue retention on revocation), and 13.2 (status component for both revocation messages). Existing numbering unchanged. Wording fix (no version change): task 2.1 now only exposes the hooks (optional `deviceId` login argument and post-login hook); 16.6 and 16.2 wire into them, which matches the dependency graph. Dependency graph: 16.1 and 16.6 in wave 8, 16.2 in wave 9, 16.3 and 16.4 in wave 10, 16.5 in wave 11; 16.6 precedes 16.2 and 16.3, 16.4 depends on 16.1 and 7.6, and 16.3 depends on 2.1, 13.2, and 16.2.
