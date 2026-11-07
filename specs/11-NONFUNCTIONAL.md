# 11 — Cross-Cutting Non-Functional Requirements

This file collects the system-wide non-functional concerns: **Performance & Concurrency (Req 11)**, **Reliability & Data Integrity recap (Req 12)**, **Scalability (Req 13)**, and **Data Ownership, Backup & Recovery (Req 14)**.

## Requirement 11: Performance and Concurrency

> **User Story:** As a Business Owner, I want the system to stay responsive under load, so that multiple outlets can transact at the same time without delays.

### Acceptance Criteria

1. WHEN the System processes concurrent Transactions from multiple Outlets, THE System SHALL record each Transaction without data loss.
2. WHEN a user requests the Dashboard with data aggregated across all Outlets, THE Dashboard SHALL return the requested metrics within 5 seconds under normal operating load.
3. WHEN a Cashier completes a Transaction at the POS_Interface, THE POS_Interface SHALL confirm the completed Transaction within 2 seconds under normal operating load.

### Design Notes

- **Concurrency without data loss (11.1):** Transactions are recorded in ACID database transactions with row locks on affected stock, and idempotency via the unique `transaction_uid` constraint. See `03-SALES-TRANSACTIONS.md`.
- **Dashboard latency (11.2):** Pre-aggregated materialised views / summary tables refreshed on sync keep aggregate reads under 5 seconds. See `08-DASHBOARD-REPORTS.md`.
- **POS confirmation (11.3):** The POS computes totals/change locally with the pure domain core and confirms the sale (receipt) in under 2 seconds, syncing to the server asynchronously. See `04-POINT-OF-SALE.md`.

## Requirement 12: Reliability and Data Integrity (recap)

> **User Story:** As a Business Owner, I want transactions and financial records stored accurately, so that my records are trustworthy.

### Acceptance Criteria

1. WHEN a Transaction is recorded, THE System SHALL persist the Transaction so that it remains available after the persistence operation completes.
2. IF two Transaction submissions carry the same Transaction identifier, THEN THE System SHALL persist only one Transaction record for that identifier.
3. IF a Transaction submission fails to persist completely, THEN THE System SHALL discard the partial Transaction and return an error without altering stock quantities or financial records.

The full atomic/idempotent design (BEGIN; insert; deduct stock; record income; COMMIT; unique `transaction_uid`) is detailed in `03-SALES-TRANSACTIONS.md`.

## Requirement 13: Scalability

> **User Story:** As a Business Owner, I want the system to grow with my business, so that adding outlets and staff does not require replacing the system.

### Acceptance Criteria

1. WHEN the Business_Owner adds a new Outlet, THE System SHALL make the Outlet available for inventory, product, transaction, employee, and reporting operations.
2. WHEN the Business_Owner adds employees, Products, or Stock_Items, THE System SHALL store and operate on the added records without requiring reconfiguration of existing records.

### Design Notes

Outlet-scoped records carry `outletId`, and the RBAC outlet scope (`10-SECURITY-ACCESS.md`) generalises to any number of outlets. Adding records does not require reconfiguring existing ones — new outlets, employees, products, and stock items are additive across all domains.

## Requirement 14: Data Ownership, Backup, and Recovery

> **User Story:** As a Business Owner, I want my data hosted on infrastructure I control and backed up automatically, so that I own my data and can recover from failures.

### Acceptance Criteria

1. WHERE the System is deployed, THE System SHALL store business data on infrastructure designated by the Business_Owner.
2. THE System SHALL create an automated backup of business data on a configured schedule.
3. WHEN the Business_Owner initiates a data restoration from a selected backup, THE System SHALL restore the business data from that backup.
4. IF an automated backup fails to complete, THEN THE System SHALL send a backup-failure notification to the Business_Owner.

### Module Interface

```typescript
interface BackupService {
  runScheduledBackup(): Promise<BackupResult>;   // 14.2 (scheduled job)
  listBackups(): Promise<BackupMeta[]>;
  restore(backupId: ID): Promise<RestoreResult>; // 14.3
}
```

A failed backup (14.4) triggers `NotificationService.notifyBackupFailure` — see `09-NOTIFICATIONS.md`.

### Design Notes

- **Self-hosting (14.1):** The system is a monolithic modular backend deployable on infrastructure the Business Owner controls. Business data is stored on that designated infrastructure, and backups go to an owner-controlled backup store. See the architecture diagram in `00-OVERVIEW.md`.
- **Automated backup schedule (14.2):** The job scheduler (e.g., BullMQ on Redis) runs `runScheduledBackup()` on the configured schedule.
- **Restore (14.3):** `restore(backupId)` restores business data from a selected backup.
- **Backup-failure notification (14.4):** A failed scheduled backup invokes `NotificationService.notifyBackupFailure`, notifying the Business Owner.
