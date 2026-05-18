# /hint-marketplace-fill-listing — Populate the Marketplace Listing

Generates a complete marketplace listing for the partner's app — the copy + categorization + visuals that practices see when browsing the Hint marketplace — and applies it via the public API after the partner reviews the draft.

The skill is a **draft + review** workflow: it gathers all inputs first, generates a coherent draft of the entire listing, shows it to the partner for approval, then PATCHes everything through in one go. No surprise mutations.

## When to use this

- The partner finished building or retrofitting their app and now needs a presentable listing before opening a release.
- The partner wants to refresh an existing listing (better copy, new screenshots, updated category, etc.).

## Required reading

1. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/api-conventions.md — hosts, auth

## Platform URLs

- **Hint API (sandbox)**: `https://api.sandbox.hint.com` — for `sbx-` API keys
- **Hint API (live)**: `https://api.hint.com` — for live API keys
- **Partner Portal**: `https://app.hint.com`

Set `$HINT_API_URL` based on the API key prefix.

## What the Public API supports today

This skill writes through the public Partner API. The fields it can set are:

| Surface | Fields | Endpoint |
|---|---|---|
| Partner profile | `name`, `email`, `redirect_url`, `auth_type` | `PATCH /partner/partner` |
| App config | `handshake_url`, `default_admin_role`, `default_non_admin_role`, `prepopulate_role_mappings`, `partner_roles`, `partner_role_names` | `PATCH /partner/app` |
| Anchor config | `source_url`, `core_page_icon` (base64 data URI), `core_page_icon_label`, `core_page_auto_adjust_height`, `interaction_title`, `interaction_description`, `interaction_type`, `settings_label` | `POST` / `PATCH /partner/app/anchors[/:id]` |
| Service config | `name`, `build_command`, `start_command`, `env_vars` (custom) | `POST` / `PATCH /partner/app/services[/:id]` |

**Out of scope today** (no public API yet — partner must do these manually in the Partner Portal): product description / tagline, product category, product images / screenshots, pricing tier, requirements / prerequisites text, support contact. Track at https://github.com/hinthealth/marketplace-skill/issues — when the product-overview public API ships, this skill will extend automatically.

## Step 1: Gather Inputs

Ask the partner four things:

1. **API key** — sandbox (`sbx-...`) or live. Determines `$HINT_API_URL`.
2. **What does the app do?** A 1-2 sentence elevator pitch. The skill uses this to draft `partner.name` and anchor labels if they're empty.
3. **(Optional) Marketing site URL.** If the partner has an existing marketing page for the app, the skill fetches it and extracts product positioning to reuse. Examples: `https://acme.com/products/hint-integration`, `https://acmemd.io`.
4. **(Optional) Per-anchor icon.** For each `core_page` anchor, the partner can provide an icon image (URL or local file). The skill base64-encodes and uploads via `POST /partner/app/anchors`.

Verify the API key + read current state:

```bash
curl -s "$HINT_API_URL/api/partner/partner" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/app" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/app/anchors" -H "Authorization: Bearer $API_KEY"
```

If `product.type` is not `app`, stop and tell the partner to contact [devsupport@hint.com](mailto:devsupport@hint.com).

## Step 2: Scrape the Marketing Site (Optional)

If the partner supplied a marketing URL, fetch the HTML and extract:

- **Page title** → candidate for `partner.name` (strip suffixes like "| Acme" / "- Home").
- **First H1** → primary value proposition.
- **First paragraph or hero copy** → tagline / one-line description.
- **Feature list (any `<ul>` or `<h2>+<p>` patterns)** → bullet points the skill can summarize.
- **Logo `<img>` (look for `class*="logo"`, `alt="<brand>"`, or `/logo.` in src)** → candidate icon. Don't auto-download; surface the URL and ask the partner if they want to use it.
- **Email / mailto links** → candidate for `partner.email` if not set.

Always show the partner what was scraped before using it. If the marketing site is gated (login required), the fetch fails — fall back to asking the partner directly.

## Step 3: Run the Developer Q&A

For fields the marketing site can't fill in (or that the partner hasn't provided), ask succinct, scoped questions. Skip the question if the field is already populated and the partner hasn't asked for a refresh.

| Field | Question |
|---|---|
| `partner.name` | "What's the public-facing name of your app as you'd want practices to see it?" (default to scraped page title) |
| `partner.email` | "What email should practices contact for support?" (default to scraped mailto:) |
| `partner.redirect_url` | "Where should Hint redirect practices after they complete activation?" (default to `$APP_URL/hint/connect/` — fetch from `/partner/app/services` if not provided) |
| `app.handshake_url` | "Confirm the handshake URL: this is where Hint POSTs the install handshake. Should be `$APP_URL/hint/handshake`." (default; ask only if not set) |
| Anchor labels | For each registered `core_page` / `settings` anchor: "What label should appear in the sidebar / settings tab?" (defaults to app name) |
| Anchor icon | For each `core_page` anchor: "Want to upload an icon? It'll appear next to the label in the sidebar. (file path, URL, or skip)" |

Don't ask more than ~5 questions in a single round. If you need more, batch and confirm.

## Step 4: Draft the Listing

Compose the full listing in a single review block. Example:

```
Hint Marketplace Listing — Draft (review before applying)

Partner profile:
  name:           Acme Health Connect
  email:          dev@acme.com
  redirect_url:   https://acme-marketplace.acme.com/hint/connect/
  auth_type:      automatic_headless

App config:
  handshake_url:  https://acme-marketplace.acme.com/hint/handshake
  default_admin_role:     admin
  default_non_admin_role: clinician

Anchors:
  core_page (panch-AbCdEf123):
    source_url:                    https://acme-marketplace.acme.com/hint/core_page
    core_page_icon:                <will upload from /Users/.../acme-logo.png>
    core_page_icon_label:          Acme Health
    core_page_auto_adjust_height:  true

  clinical_interaction (panch-XyZ789):
    source_url:                    https://acme-marketplace.acme.com/hint/clinical_interaction
    interaction_title:             Sync to Acme
    interaction_type:              acme_sync

Out of scope (set manually in Partner Portal):
  - Product description / tagline (Public API does not expose product PATCH yet)
  - Product category
  - Marketplace screenshots
  - Pricing tier
```

Ask the partner: **"Apply this listing now, or want to tweak something first?"** If they ask to tweak, loop back to Step 3 for the relevant field.

## Step 5: Apply via the Public API

Once approved, write everything through with sequenced PATCH/POST calls. **Each call's response should be checked** — if any fail, stop and report which (don't keep applying after a failure).

```bash
# Partner profile
curl -s -X PATCH "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "$PARTNER_PATCH_BODY"

# App config
curl -s -X PATCH "$HINT_API_URL/api/partner/app" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "$APP_PATCH_BODY"

# Anchor updates (one PATCH per anchor)
for ANCHOR_ID in $UPDATED_ANCHOR_IDS; do
  curl -s -X PATCH "$HINT_API_URL/api/partner/app/anchors/$ANCHOR_ID" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "$ANCHOR_BODY"
done
```

For `core_page_icon` uploads: convert the file to a base64 data URI client-side before sending:

```bash
ICON_DATA_URI="data:image/png;base64,$(base64 -i path/to/icon.png | tr -d '\n')"
curl -s -X PATCH "$HINT_API_URL/api/partner/app/anchors/$ANCHOR_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"anchor\": {\"core_page_icon\": \"$ICON_DATA_URI\"}}"
```

The platform decodes the data URI into an attachment automatically. PNG / JPEG / SVG all work; max 5MB.

## Step 6: Verify & Report

Re-fetch the partner/app/anchors state and confirm each field was applied:

```bash
curl -s "$HINT_API_URL/api/partner/partner" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/app" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/app/anchors" -H "Authorization: Bearer $API_KEY"
```

Print a diff:

```
Listing applied!

  Updated:
    partner.name:           "" → "Acme Health Connect"
    partner.email:          "" → "dev@acme.com"
    partner.redirect_url:   "" → "https://acme-marketplace.acme.com/hint/connect/"
    app.handshake_url:      "" → "https://acme-marketplace.acme.com/hint/handshake"
    anchor[core_page].core_page_icon_label: "" → "Acme Health"
    anchor[core_page].core_page_icon:        (uploaded acme-logo.png, 14KB)
    anchor[clinical_interaction].interaction_title: "" → "Sync to Acme"

  Still to do manually in the Partner Portal (https://app.hint.com):
    - Product description / tagline
    - Product category selection
    - Marketplace screenshots
    - Pricing tier

  Preview your listing: https://app.hint.com/partner/dashboard
```

## Troubleshooting

- **`product.type must be app` error** — the partner's product type isn't set to `app`. This is admin-only; ask the partner to contact [devsupport@hint.com](mailto:devsupport@hint.com) before retrying.
- **Anchor icon upload fails with 422** — the icon is larger than 5MB or isn't an image MIME type. Shrink it or try a different format.
- **PATCH /partner/partner returns 422 with role validation errors** — the partner's role config is inconsistent (e.g. `default_admin_role` doesn't match any key in `partner_roles`). Out of scope for this skill; ask the partner to fix in the Partner Portal under Activation Settings, then re-run.
- **Marketing-site fetch returns 401 / 403** — the site is gated. Fall back to asking the partner directly for the copy.

For anything else, contact [devsupport@hint.com](mailto:devsupport@hint.com).
