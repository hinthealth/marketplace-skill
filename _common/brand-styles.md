# Hint Brand & UI Styles (shared fragment)

Tokens for embedded surfaces so the partner's app feels native inside the Hint portal. The Node.js reference renderers in [`node-template.md`](./node-template.md) already use these — port the same values to other stacks.

> **Two distinct icon assets — don't conflate.** The marketplace listing icon (`partner_product.icon` / `icon_url`) is a filled brand mark for the listing tile. The Core Page sidebar icon (`surface.core_page_icon` / `core_page_icon_url`) is an outlined Material-Symbols-style glyph for the practice's left nav. Different aesthetic, different field, different consumer. Reusing the listing icon for the sidebar leaves a saturated brand block fighting Hint's nav. Sidebar specifics live in [`sidebar-icons.md`](./sidebar-icons.md); the official guideline is at <https://developers.hint.com/docs/partner-asset-guidelines>.

## Font

```css
font-family: "Wix Madefor Text", system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
```

Load via Google Fonts:

```html
<link href="https://fonts.googleapis.com/css2?family=Wix+Madefor+Text:wght@400;500;700&display=swap" rel="stylesheet">
```

## Colors

| Token | Hex | Use for |
|-------|-----|---------|
| Primary | `#0E68E2` | Buttons, links, active states |
| Primary hover | `#083C82` | Button hover |
| Primary active | `#052652` | Button press |
| Primary disabled | `#C1DAFB` | Disabled buttons |
| Background | `#FFFFFF` | Cards, page background |
| Surface | `#EFF5FD` | Table headers, secondary backgrounds |
| Text primary | `#1D2334` | Default body text |
| Text secondary | `#43739E` | Labels, secondary info |
| Text disabled | `#8A90A5` | Disabled text |
| Text on primary | `#FFFFFF` | Text on primary buttons |
| Border subtle | `#E6ECF4` | Table rows, card outlines |
| Border default | `#C2C7CF` | Input borders |
| Field outline | `#C4DCF8` | Input focus |
| Success text | `#00602D` | Success messages |
| Success bg | `#DCFCE7` | Success badges/alerts |
| Error text | `#CD0211` | Error messages |
| Error bg | `#FFE2E2` | Error badges/alerts |
| Warning text | `#973C00` | Warning messages |
| Warning bg | `#FEF3C6` | Warning badges/alerts |

## Typography scale

| Style | Weight / Size / Line height | Use for |
|-------|----------------------------|---------|
| heading-md | 400 / 24px / 32px | Page titles |
| heading-sm | 400 / 18px / 24px | Section headers, card headers |
| title-md-bold | 700 / 16px / 24px | Modal titles |
| title-sm | 500 / 14px / 20px | Column headers, button text |
| label-md | 500 / 14px / 16px | Form labels |
| body-md | 400 / 14px / 20px | Body text, descriptions |
| body-sm | 400 / 12px / 16px | Timestamps, metadata |

## Components

```css
.btn-primary {
  background: #0E68E2;
  color: #FFFFFF;
  border: none;
  border-radius: 8px;
  padding: 8px 16px;
  font-family: inherit;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
}
.btn-primary:hover { background: #083C82; }
.btn-primary:active { background: #052652; }
.btn-primary:disabled { background: #C1DAFB; cursor: default; }

.card {
  background: #FFFFFF;
  border: 1px solid #E6ECF4;
  border-radius: 12px;
  padding: 24px;
}

input, select, textarea {
  border: 1px solid #C4DCF8;
  border-radius: 8px;
  padding: 8px 12px;
  font-family: inherit;
  font-size: 14px;
  color: #1D2334;
}
input:focus { outline: none; border-color: #0E68E2; box-shadow: 0 0 0 3px rgba(14, 104, 226, 0.25); }
```

## Responsive baseline — ship with every app, regardless of surface

**Marketplace apps are routinely opened on tablets and phones inside the practice mobile experience.** A desktop-only layout — fixed multi-column grids, padding tuned for ≥800 px viewports, rows with right-aligned chips — breaks visibly on iPhone-class viewports (393 px wide) the first time it ships. Building responsive from v1 avoids the retroactive patching cycle.

Two breakpoints cover ~all real cases:

- `max-width: 720px` — tablet
- `max-width: 480px` — phone

```css
/* Base — desktop default */
body { padding: 24px; }
h1 { font-size: 28px; }
.kpi-value { font-size: 36px; }
.kpis, .cards, .dashboard-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; }

@media (max-width: 720px) {
  body { padding: 12px; }
  h1 { font-size: 22px; }
  .kpi-value { font-size: 26px; }
  /* Any multi-column grid collapses to a single column */
  .kpis, .cards, .dashboard-grid { grid-template-columns: 1fr; }
  /* Rows with right-aligned secondary content (chips, meta) restructure into a 2-row layout
     so the secondary content can wrap below the primary instead of cramming into a narrow column. */
  .worklist-row { display: block; }
  .worklist-row .row-meta { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 6px; }
}

@media (max-width: 480px) {
  body { padding: 8px; }
  h1 { font-size: 20px; }
  .kpi-value { font-size: 24px; }
}
```

The `<meta name="viewport" content="width=device-width, initial-scale=1">` tag in the HTML head is essential — without it iOS Safari scales the page like a desktop site and the breakpoints never fire. The template includes it; don't drop it.

## Empty-state copy patterns

Real marketplace apps hit several "no data yet" states routinely:

| State | When | Recommended copy |
|---|---|---|
| No active members | Fresh practice, no memberships yet | "No active members yet. New patients you enroll in Hint will appear here." |
| No `<resource>` data yet | Membership exists, but no labs / charges / interactions of the relevant type | "No labs on file yet for this panel. They'll appear here as they're imported from the EMR." |
| Sandbox-only context | The user is in a sandbox practice with no seed data | "This is a sandbox practice. Install **Sandbox Studio** from the Hint marketplace to seed realistic test data." |
| Backend unavailable | API or DB error during render | "Couldn't load <resource> right now. Refresh in a moment, or contact your practice admin if this persists." |

Ship these states from v1 — every fresh install hits at least one of them, and a blank screen reads as broken even when the app is working correctly. The recommended pattern is a centered card with a one-line headline and a brief follow-up sentence; no spinner unless an action is pending.
