# 03 — Sales & Transaction Oversight + Reliability / Data Integrity

**Requirement 3** (Sales and Transaction Oversight) and **Requirement 12** (Reliability and Data Integrity). This is the integrity-critical path of the system: idempotent, atomic transaction recording.

> **User Story (Req 3):** As a Business Owner, I want to view and manage transactions across all outlets, so that I can monitor sales activity and authorise corrections.
>
> **User Story (Req 12):** As a Business Owner, I want transactions and financial records stored accurately, so that my records are trustworthy.

## Acceptance Criteria — Requirement 3

1. WHEN the Business_Owner requests the transaction list, THE System SHALL display Transactions from all Outlets.
2. WHEN a Transaction is synchronised from an Outlet, THE System SHALL make the Transaction visible in the transaction list within 60 seconds of synchronisation.
3. WHEN a user with transaction access selects a Transaction, THE System SHALL display the products, quantities, prices, discounts, payment methods, and Cashier associated with that Transaction.
4. WHEN a user with transaction access applies a filter by Outlet, date, product, or employee, THE System SHALL display only the Transactions that match the selected filter.
5. WHEN a Refund, cancellation, or correction is requested, THE System SHALL require authorisation from a Business_Owner or Outlet_Manager before applying the change.
6. WHEN an authorised Refund is completed for a Transaction linked to a Stock_Item, THE System SHALL restore the refunded quantity to the linked Stock_Item.
7. WHEN a user with transaction access requests a sales summary, THE System SHALL display total sales per Outlet and total sales across all Outlets for the selected period.
8. WHEN a user requests a receipt for a completed Transaction, THE System SHALL generate a receipt that is available to print or share.

## Acceptance Criteria — Requirement 12

1. WHEN a Transaction is recorded, THE System SHALL persist the Transaction so that it remains available after the persistence operation completes.
2. IF two Transaction submissions carry the same Transaction identifier, THEN THE System SHALL persist only one Transaction record for that identifier.
3. IF a Transaction submission fails to persist completely, THEN THE System SHALL discard the partial Transaction and return an error without altering stock quantities or financial records.

## Module Interface

```typescript
interface SalesService {
  recordTransaction(input: TransactionInput): Promise<Transaction>; // idempotent by uid 4.10/12.2, atomic 12.3
  listTransactions(scope: OutletScope, filter: TransactionFilter): Promise<Transaction[]>; // 3.1, 3.4
  getTransaction(id: ID): Promise<TransactionDetail>;    // 3.3
  requestAdjustment(input: AdjustmentRequest): Promise<AdjustmentTicket>; // 3.5 requires authorisation
  applyRefund(input: RefundInput): Promise<Refund>;      // 3.6 restores stock, requires auth
  getSalesSummary(filter: PeriodFilter): Promise<SalesSummary>; // 3.7
  generateReceipt(transactionId: ID): Promise<Receipt>;  // 3.8, 4.9
}
```

`recordTransaction` is the integrity-critical path. It runs in a single database transaction that: (a) inserts the transaction keyed on `transaction_uid` with a unique constraint; (b) deducts linked stock with row locks; (c) records income into accounting; (d) commits. A unique-constraint violation on `transaction_uid` is caught and resolved by returning the existing record rather than erroring.

## Transaction Sync and POS Flow

```mermaid
sequenceDiagram
    participant Cashier
    participant POS as POS Client
    participant Local as Local Queue
    participant API
    participant DB

    Cashier->>POS: Complete sale
    POS->>POS: Compute totals/change (pure core)
    POS->>Local: Persist with transaction_uid
    POS-->>Cashier: Receipt (< 2s, Req 11.3)
    loop Sync (target < 60s, Req 3.2)
        Local->>API: POST transaction (transaction_uid)
        alt uid already recorded
            API-->>Local: Return original record (Req 4.10 / 12.2)
        else new
            API->>DB: BEGIN; insert tx; deduct stock; record income; COMMIT (Req 12.3)
            API-->>Local: Confirmed record
        end
        Local->>Local: Mark synced
    end
```

## Idempotency and Atomicity

- **Idempotency (4.10, 12.2):** The client supplies a `transaction_uid`. The database enforces a unique constraint on it. If a duplicate submission arrives (e.g., retry after a flaky connection), the server catches the unique-constraint violation and returns the original record instead of creating a second transaction.
- **Atomic record (12.1, 12.3):** Recording runs as a single ACID transaction:

  ```sql
  BEGIN;
    INSERT transaction (keyed on transaction_uid);
    deduct linked stock (row locks);
    record income into accounting;
  COMMIT;
  ```

  If any step fails, the whole transaction rolls back — no partial transaction, no altered stock quantities, no altered financial records (12.3).

## Data Model Entities

Monetary fields are integers in minor currency units. Quantities are integers.

```typescript
interface Transaction {
  id: ID; transactionUid: string;  // client-supplied, UNIQUE (idempotency 4.10/12.2)
  outletId: ID; cashierId: ID; shiftId: ID;
  lines: TransactionLine[]; payments: Payment[];
  subtotal: Money; discountTotal: Money; taxTotal: Money; total: Money;
  status: 'COMPLETED' | 'REFUNDED' | 'PARTIALLY_REFUNDED';
  createdAt: Timestamp;
}

interface TransactionLine { productId: ID; variationId?: ID; quantity: Quantity; unitPrice: Money; lineDiscount: Money; lineTotal: Money; }
interface Payment { method: 'CASH' | 'BANK_TRANSFER' | 'DEBIT' | 'CREDIT' | 'QRIS'; amount: Money; tendered?: Money; change?: Money; }

interface Refund {
  id: ID; transactionId: ID; lines: RefundLine[];
  amount: Money; authorisedBy: ID; createdAt: Timestamp;
}
interface RefundLine { productId: ID; quantity: Quantity; amount: Money; }
```

## Invariants

- `Transaction.total == subtotal - discountTotal + taxTotal`.
- `sum(line.lineTotal) == subtotal`.
- `Transaction.transactionUid` is globally unique (database unique constraint).

## Cross-References

- Refund stock restoration (3.6) writes a `REFUND` `StockMovement` — see `01-INVENTORY.md`.
- Transaction income recording (7.3) is called within `recordTransaction` — see `07-ACCOUNTING.md`.
- POS-side cart/change/split logic — see `04-POINT-OF-SALE.md`.
- Unusual-activity notifications for large refunds/corrections (9.3) — see `09-NOTIFICATIONS.md`.
