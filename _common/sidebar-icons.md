# Sidebar icons for Core Page apps (shared fragment)

**Read this when an app registers a `core_page` anchor.** The Core Page icon shows up in every practice's left sidebar — right next to Patients / Employers / Reports / Admin — and it's what the practitioner sees every day. Getting it right is high-leverage UX; getting it wrong (or skipping it, so Hint falls back to the generic placeholder) leaves a permanent "looks unfinished" stain on the install.

## The sidebar icon is NOT the listing icon

Two distinct assets, easy to conflate:

| Field | Where it shows | Visual style |
|---|---|---|
| `partner_product.icon` / `icon_url` | The marketplace listing tile (`/apps/<slug>` and the browse grid) | **Filled brand mark** — square card, your brand color, your glyph, your typography. A logo. |
| `app.anchors[core_page].core_page_icon` / `core_page_icon_url` | The practice's left nav sidebar | **Outlined Material Symbols glyph** — single-color (Hint tints it for hover / selected states), no background, no fill, matches Hint's nav style |

If you reuse the listing icon for the sidebar, the result is a saturated brand-color block that fights every other nav item visually. The full guideline is at <https://developers.hint.com/docs/partner-asset-guidelines> — bullet 1 (listing) vs bullet 3 (sidebar).

## Picking the glyph

Material Symbols (Google's open-source icon set, https://fonts.google.com/icons) is the recommended source — it's what Hint's own nav uses, so a sidebar icon picked from there drops in cleanly with no styling adjustments.

**Domain → glyph mapping.** Use the partner's app description (gathered in `create-app` Step 1) to pick the right glyph from `_common/sidebar-icons/`:

| App domain | Primary | Alternate |
|---|---|---|
| Labs / clinical data / diagnostics | `science.svg` | `biotech.svg` |
| Billing / revenue / payments | `receipt_long.svg` | `payments.svg` |
| Messaging / outreach / patient comms | `forum.svg` | `mail.svg` |
| Scheduling / appointments | `calendar_month.svg` | `schedule.svg` |
| Members / patients / panel | `groups.svg` | `person.svg` |
| Reports / analytics / dashboards | `bar_chart.svg` | `monitoring.svg` |
| Settings / configuration | `settings.svg` | `tune.svg` |

If the app's domain isn't in the table, browse https://fonts.google.com/icons for an Outlined-style glyph at weight 400 (normal grade), download it, and apply the cleanup rule below before using it.

## Cleanup rule for custom glyphs

Material Symbols' export ships SVGs with `height="24"` and `width="24"` attributes hard-coded. Hint's sidebar sizes the icon via CSS; the explicit attributes fight that. Strip them:

```
- Keep:   xmlns, viewBox, <path> data
- Remove: height, width, fill (let it inherit currentColor)
```

In one line:

```bash
sed -E 's/ (height|width|fill)="[^"]*"//g' downloaded.svg > clean.svg
```

The shipped SVGs under `_common/sidebar-icons/` are already cleaned and paste-ready.

## Encoding for the PATCH

The `core_page_icon` field on the anchor takes a `data:image/svg+xml;base64,...` URI (max 5MB, but SVGs are kilobytes so this is irrelevant in practice). To encode:

```bash
ICON_DATA_URI="data:image/svg+xml;base64,$(base64 < _common/sidebar-icons/science.svg | tr -d '\n')"
```

Then attach it to the Core Page anchor:

```bash
curl -s -X PATCH "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/anchors/$ANCHOR_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {
    \"core_page_icon\": \"$ICON_DATA_URI\",
    \"core_page_icon_label\": \"$LABEL\"
  }}"
```

Confirm by `GET`-ing the anchor — `core_page_icon_url` (read-side field name; asymmetric with the PATCH-side `core_page_icon`) should now be a populated URL instead of `null`.

## Picking the label

`core_page_icon_label` is the sidebar text shown next to the icon. It's a tooltip on hover on desktop and visible text on mobile sidebars. Keep it short:

- **Default:** the app's name, truncated to ≤14 characters.
- **If the name is longer than 14 characters:** ask the partner for a short form. "Panel Lab Health Snapshots" → "Panel Labs"; "Outreach & Messaging Hub" → "Outreach". Don't auto-truncate mid-word — it produces "Panel Lab Heal" which reads as a typo.

## Verification

After PATCHing, the anchor's GET response should show:

```json
{
  "anchor": {
    "type": "core_page",
    "source_url": "https://<app>/hint/core_page",
    "core_page_icon_url": "https://...storage.../core_page_icon.svg",
    "core_page_icon_label": "Panel Labs"
  }
}
```

If `core_page_icon_url` is still `null` after the PATCH lands, double-check the data URI is well-formed (`data:image/svg+xml;base64,` prefix, no embedded newlines in the base64 chunk) and that the anchor's `type` is `core_page` — the icon fields are no-ops on `clinical_interaction` / `clinical_chart` / `settings` anchors.
