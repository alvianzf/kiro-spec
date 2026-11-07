# 09 — Notifications (V3 changes)

> Delta file. Baseline: `../v1/09-NOTIFICATIONS.md`. Source: `cashier-business-owner` V3. Presentation only.

Alert and status displays (low-stock, out-of-stock, cash-register discrepancy; Req 1.8, 1.9, 9.4) render as icon + text label + border pattern:

- Error / out-of-stock / discrepancy: error icon, explicit label, 2px strong border with a solid left bar on the primary tint.
- Low-stock / warning: warning icon, label, dashed border in the warning colour.
- Success: check icon, label, success-coloured border.
- There is no separate bright-red error token, because the primary colour is itself a deep red.