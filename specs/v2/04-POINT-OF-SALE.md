# 04 — Point of Sale (V2 changes)

> Delta file. Baseline: `../v1/04-POINT-OF-SALE.md`. Source: `cashier-business-owner` V2, Requirement 10.12 (design: POS client).

## Behaviour change

- **10.12** IF a Session expires while the POS_Interface holds Transactions that are not yet synchronised, THEN THE System SHALL retain those Transactions and synchronise them after the user re-authenticates.

## Design notes

- The POS applies the same expiry and redirect behaviour as the Dashboard.
- The local IndexedDB transaction queue is keyed by outlet and device, not by session. It is never cleared on expiry or logout.
- Sync requests that receive `401 SESSION_EXPIRED` leave items in the queue. Sync resumes after the next successful login.
- Idempotency via `transaction_uid` (Req 4.10 / 12.2) makes retried submissions safe.