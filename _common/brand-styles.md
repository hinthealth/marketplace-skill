# Hint Brand & UI Styles (shared fragment)

Tokens for embedded surfaces so the partner's app feels native inside the Hint portal. The Node.js reference renderers in [`node-template.md`](./node-template.md) already use these — port the same values to other stacks.

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
