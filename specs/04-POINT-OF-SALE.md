# 04 — Point of Sale + Usability

**Requirement 4** (Point of Sale) and **Requirement 15** (Usability). Client-side POS logic, much of it implemented as pure functions for property testing.

> **User Story (Req 4):** As a Cashier, I want a simple interface to process customer transactions, so that I can serve customers accurately and quickly.
>
> **User Story (Req 15):** As a Cashier, I want the POS interface to be easy to operate, so that I can process transactions with minimal training.

## Acceptance Criteria — Requirement 4

1. WHEN a Cashier searches by product name, SKU, or barcode, THE POS_Interface SHALL display the matching Products available at the Cashier's assigned Outlet.
2. WHEN a Cashier adds, removes, or modifies a Product in the shopping cart, THE POS_Interface SHALL update the cart contents accordingly.
3. WHEN the cart contents change, THE POS_Interface SHALL recalculate the subtotal, applicable discounts, applicable taxes, and total.
4. WHEN a Cashier enters a tendered cash amount for a cash payment, THE POS_Interface SHALL calculate the change due as the tendered amount minus the total.
5. WHERE a Cashier selects split payment, THE POS_Interface SHALL allow the total to be divided across multiple payment methods and SHALL complete the Transaction only when the sum of entered payments equals the total.
6. THE POS_Interface SHALL support cash, bank transfer, debit card, credit card, and QRIS payment methods.
7. WHEN a Cashier begins a Shift, THE POS_Interface SHALL record the opening cash balance entered by the Cashier.
8. WHEN a Cashier ends a Shift, THE POS_Interface SHALL record the closing cash balance entered by the Cashier.
9. WHEN a Cashier completes a Transaction, THE POS_Interface SHALL generate a receipt for that Transaction.
10. IF a Transaction submission carries an identifier that matches an already-recorded Transaction, THEN THE POS_Interface SHALL reject the duplicate submission and return the original Transaction record.

## Acceptance Criteria — Requirement 15

1. THE POS_Interface SHALL present the search, cart, payment, and receipt actions required to complete a Transaction on a single workflow without navigation to the Dashboard.
2. WHEN a Cashier enters an invalid value during a Transaction, THE POS_Interface SHALL display a descriptive message identifying the invalid value.

## Module Interface (client-side)

```typescript
interface PosController {
  searchProducts(query: string, outletId: ID): Promise<Product[]>; // 4.1
  updateCart(cart: Cart, op: CartOp): Cart;                         // 4.2 (pure)
  recalculateCart(cart: Cart): CartTotals;                          // 4.3 (pure)
  calculateChange(total: Money, tendered: Money): Money;            // 4.4 (pure)
  validateSplitPayment(total: Money, payments: Payment[]): boolean; // 4.5 (pure)
  openShift(outletId: ID, openingBalance: Money): Promise<Shift>;   // 4.7
  closeShift(shiftId: ID, closingBalance: Money): Promise<ShiftSummary>; // 4.8
  submitTransaction(input: TransactionInput): Promise<Transaction>; // delegates to SalesService
}
```

### Pure Functions (primary property-test targets)

- `updateCart(cart, op)` → `Cart` (4.2) — add/remove/modify cart entries.
- `recalculateCart(cart)` → `CartTotals` (4.3) — subtotal, discounts, taxes, total.
- `calculateChange(total, tendered)` → `Money` (4.4) — `tendered - total`.
- `validateSplitPayment(total, payments)` → `boolean` (4.5) — true only when `sum(payments) == total`.

`submitTransaction` delegates to `SalesService.recordTransaction`, which enforces idempotency and atomicity (see `03-SALES-TRANSACTIONS.md`).

## Data Model Entity

Monetary fields are integers in minor currency units.

```typescript
interface Shift {
  id: ID; cashierId: ID; outletId: ID;
  openingBalance: Money; closingBalance?: Money;
  expectedBalance?: Money;  // opening + cash sales - cash refunds (for 9.4)
  openedAt: Timestamp; closedAt?: Timestamp;
}
```

## Payment Methods (4.6)

The POS supports: **cash**, **bank transfer**, **debit card**, **credit card**, and **QRIS** (the Indonesian QR-code standard). Represented as `Payment.method: 'CASH' | 'BANK_TRANSFER' | 'DEBIT' | 'CREDIT' | 'QRIS'`.

## Key Rules

- **Split payment (4.5):** The total may be divided across multiple payment methods. The transaction completes only when the sum of entered payments equals the total — enforced by the pure `validateSplitPayment`.
- **Duplicate prevention (4.10):** A submission carrying an already-recorded identifier is rejected and the original record is returned (idempotency via `transaction_uid`). See `03-SALES-TRANSACTIONS.md`.
- **Single-workflow usability (15.1):** Search, cart, payment, and receipt are presented on one workflow, with no navigation to the Dashboard required to complete a sale.
- **Descriptive validation (15.2):** Invalid values during a transaction produce a descriptive message identifying the offending value.

## Cross-References

- Shift closing balance feeds cash-discrepancy detection (9.4) via `expectedBalance = openingBalance + cashSales - cashRefunds` — see `09-NOTIFICATIONS.md`.
- Receipt generation (4.9) is backed by `SalesService.generateReceipt` (3.8) — see `03-SALES-TRANSACTIONS.md`.
- Fast confirmation target (< 2s, Req 11.3) — see `11-NONFUNCTIONAL.md`.
