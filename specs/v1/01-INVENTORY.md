# 01 — Stock & Inventory Management

**Requirement 1** — covers all 11 acceptance criteria for stock items, movements, transfers, thresholds, alerts, history, and valuation.

> **User Story:** As a Business Owner, I want to manage stock items across all outlets, so that I can maintain accurate inventory and avoid shortages.

## Acceptance Criteria

1. WHEN the Business_Owner submits a valid new stock item, THE System SHALL create the Stock_Item and associate it with the selected Outlet.
2. WHEN the Business_Owner submits an edit to an existing Stock_Item, THE System SHALL update the Stock_Item with the submitted values.
3. WHEN the Business_Owner requests deletion of a Stock_Item, THE System SHALL remove the Stock_Item from the selected Outlet.
4. WHEN a user with inventory access requests stock quantities for an Outlet, THE System SHALL display the available quantity for each Stock_Item at that Outlet.
5. WHEN a stock movement, adjustment, or transfer is recorded, THE System SHALL update the affected Stock_Item quantities and record the change in the stock movement history.
6. WHEN the Business_Owner initiates a stock transfer between two Outlets, THE System SHALL decrease the quantity at the source Outlet and increase the quantity at the destination Outlet by the transferred amount.
7. WHEN the Business_Owner sets a Low_Stock_Threshold for a Stock_Item, THE System SHALL store the threshold value for that Stock_Item.
8. IF the available quantity of a Stock_Item is at or below its Low_Stock_Threshold, THEN THE System SHALL generate a low-stock alert for that Stock_Item.
9. IF the available quantity of a Stock_Item reaches zero, THEN THE System SHALL generate an out-of-stock alert for that Stock_Item.
10. WHEN a user with inventory access requests the stock movement history for a Stock_Item, THE System SHALL display each recorded movement with its date, type, quantity, and originating user.
11. WHEN a user with inventory access requests inventory valuation for an Outlet, THE System SHALL display the total valuation calculated from the quantity and recorded cost of each Stock_Item at that Outlet.

## Module Interface

```typescript
interface InventoryService {
  createStockItem(outletId: ID, input: StockItemInput): Promise<StockItem>;      // 1.1
  updateStockItem(stockItemId: ID, input: StockItemInput): Promise<StockItem>;   // 1.2
  deleteStockItem(stockItemId: ID): Promise<void>;                               // 1.3
  getStockLevels(outletId: ID): Promise<StockLevel[]>;                           // 1.4
  recordMovement(movement: StockMovementInput): Promise<StockMovement>;          // 1.5
  transferStock(input: StockTransferInput): Promise<StockTransferResult>;        // 1.6
  setLowStockThreshold(stockItemId: ID, threshold: number): Promise<StockItem>;  // 1.7
  getMovementHistory(stockItemId: ID): Promise<StockMovement[]>;                 // 1.10
  getInventoryValuation(outletId: ID): Promise<ValuationReport>;                 // 1.11
}
```

Low-stock (1.8) and out-of-stock (1.9) alerts are raised by an internal `evaluateStockAlerts(stockItem)` function invoked after every quantity change; the function emits alert events consumed by the Notification Module (see `09-NOTIFICATIONS.md`).

## Data Model Entities

Monetary fields are integers in minor currency units. Quantities are integers.

```typescript
interface StockItem {
  id: ID; outletId: ID; name: string;
  quantity: Quantity;          // never negative (invariant)
  unitCost: Money;             // for valuation (1.11)
  lowStockThreshold: Quantity; // (1.7)
}

interface StockMovement {
  id: ID; stockItemId: ID; type: 'IN' | 'OUT' | 'ADJUST' | 'TRANSFER_IN' | 'TRANSFER_OUT' | 'SALE' | 'REFUND';
  quantityDelta: number;       // signed; sum of deltas + initial == current quantity
  occurredAt: Timestamp; originatingUserId: ID; note?: string;
}
```

## Invariants

- `StockItem.quantity >= 0` at all times (no overselling).
- Sum of a stock item's `StockMovement.quantityDelta` plus its initial quantity equals its current `quantity`.

## Alert Emission (links to Notifications)

After any quantity change, `evaluateStockAlerts(stockItem)` runs:

- If available quantity is at or below `lowStockThreshold`, emit a **low-stock alert** (1.8), consumed by `NotificationService.notifyLowStock` (Req 9.1).
- If available quantity reaches zero, emit an **out-of-stock alert** (1.9), consumed by `NotificationService.notifyOutOfStock` (Req 9.2).

See `09-NOTIFICATIONS.md` for recipient resolution (Business Owner + relevant Outlet Manager).
