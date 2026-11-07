# 04 — Point of Sale (V3 changes)

> Delta file. Baseline: `../v1/04-POINT-OF-SALE.md`. Source: `cashier-business-owner` V3. Presentation only.

- Primary actions (for example Complete Sale) use the primary colour with white text and hover/pressed tokens. Surfaces are white with black text.
- Two-tone focus ring on all interactive elements; it is never removed.
- Invalid-value messages (Req 15.2) use the shared status component with icon, text, and `aria-describedby` linkage.
- No colour literals in POS components (lint rule).