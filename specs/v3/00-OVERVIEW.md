# 00 — Overview (V3 changes)

> Delta file. Baseline: `../v1/00-OVERVIEW.md` plus `../v2/00-OVERVIEW.md`. Source spec: `cashier-business-owner` V3. Presentation only: no requirements, API, or data changes.

## Technology stack addition

- **Client styling:** Dashboard and POS share one theme package (`@app/theme`) that exports CSS custom properties and a typed TypeScript theme object from one token source. Components reference tokens only.

## Brand theme

Primary `#990000`, secondary `#000000`, accent `#FFFFFF`, applied to the Dashboard, POS_Interface, and Login_Page.

| Token | Value | Purpose |
| --- | --- | --- |
| `--color-primary` | `#990000` | Primary actions |
| `--color-primary-hover` / `-pressed` | `#7A0000` / `#5C0000` | Interaction states |
| `--color-primary-tint` | `#FBEAEA` | Subtle tinted background |
| `--color-secondary` | `#000000` | Text, top bar, sidebar, strong borders |
| `--color-accent` / `--color-surface` | `#FFFFFF` | Surfaces, and text/icons on dark backgrounds |
| `--color-surface-muted` | `#F5F5F5` | Page background |
| `--color-border` / `--color-border-strong` | `#D4D4D4` / `#000000` | Borders |
| `--color-text-muted` | `#595959` | Secondary text |
| `--color-success` / `--color-warning` | `#1B5E20` / `#8A5A00` | Status colours distinct from primary |
| `--focus-ring-inner` / `--focus-ring-outer` | `#FFFFFF` / `#000000` | Two-tone focus ring |

Derived values are proposals; the automated contrast test is the authority.

## Accessibility rules

- `#990000` on `#FFFFFF` and `#FFFFFF` on `#990000` are about 8.9:1 (pass AA and AAA). `#990000` on `#000000` is about 2.4:1 and fails AA, so primary-coloured text, icons, or borders are never placed on black.
- Only foreground/background pairs in `PERMITTED_PAIRS` may be used (4.5:1 normal text, 3:1 large text and UI boundaries).
- Status never relies on colour alone: every status uses icon + text label + border pattern.
- Colour literals are banned outside the theme package (lint rule). Full WCAG validation still requires manual testing with assistive technologies.