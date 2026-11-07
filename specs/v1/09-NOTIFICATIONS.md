# 09 — Notifications

**Requirement 9** (Notifications) plus **Requirement 14.4** (backup-failure notification). Event-driven alerts dispatched to the Business Owner and relevant Outlet Managers.

> **User Story:** As a Business Owner, I want to be notified of important business events, so that I can respond to issues promptly.

## Acceptance Criteria — Requirement 9

1. IF the available quantity of a Stock_Item is at or below its Low_Stock_Threshold, THEN THE System SHALL send a low-stock notification to the Business_Owner and the relevant Outlet_Manager.
2. IF the available quantity of a Stock_Item reaches zero, THEN THE System SHALL send an out-of-stock notification to the Business_Owner and the relevant Outlet_Manager.
3. WHEN a Refund or transaction correction exceeds a configured threshold amount, THE System SHALL send an unusual-activity notification to the Business_Owner.
4. IF a Cashier's recorded closing cash balance differs from the System-calculated expected balance for the Shift, THEN THE System SHALL send a cash-register-discrepancy notification to the Business_Owner and the relevant Outlet_Manager.
5. IF an employee has a recorded absence or late arrival, THEN THE System SHALL send an attendance-issue notification to the relevant Outlet_Manager.
6. WHEN a Payroll_Record or supplier payment has an outstanding balance on or after its due date, THE System SHALL send a pending-payment notification to the Business_Owner.

## Related Criterion — Requirement 14.4 (Backup Failure)

4. IF an automated backup fails to complete, THEN THE System SHALL send a backup-failure notification to the Business_Owner.

## Module Interface

```typescript
interface NotificationService {
  notifyLowStock(stockItem, recipients): Promise<void>;      // 9.1
  notifyOutOfStock(stockItem, recipients): Promise<void>;    // 9.2
  notifyUnusualActivity(refundOrCorrection): Promise<void>;  // 9.3
  notifyCashDiscrepancy(shift): Promise<void>;               // 9.4
  notifyAttendanceIssue(entry): Promise<void>;               // 9.5
  notifyPendingPayment(obligation): Promise<void>;           // 9.6
  notifyBackupFailure(job): Promise<void>;                   // 14.4
}
```

Recipient resolution is a pure function `resolveRecipients(event, outlet)` → `{ businessOwner, outletManager? }`.

## Event Sources

| Notification | Trigger | Source Domain |
| --- | --- | --- |
| Low-stock (9.1) | Quantity at/below threshold (`evaluateStockAlerts`) | `01-INVENTORY.md` |
| Out-of-stock (9.2) | Quantity reaches zero (`evaluateStockAlerts`) | `01-INVENTORY.md` |
| Unusual activity (9.3) | Refund/correction exceeds configured threshold | `03-SALES-TRANSACTIONS.md` |
| Cash discrepancy (9.4) | Closing balance differs from `expectedBalance` | `04-POINT-OF-SALE.md` |
| Attendance issue (9.5) | Recorded absence or late arrival | `05-EMPLOYEES.md` |
| Pending payment (9.6) | Outstanding payroll/liability at/after due date | `06-PAYROLL.md`, `07-ACCOUNTING.md` |
| Backup failure (14.4) | Scheduled backup job fails | `11-NONFUNCTIONAL.md` |

## Notes

- **`resolveRecipients(event, outlet)` (pure):** Resolves the `{ businessOwner, outletManager? }` recipient set for an event. Being side-effect-free, it is a property-test target. The relevant Outlet Manager is the one assigned to the event's outlet.
- **Cash discrepancy (9.4):** For a cash-only shift, the expected balance is `openingBalance + cashSales - cashRefunds` (see the Shift entity in `04-POINT-OF-SALE.md`). A mismatch against the recorded closing balance raises the notification.
