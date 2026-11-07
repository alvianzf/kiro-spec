# Requirements Document

## Introduction

This document defines the requirements for a cashier / point-of-sale (POS) application whose primary user is the Business Owner. The application enables the Business Owner to monitor sales, manage inventory, oversee employees, handle payroll, and track business finances across multiple outlets from a centralised dashboard. The application also provides a POS interface that allows Cashiers to process customer transactions and Outlet Managers to oversee daily operations within their assigned outlets.

The system supports three user roles with distinct levels of access and is designed to operate reliably across multiple outlets with concurrent transactions. Business data is hosted on infrastructure controlled by the Business Owner, with automated backup and recovery.

## Glossary

- **System**: The cashier / point-of-sale application described in this document, including its server, database, dashboard, and POS components.
- **POS_Interface**: The component of the System used by Cashiers to process customer transactions.
- **Dashboard**: The centralised component of the System that presents aggregated business metrics and reports.
- **Business_Owner**: A user with full access to all outlets, transactions, inventory, products, employees, payroll, financial records, and configuration.
- **Outlet_Manager**: A user who manages daily operations, employees, inventory, and sales within one or more assigned outlets.
- **Cashier**: A user who processes sales transactions and manages their own cashier shifts within an assigned outlet.
- **Outlet**: A physical business location that holds its own stock, employees, transactions, and cash balances.
- **Stock_Item**: A quantity of a physical good held in inventory at a specific Outlet.
- **Product**: A sellable entity with attributes such as name, category, price, cost, SKU, and barcode, which may be linked to one or more Stock_Items.
- **Product_Variation**: A distinct sellable configuration of a Product (for example, size or colour).
- **Transaction**: A record of a completed or attempted sale, including products, quantities, prices, discounts, payment methods, and the Cashier who processed it.
- **Refund**: A reversal of a previously completed Transaction, in full or in part.
- **Shift**: A period during which a Cashier processes transactions, bounded by a recorded opening cash balance and a recorded closing cash balance.
- **QRIS**: A standardised Indonesian QR-code payment method supported by the System.
- **Payroll_Record**: A record of salary, allowances, bonuses, overtime, and deductions calculated for an employee over a pay period.
- **Timesheet**: A record of an employee's clock-in and clock-out events, late arrivals, absences, and overtime for a pay period.
- **Audit_Log**: A chronological record of security-relevant and financially significant operations performed in the System.
- **Low_Stock_Threshold**: The minimum quantity value configured for a Stock_Item at or below which the System raises a low-stock alert.
- **Role**: A named set of permissions assigned to a user (Business_Owner, Outlet_Manager, or Cashier).

## Requirements

### Requirement 1: Stock and Inventory Management

**User Story:** As a Business Owner, I want to manage stock items across all outlets, so that I can maintain accurate inventory and avoid shortages.

#### Acceptance Criteria

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

### Requirement 2: Product Management

**User Story:** As a Business Owner, I want to manage products and link them to inventory, so that stock is deducted automatically and I can analyse product performance.

#### Acceptance Criteria

1. WHEN the Business_Owner submits a valid new product, THE System SHALL create the Product with its name, category, price, cost, SKU, and barcode.
2. WHEN the Business_Owner submits an edit to an existing Product, THE System SHALL update the Product with the submitted values.
3. WHEN the Business_Owner requests deletion of a Product, THE System SHALL remove the Product from the catalogue.
4. WHEN the Business_Owner assigns a Product to one or more Outlets, THE System SHALL make the Product available for sale only at the assigned Outlets.
5. WHEN the Business_Owner creates a Product_Variation for a Product, THE System SHALL store the Product_Variation and make it available for sale under that Product.
6. WHERE a Product is linked to a Stock_Item, WHEN a Transaction that includes the Product is completed, THE System SHALL deduct the sold quantity from the linked Stock_Item.
7. IF the Business_Owner submits a Product with a SKU or barcode that already exists, THEN THE System SHALL reject the submission and return a descriptive error.
8. WHEN a user with product access requests product sales performance, THE System SHALL display the quantity sold and revenue generated for each Product over the selected period.
9. WHEN a user with product access requests best-selling products, THE System SHALL display Products ranked in descending order of quantity sold over the selected period.

### Requirement 3: Sales and Transaction Oversight

**User Story:** As a Business Owner, I want to view and manage transactions across all outlets, so that I can monitor sales activity and authorise corrections.

#### Acceptance Criteria

1. WHEN the Business_Owner requests the transaction list, THE System SHALL display Transactions from all Outlets.
2. WHEN a Transaction is synchronised from an Outlet, THE System SHALL make the Transaction visible in the transaction list within 60 seconds of synchronisation.
3. WHEN a user with transaction access selects a Transaction, THE System SHALL display the products, quantities, prices, discounts, payment methods, and Cashier associated with that Transaction.
4. WHEN a user with transaction access applies a filter by Outlet, date, product, or employee, THE System SHALL display only the Transactions that match the selected filter.
5. WHEN a Refund, cancellation, or correction is requested, THE System SHALL require authorisation from a Business_Owner or Outlet_Manager before applying the change.
6. WHEN an authorised Refund is completed for a Transaction linked to a Stock_Item, THE System SHALL restore the refunded quantity to the linked Stock_Item.
7. WHEN a user with transaction access requests a sales summary, THE System SHALL display total sales per Outlet and total sales across all Outlets for the selected period.
8. WHEN a user requests a receipt for a completed Transaction, THE System SHALL generate a receipt that is available to print or share.

### Requirement 4: Point of Sale

**User Story:** As a Cashier, I want a simple interface to process customer transactions, so that I can serve customers accurately and quickly.

#### Acceptance Criteria

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

### Requirement 5: Employee Management

**User Story:** As a Business Owner, I want to manage employee accounts, schedules, and attendance, so that I can oversee my workforce across outlets.

#### Acceptance Criteria

1. WHEN the Business_Owner submits a valid new employee account, THE System SHALL create the employee account with the assigned Role.
2. WHEN the Business_Owner submits an edit to an employee account, THE System SHALL update the account with the submitted values.
3. WHEN the Business_Owner or Outlet_Manager assigns an employee to one or more Outlets, THE System SHALL record the Outlet assignment for that employee.
4. WHEN the Business_Owner or Outlet_Manager configures a working schedule or Shift for an employee, THE System SHALL store the schedule for that employee.
5. WHEN an employee clocks in or clocks out, THE System SHALL record the event with its timestamp in the employee's Timesheet.
6. WHEN an employee clocks in after the scheduled Shift start time, THE System SHALL record the event as a late arrival.
7. WHEN an employee works beyond the scheduled Shift end time, THE System SHALL record the additional duration as overtime.
8. WHEN an Outlet_Manager approves an attendance correction, THE System SHALL update the affected Timesheet with the corrected values and record the approving user in the Audit_Log.

### Requirement 6: Payroll Management

**User Story:** As a Business Owner, I want payroll calculated from attendance and approved timesheets, so that I can pay employees accurately and keep payroll records.

#### Acceptance Criteria

1. WHEN the Business_Owner configures an employee's salary, hourly rate, or daily wage, THE System SHALL store the configured compensation for that employee.
2. WHEN the Business_Owner configures allowances, bonuses, overtime rates, or deductions for an employee, THE System SHALL store those values for payroll calculation.
3. WHEN the Business_Owner initiates payroll calculation for a pay period, THE System SHALL calculate each employee's Payroll_Record from the configured compensation and the approved Timesheet for that period.
4. WHEN payroll calculation for a pay period completes, THE System SHALL generate a monthly payroll summary and an individual payslip for each included employee.
5. WHEN the Business_Owner records a salary payment against a Payroll_Record, THE System SHALL mark the paid amount and recalculate the outstanding payroll balance.
6. WHEN a user with payroll access requests historical payroll, THE System SHALL display the Payroll_Records for the selected pay periods.

### Requirement 7: Accounting and Cash Flow

**User Story:** As a Business Owner, I want to record income and expenses and generate financial reports, so that I can understand the financial position of my business.

#### Acceptance Criteria

1. WHEN the Business_Owner records an income or expense entry, THE System SHALL store the entry with its amount, date, and category.
2. WHEN the Business_Owner categorises an expense, THE System SHALL assign the expense to one of the configured categories, including rent, utilities, salaries, stock purchases, and operational costs.
3. WHEN a Transaction is completed, THE System SHALL record the Transaction revenue as income associated with the originating Outlet.
4. WHEN a user with accounting access requests cash balances, THE System SHALL display the current cash balance for each Outlet.
5. WHEN the Business_Owner records a supplier payment or outstanding liability, THE System SHALL store the obligation with its amount, due date, and counterparty.
6. WHEN a user with accounting access requests a financial report, THE System SHALL generate an income statement, a profit and loss report, or a cash flow report for the selected period.
7. WHEN the Business_Owner records a manual financial adjustment, THE System SHALL apply the adjustment and record the adjustment in the Audit_Log with the originating user and timestamp.

### Requirement 8: Dashboard and Reports

**User Story:** As a Business Owner, I want a centralised dashboard with reports and exports, so that I can monitor overall business performance at a glance.

#### Acceptance Criteria

1. WHEN the Business_Owner opens the Dashboard, THE Dashboard SHALL display total sales, total transactions, total expenses, and estimated profit for the selected period.
2. WHEN the Business_Owner views sales performance, THE Dashboard SHALL display sales totals by Outlet and by employee for the selected period.
3. WHEN the Business_Owner views product performance, THE Dashboard SHALL display best-selling Products and slow-moving Products for the selected period.
4. WHEN the Business_Owner views inventory status, THE Dashboard SHALL display current stock levels and the list of Stock_Items at or below their Low_Stock_Threshold.
5. WHEN the Business_Owner views workforce status, THE Dashboard SHALL display employee attendance summaries and payroll summaries for the selected period.
6. WHEN the Business_Owner selects a daily, weekly, monthly, or custom reporting period, THE Dashboard SHALL display the reported metrics for the selected period.
7. WHEN the Business_Owner applies an Outlet filter, THE Dashboard SHALL display only the data for the selected Outlet.
8. WHEN the Business_Owner requests an export of a report, THE System SHALL produce the report in the selected PDF, Excel, or CSV format.

### Requirement 9: Notifications

**User Story:** As a Business Owner, I want to be notified of important business events, so that I can respond to issues promptly.

#### Acceptance Criteria

1. IF the available quantity of a Stock_Item is at or below its Low_Stock_Threshold, THEN THE System SHALL send a low-stock notification to the Business_Owner and the relevant Outlet_Manager.
2. IF the available quantity of a Stock_Item reaches zero, THEN THE System SHALL send an out-of-stock notification to the Business_Owner and the relevant Outlet_Manager.
3. WHEN a Refund or transaction correction exceeds a configured threshold amount, THE System SHALL send an unusual-activity notification to the Business_Owner.
4. IF a Cashier's recorded closing cash balance differs from the System-calculated expected balance for the Shift, THEN THE System SHALL send a cash-register-discrepancy notification to the Business_Owner and the relevant Outlet_Manager.
5. IF an employee has a recorded absence or late arrival, THEN THE System SHALL send an attendance-issue notification to the relevant Outlet_Manager.
6. WHEN a Payroll_Record or supplier payment has an outstanding balance on or after its due date, THE System SHALL send a pending-payment notification to the Business_Owner.

### Requirement 10: User Access and Security

**User Story:** As a Business Owner, I want secure authentication and role-based access control, so that sensitive business and employee data is protected.

#### Acceptance Criteria

1. WHEN a user submits valid credentials, THE System SHALL authenticate the user and establish an authenticated session.
2. IF a user submits invalid credentials, THEN THE System SHALL deny access and return an authentication error.
3. WHEN an authenticated user requests a resource, THE System SHALL grant access only to the operations permitted by the user's Role.
4. WHERE a user holds the Cashier or Outlet_Manager Role, THE System SHALL restrict that user's access to the Outlets assigned to the user.
5. WHEN the Business_Owner configures the permissions for a Role or user, THE System SHALL apply the configured permissions on the user's subsequent requests.
6. WHEN a user performs an operation on a financial record, a payroll record, or a security configuration, THE System SHALL record the operation in the Audit_Log with the originating user, the affected record, and the timestamp.

### Requirement 11: Performance and Concurrency

**User Story:** As a Business Owner, I want the system to stay responsive under load, so that multiple outlets can transact at the same time without delays.

#### Acceptance Criteria

1. WHEN the System processes concurrent Transactions from multiple Outlets, THE System SHALL record each Transaction without data loss.
2. WHEN a user requests the Dashboard with data aggregated across all Outlets, THE Dashboard SHALL return the requested metrics within 5 seconds under normal operating load.
3. WHEN a Cashier completes a Transaction at the POS_Interface, THE POS_Interface SHALL confirm the completed Transaction within 2 seconds under normal operating load.

### Requirement 12: Reliability and Data Integrity

**User Story:** As a Business Owner, I want transactions and financial records stored accurately, so that my records are trustworthy.

#### Acceptance Criteria

1. WHEN a Transaction is recorded, THE System SHALL persist the Transaction so that it remains available after the persistence operation completes.
2. IF two Transaction submissions carry the same Transaction identifier, THEN THE System SHALL persist only one Transaction record for that identifier.
3. IF a Transaction submission fails to persist completely, THEN THE System SHALL discard the partial Transaction and return an error without altering stock quantities or financial records.

### Requirement 13: Scalability

**User Story:** As a Business Owner, I want the system to grow with my business, so that adding outlets and staff does not require replacing the system.

#### Acceptance Criteria

1. WHEN the Business_Owner adds a new Outlet, THE System SHALL make the Outlet available for inventory, product, transaction, employee, and reporting operations.
2. WHEN the Business_Owner adds employees, Products, or Stock_Items, THE System SHALL store and operate on the added records without requiring reconfiguration of existing records.

### Requirement 14: Data Ownership, Backup, and Recovery

**User Story:** As a Business Owner, I want my data hosted on infrastructure I control and backed up automatically, so that I own my data and can recover from failures.

#### Acceptance Criteria

1. WHERE the System is deployed, THE System SHALL store business data on infrastructure designated by the Business_Owner.
2. THE System SHALL create an automated backup of business data on a configured schedule.
3. WHEN the Business_Owner initiates a data restoration from a selected backup, THE System SHALL restore the business data from that backup.
4. IF an automated backup fails to complete, THEN THE System SHALL send a backup-failure notification to the Business_Owner.

### Requirement 15: Usability

**User Story:** As a Cashier, I want the POS interface to be easy to operate, so that I can process transactions with minimal training.

#### Acceptance Criteria

1. THE POS_Interface SHALL present the search, cart, payment, and receipt actions required to complete a Transaction on a single workflow without navigation to the Dashboard.
2. WHEN a Cashier enters an invalid value during a Transaction, THE POS_Interface SHALL display a descriptive message identifying the invalid value.
