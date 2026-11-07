# 04 — Point of Sale (V4 changes)

> Delta file. Baseline: `../v1/04-POINT-OF-SALE.md` plus V2 and V3 deltas. Source: `cashier-business-owner` V4, Requirements 10.22 and 10.28.

- **10.28** IF a Session is revoked while the POS_Interface holds Transactions that are not yet synchronised, THEN THE System SHALL retain those Transactions and synchronise them after the user re-authenticates.
- The IndexedDB queue is keyed by outlet and device, never by session, and is not cleared on revocation. Sync that receives `401 SESSION_REVOKED` (either reason) leaves items queued and resumes after re-login. `transaction_uid` idempotency makes retries safe.
- The POS app generates a UUID v4 `deviceId` once, persists it in installation storage, and sends it with every login.