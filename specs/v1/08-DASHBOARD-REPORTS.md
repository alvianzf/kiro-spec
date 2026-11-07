# 08 — Dashboard & Reports

**Requirement 8** (Dashboard and Reports) plus the **5-second aggregation target from Requirement 11.2**. Centralised metrics, reporting periods, outlet filtering, and exports.

> **User Story:** As a Business Owner, I want a centralised dashboard with reports and exports, so that I can monitor overall business performance at a glance.

## Acceptance Criteria — Requirement 8

1. WHEN the Business_Owner opens the Dashboard, THE Dashboard SHALL display total sales, total transactions, total expenses, and estimated profit for the selected period.
2. WHEN the Business_Owner views sales performance, THE Dashboard SHALL display sales totals by Outlet and by employee for the selected period.
3. WHEN the Business_Owner views product performance, THE Dashboard SHALL display best-selling Products and slow-moving Products for the selected period.
4. WHEN the Business_Owner views inventory status, THE Dashboard SHALL display current stock levels and the list of Stock_Items at or below their Low_Stock_Threshold.
5. WHEN the Business_Owner views workforce status, THE Dashboard SHALL display employee attendance summaries and payroll summaries for the selected period.
6. WHEN the Business_Owner selects a daily, weekly, monthly, or custom reporting period, THE Dashboard SHALL display the reported metrics for the selected period.
7. WHEN the Business_Owner applies an Outlet filter, THE Dashboard SHALL display only the data for the selected Outlet.
8. WHEN the Business_Owner requests an export of a report, THE System SHALL produce the report in the selected PDF, Excel, or CSV format.

## Related Criterion — Requirement 11.2 (Performance)

2. WHEN a user requests the Dashboard with data aggregated across all Outlets, THE Dashboard SHALL return the requested metrics within 5 seconds under normal operating load.

## Module Interface

```typescript
interface DashboardService {
  getOverview(filter: PeriodFilter, scope: OutletScope): Promise<DashboardOverview>; // 8.1
  getSalesPerformance(filter, scope): Promise<SalesPerformanceView>; // 8.2
  getProductPerformance(filter, scope): Promise<ProductPerformanceView>; // 8.3
  getInventoryStatus(scope): Promise<InventoryStatusView>; // 8.4
  getWorkforceStatus(filter, scope): Promise<WorkforceStatusView>; // 8.5
  exportReport(view: ReportView, format: 'PDF' | 'EXCEL' | 'CSV'): Promise<ExportFile>; // 8.8
}
```

## Performance Strategy (11.2)

To meet the 5-second aggregation target (11.2), dashboard reads use **pre-aggregated materialised views / summary tables** refreshed on transaction sync, rather than scanning raw transactions on each request. The reporting period (daily, weekly, monthly, or custom, per 8.6) and outlet filter (8.7) constrain the queries against these summary tables.

## Export Formats (8.8)

Reports can be produced as **PDF**, **Excel**, or **CSV**.

## Cross-References

- Sales totals draw from the Sales module (`03-SALES-TRANSACTIONS.md`).
- Product performance (best-selling / slow-moving) reuses the Product module analysis (`02-PRODUCTS.md`).
- Inventory status reuses stock levels and low-stock lists (`01-INVENTORY.md`).
- Workforce status reuses attendance (`05-EMPLOYEES.md`) and payroll summaries (`06-PAYROLL.md`).
- Outlet scope is injected by RBAC (`10-SECURITY-ACCESS.md`); `OutletScope` governs what the Business Owner vs. a Manager can aggregate.
