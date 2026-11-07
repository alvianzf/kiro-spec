# 07 — Accounting & Cash Flow

**Requirement 7** — covers all 7 acceptance criteria for income/expense entries, expense categorisation, automatic sales revenue, cash balances, liabilities, financial reports, and audited manual adjustments.

> **User Story:** As a Business Owner, I want to record income and expenses and generate financial reports, so that I can understand the financial position of my business.

## Acceptance Criteria

1. WHEN the Business_Owner records an income or expense entry, THE System SHALL store the entry with its amount, date, and category.
2. WHEN the Business_Owner categorises an expense, THE System SHALL assign the expense to one of the configured categories, including rent, utilities, salaries, stock purchases, and operational costs.
3. WHEN a Transaction is completed, THE System SHALL record the Transaction revenue as income associated with the originating Outlet.
4. WHEN a user with accounting access requests cash balances, THE System SHALL display the current cash balance for each Outlet.
5. WHEN the Business_Owner records a supplier payment or outstanding liability, THE System SHALL store the obligation with its amount, due date, and counterparty.
6. WHEN a user with accounting access requests a financial report, THE System SHALL generate an income statement, a profit and loss report, or a cash flow report for the selected period.
7. WHEN the Business_Owner records a manual financial adjustment, THE System SHALL apply the adjustment and record the adjustment in the Audit_Log with the originating user and timestamp.

## Module Interface

```typescript
interface AccountingService {
  recordEntry(input: FinancialEntryInput): Promise<FinancialEntry>; // 7.1, 7.2
  recordTransactionIncome(tx: Transaction): Promise<void>;          // 7.3 (called by SalesService)
  getCashBalances(scope: OutletScope): Promise<CashBalance[]>;      // 7.4
  recordLiability(input: LiabilityInput): Promise<Liability>;       // 7.5
  generateReport(type: ReportType, filter: PeriodFilter): Promise<FinancialReport>; // 7.6
  recordAdjustment(input: AdjustmentInput, actor: ID): Promise<FinancialEntry>; // 7.7 (audited)
}
```

## Data Model Entities

Monetary fields are integers in minor currency units.

```typescript
interface FinancialEntry {
  id: ID; outletId?: ID; type: 'INCOME' | 'EXPENSE';
  amount: Money; date: DateOnly; category: ExpenseCategory | 'SALES_REVENUE';
  isAdjustment: boolean; actorId?: ID;  // (7.7)
}
type ExpenseCategory = 'RENT' | 'UTILITIES' | 'SALARIES' | 'STOCK_PURCHASES' | 'OPERATIONAL';

interface Liability { id: ID; counterparty: string; amount: Money; dueDate: DateOnly; paidAmount: Money; }
interface CashBalance { outletId: ID; balance: Money; }
```

## Notes

- **Automatic revenue recording (7.3):** `recordTransactionIncome(tx)` is called by the Sales Module within the atomic `recordTransaction` flow. The completed transaction's revenue is recorded as a `FinancialEntry` of type `INCOME` with category `SALES_REVENUE`, associated with the originating Outlet. See `03-SALES-TRANSACTIONS.md`.
- **Audited manual adjustments (7.7):** `recordAdjustment(input, actor)` applies the adjustment, flags the `FinancialEntry` with `isAdjustment: true` and `actorId`, and records the adjustment in the Audit_Log with the originating user and timestamp. See `10-SECURITY-ACCESS.md`.
- **Expense categories (7.2):** `RENT`, `UTILITIES`, `SALARIES`, `STOCK_PURCHASES`, `OPERATIONAL`.

## Cross-References

- Liabilities with an outstanding balance on or after their due date trigger pending-payment notifications (9.6) — see `09-NOTIFICATIONS.md`.
- Expenses and estimated profit surface on the dashboard overview (8.1) — see `08-DASHBOARD-REPORTS.md`.
- Financial record operations are recorded in the Audit_Log (10.6) — see `10-SECURITY-ACCESS.md`.
