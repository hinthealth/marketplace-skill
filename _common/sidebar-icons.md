# Sidebar icons for Core Page apps (shared fragment)

**Read this when an app registers a `core_page` surface.** The Core Page icon shows up in every practice's left sidebar — right next to Patients / Employers / Reports / Admin — and it's what the practitioner sees every day. Getting it right is high-leverage UX; getting it wrong (or skipping it, so Hint falls back to the generic placeholder) leaves a permanent "looks unfinished" stain on the install.

## The sidebar icon is NOT the listing icon

Two distinct assets, easy to conflate:

| Field | Where it shows | Visual style |
|---|---|---|
| `product.icon` / `icon_url` | The marketplace listing tile (`/apps/<slug>` and the browse grid) | **Filled brand mark** — square card, your brand color, your glyph, your typography. A logo. |
| `app.surfaces[core_page].core_page_icon` / `core_page_icon_url` | The practice's left nav sidebar | **Outlined Material Symbols glyph** — single-color (Hint tints it for hover / selected states), no background, no fill, matches Hint's nav style |

If you reuse the listing icon for the sidebar, the result is a saturated brand-color block that fights every other nav item visually. The full guideline is at <https://developers.hint.com/docs/partner-asset-guidelines> — bullet 1 (listing) vs bullet 3 (sidebar).

## Picking the glyph

Material Symbols (Google's open-source icon set, https://fonts.google.com/icons) is the canonical source — it's what Hint's own nav uses, so a sidebar icon picked from there drops in cleanly with no styling adjustments. Browse the set in the picker, pick the **Outlined** style at weight 400 (normal grade), and note the glyph's name (e.g. `science`, `receipt_long`, `groups`).

**Domain → suggested glyph.** Use the partner's app description (gathered in `create-app` Step 1) to pick a starting glyph. None of this is prescriptive — if the app's domain has a more specific match in Material Symbols, use it.

| App domain | Primary | Alternate |
|---|---|---|
| Labs / clinical data / diagnostics | `science` | `biotech` |
| Billing / revenue / payments | `receipt_long` | `payments` |
| Messaging / outreach / patient comms | `forum` | `mail` |
| Scheduling / appointments | `calendar_month` | `schedule` |
| Members / patients / panel | `groups` | `person` |
| Reports / analytics / dashboards | `bar_chart` | `monitoring` |
| Settings / configuration | `settings` | `tune` |

## Fetch the SVG (canonical URL)

Material Symbols' source lives in `google/material-design-icons` on GitHub. The raw SVG for any glyph follows a predictable URL pattern, so the LLM can fetch it on-demand without bundling anything:

```bash
GLYPH="science"  # ← any name from fonts.google.com/icons
curl -sL "https://raw.githubusercontent.com/google/material-design-icons/master/symbols/web/${GLYPH}/materialsymbolsoutlined/${GLYPH}_24px.svg"
```

This is the **outlined** weight-400 variant. Other weights / fills follow the same pattern (`materialsymbolsoutlined` → `materialsymbolsrounded` → `materialsymbolssharp`, plus `_wght100..900_grad..._fill1` qualifier folders) but Hint's sidebar expects outlined 400, so stick with this URL form unless there's a specific reason not to.

## Cleanup rule

The upstream SVG ships with `height="24"` and `width="24"` attributes hard-coded. Hint's sidebar sizes the icon via CSS; the explicit attributes fight that. Strip them, along with any explicit `fill` so the icon inherits `currentColor` and lets Hint tint it for hover / selected states:

```
- Keep:   xmlns, viewBox, <path> data
- Remove: height, width, fill
```

In one line:

```bash
sed -E 's/ (height|width|fill)="[^"]*"//g'
```

## Encoding for the PATCH

The `core_page_icon` field on the surface takes a `data:image/svg+xml;base64,...` URI (max 5MB, but SVGs are kilobytes so this is irrelevant in practice). Full one-liner that fetches → cleans → encodes:

```bash
GLYPH="science"
CLEAN_SVG=$(curl -sL "https://raw.githubusercontent.com/google/material-design-icons/master/symbols/web/${GLYPH}/materialsymbolsoutlined/${GLYPH}_24px.svg" \
  | sed -E 's/ (height|width|fill)="[^"]*"//g')
ICON_DATA_URI="data:image/svg+xml;base64,$(printf '%s' "$CLEAN_SVG" | base64 | tr -d '\n')"
```

Then attach it to the Core Page surface:

```bash
curl -s -X PATCH "$HINT_API_URL/api/partner/products/$PRODUCT_ID/app/surfaces/$SURFACE_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"surface\": {
    \"core_page_icon\": \"$ICON_DATA_URI\",
    \"core_page_icon_label\": \"$LABEL\"
  }}"
```

Confirm by `GET`-ing the surface — `core_page_icon_url` (read-side field name; asymmetric with the PATCH-side `core_page_icon`) should now be a populated URL instead of `null`.

## Picking the label

`core_page_icon_label` is the sidebar text shown next to the icon. It's a tooltip on hover on desktop and visible text on mobile sidebars. Keep it short:

- **Default:** the app's name, truncated to ≤14 characters.
- **If the name is longer than 14 characters:** ask the partner for a short form. "Panel Lab Health Snapshots" → "Panel Labs"; "Outreach & Messaging Hub" → "Outreach". Don't auto-truncate mid-word — it produces "Panel Lab Heal" which reads as a typo.

## Verification

After PATCHing, the surface's GET response should show:

```json
{
  "surface": {
    "type": "core_page",
    "source_url": "https://<app>/hint/core_page",
    "core_page_icon_url": "https://...storage.../core_page_icon.svg",
    "core_page_icon_label": "Panel Labs"
  }
}
```

If `core_page_icon_url` is still `null` after the PATCH lands, double-check:

1. The data URI is well-formed (`data:image/svg+xml;base64,` prefix, no embedded newlines in the base64 chunk).
2. The surface's `type` is `core_page` — the icon fields are no-ops on `clinical_interaction` / `clinical_chart` / `settings` surfaces.
3. The glyph name in the URL exists in Material Symbols — a 404 from `raw.githubusercontent.com` produces an empty `CLEAN_SVG` and a `data:image/svg+xml;base64,` URI with no payload, which Hint rejects.
