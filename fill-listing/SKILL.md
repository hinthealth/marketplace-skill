# /hint-marketplace-fill-listing — Populate the Marketplace Listing

Guides the partner through filling in their marketplace listing — the name, tagline, overview, feature highlights, customer quotes, categories, links, icon, screenshots, and pre-install preconditions that practices see when browsing the Hint marketplace.

The skill is structured by **listing section**. Each section maps to its own public-API endpoint (one per concern, not one mega-PATCH) so the partner can update — or re-run — just the parts they want. The skill scrapes the partner's marketing site (with permission) and asks targeted questions to fill in what the scrape misses, then writes each section through on approval.

## Listing sections

| Section | Endpoint | Notes |
|---|---|---|
| **Identity** — `name`, `slug`, `type`, `summary`, `built_by_name`, `built_by_url`, `icon` | `PATCH /partner/partner_products/:id` | `slug` + `type` are create-only on the public API — Hint support changes them later. |
| **Overview** — long description + overview images | `GET/PATCH /partner/partner_products/:id/overview` | Returns an empty in-memory overview if none exists; PATCH wraps id-keyed image reconciliation (edit-by-id, create-new, delete-omitted). |
| **Highlights** — 3–5 feature cards (`title`, `description`, `image`, `position`) | `POST/PATCH/DELETE /partner/partner_products/:id/highlights[/:hid]` | `position` is integer; defaults to next slot via `acts_as_list`. |
| **Quotes** — testimonials (`text`, `author`, `author_title`, `author_image`, `position`) | `POST/PATCH/DELETE /partner/partner_products/:id/quotes[/:qid]` | Field names are `text` (NOT `body`) and `author` (NOT `author_name`). |
| **Categories** — 1–3 from Hint's catalog (`name`) | `POST /partner/partner_products/:id/categories` + `PATCH /partner/partner_products/:id/categories/:cid` + `DELETE /partner/partner_products/:id/categories/:cid` + `GET /partner/product_categories` (flat catalog) | POST creates the category-by-name if new and attaches; PATCH attaches an existing category by id. |
| **Links** — supporting URLs (`link_type`, `value`, `title`, `utm_campaign`, `utm_content`, `position`) | `POST/PATCH/DELETE /partner/partner_products/:id/links[/:lid]` | `link_type` enum is `url \| phone \| email` (NOT free-text labels). `product_cta` is reserved and managed via the product Identity endpoint — not partner-settable here. |
| **Preconditions** — pre-install steps (`type`, `url`) | `POST/PATCH/DELETE /partner/partner_products/:id/preconditions[/:pid]` | `type` enum is `external_account \| external_install \| onboarding_call`. |

Two sections are **intentionally not partner-settable** via the public API. Both are hinter-curated decisions on Hint's side:

| Section | How to change it |
|---|---|
| **Pricing** — `description` + tier name | Email [devsupport@hint.com](mailto:devsupport@hint.com). |
| **Requirements** — install gate (`type` = `feature`/`tier`, `feature`, `starting_tier`) | Email [devsupport@hint.com](mailto:devsupport@hint.com). |

## Why per-section, not one mega-PATCH

- Partners refresh different sections at different cadences (logo monthly, quotes when a new case study lands).
- Per-section endpoints make permission scoping + audit logging cleaner — a partner can grant a 3rd-party tool "highlights write only".
- An LLM-generated draft might be 90% right; the partner edits one section and re-applies it without overwriting the others.
- The admin-side `Partner::Product::ReplaceListing` is one big writer because the hinter UI saves the entire LLM output at once. The partner-side workflow is incremental.

## Required reading

- [`_common/api-conventions.md`](../_common/api-conventions.md) — hosts, auth, response shapes.

## Platform URLs

Set `$HINT_API_URL=https://api.hint.com` (sandbox + live share the host; the api key prefix tells them apart). Partner Portal at `https://app.hint.com`. Full conventions in [`_common/api-conventions.md`](../_common/api-conventions.md).

## Step 1: Gather Inputs

Ask the partner:

1. **API key** — sandbox (`sbx-...`) or live.
2. **Which product?** Partners can own multiple products. List them first and confirm:
   ```bash
   curl -s "$HINT_API_URL/api/partner/partner_products" -H "Authorization: Bearer $API_KEY"
   ```
   Returns a JSON array; each element has `id` (a public id like `pp-xxxx`), `name`, `type`. Filter to `type == "app"` — listing fields below apply to app-type products. Save the product's `id` as `$PRODUCT_ID`.
3. **Which sections do you want to fill?** Show the table above. Default to "all sections that are unset". The partner can opt-in/out per section.
4. **(Optional) Marketing site URL.** If supplied, the skill fetches the page and extracts candidate content for each section (logo, summary, highlights, quotes, links).
5. **(Optional) Existing assets** — local file paths for any images the partner wants to upload directly (icon, overview images, highlight thumbnails, quote headshots).

Read current state for the chosen product:

```bash
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/overview" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/highlights" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/quotes" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/categories" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/links" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/preconditions" -H "Authorization: Bearer $API_KEY"
```

## Step 2: Scrape + Q&A — per section

For each section the partner opted into, gather inputs in this order: existing state → marketing-site scrape → developer Q&A for whatever's still missing. Skip a section cleanly if everything's already populated.

### 2.1 Identity (`name`, `summary`, `built_by_*`, `icon`)

- **Existing**: `name`, `summary`, `built_by_name`, `built_by_url`, `icon_url` from `GET /partner/partner_products/:id`.
- **Scrape**: page `<title>` → candidate `name`. First `<h1>` or hero copy → candidate `summary` (strip the company/product name; the marketplace card already shows it separately). Page `<img class*="logo">` or `/logo.*` → candidate `icon`. Page domain → candidate `built_by_url`.
- **Q&A**: ask the partner directly for whatever the scrape didn't fill. **Cap `summary` at ≤9 words / ≤60 chars** — marketplace cards render summary in a single line under the app name, and anything longer truncates with an ellipsis on the listing page. A 9-word cap forces the partner to pick the one thing the app does best ("Panel-wide lab health monitoring"), not a paragraph ("A dashboard for managing patient lab results, member trends, and overdue follow-ups"). Apply the cap **both at draft time** (when generating from the marketing-site scrape) and **before the PATCH** — if the draft runs long, ask the partner to pick a tighter phrasing rather than truncating silently. Confirm icon format is PNG / JPEG / SVG / GIF, ≤5MB.
- **Skip `slug` and `type`**: both are create-only on the public API. If they're wrong, the partner emails devsupport@hint.com.

### 2.2 Overview (`description`, `images[]`)

- **Existing**: `description` + `images[]` from `GET /partner/partner_products/:id/overview`. The endpoint returns an empty (in-memory) overview when none exists — no need to special-case 404.
- **Scrape**: first two paragraphs of the marketing page body after the hero → candidate `description` (cap at 500 chars, 2-3 sentences). All non-decorative `<img>` tags in the hero / "features" section → candidate `images` (max 4, surface URLs for the partner to confirm before download).
- **Q&A**: trim or replace any candidate the partner doesn't approve. Don't auto-download images the partner hasn't OK'd — show URLs first.
- **Images are id-keyed**. Existing images have an `id` you can send back to update in place, omit to delete, or leave off for new uploads. Don't destroy-and-rebuild — pass through ids you want to keep.

### 2.3 Highlights (3–5 entries; each `title`, `description`, `image`, `position`)

- **Existing**: `GET /partner/partner_products/:id/highlights` returns the array.
- **Scrape**: `<h2>+<p>` pairs in the marketing page's features section → candidates. Cap title at ~5 words, description at ~140 chars. Look for an `<img>` next to each pair → candidate `image`.
- **Q&A**: confirm or rewrite each title/description. Ideal count is 3–5 — if the scrape yields more, ask the partner to pick.
- `position` is an integer; if omitted, it appends to the end via `acts_as_list`.

### 2.4 Quotes (testimonials; each `text`, `author`, `author_title`, `author_image`, `position`)

- **Existing**: `GET /partner/partner_products/:id/quotes`.
- **Field names** (easy to get wrong): the body field is `text`, NOT `body`. The attribution field is `author`, NOT `author_name`. `author_title` is the "Role, Company" line.
- **Scrape**: `<blockquote>`, `<div class*="testimonial">`, or `<q>` elements → candidates. Strip surrounding quote marks. **Never invent quotes** — if the marketing page has none, return empty and ask if the partner wants to manually add any.
- **Q&A**: confirm each quote verbatim. Don't paraphrase.

### 2.5 Categories (1–3 entries)

- **Existing**: `GET /partner/partner_products/:id/categories`.
- **Catalog**: `GET /partner/product_categories` returns the flat list of all marketplace categories currently in use, ordered by name. Pick from this list to align with existing browse — only invent a new category name if nothing fits.
- **Attach by name (creates if new)**: `POST /partner/partner_products/:id/categories` with `{ "name": "Communication" }`. If a category with that name already exists, it gets attached; otherwise a new one is created and attached.
- **Attach an existing category by id**: `PATCH /partner/partner_products/:id/categories/:cid`.
- **Q&A**: show the catalog, let the partner pick 1–3.

### 2.6 Links (supporting URLs)

- **Existing**: `GET /partner/partner_products/:id/links`.
- **Field names**: `link_type` (enum: `url` / `phone` / `email`), `value` (the URL / phone number / email address), `title` (display label), optional `utm_campaign` (defaults to `hint_marketplace`), optional `utm_content`, optional `position`.
- **`product_cta` is reserved**. It's the in-app install CTA managed via the main product endpoint — not partner-settable here.
- **Scrape**: header / footer nav links matching `docs`, `help`, `pricing`, `blog`, `case studies`. Filter out social media + same-page anchors. For email/phone links, decide between `link_type: email`/`phone` based on the `mailto:` / `tel:` scheme.
- **Q&A**: confirm or trim. Marketplace renders these as a row of pill-shaped links — keep them tight.

### 2.7 Preconditions (pre-install steps)

- **Existing**: `GET /partner/partner_products/:id/preconditions`.
- **Field names**: `type` (enum: `external_account` / `external_install` / `onboarding_call`), `url` (the destination — signup page, install link, or scheduling page depending on `type`).
- A precondition is something the practice must complete BEFORE installing — typically creating an account on the partner's side, installing a companion native app, or scheduling onboarding.
- **Q&A**: "Does a practice need to do anything outside Hint before installing your app? (sign up on your site, install a desktop app, book a call?)" Map the answer to one of the three `type` values.

## Step 3: Show the Diff — per section

For each section the partner filled in, show a per-section diff block. Don't bundle all sections into one giant review — partners scan section-by-section, and grouping reflects how they'll edit it later.

```
Identity (1 change):
  summary:    ""  →  "Practice messaging that lives where your team already works"

Overview (2 changes):
  description:  ""           →  "Acme Health Connect adds secure two-way messaging…"
  images:       []           →  3 images from acme.com (review URLs: …)

Highlights (3 new):
  + "Two-way SMS"        — "Reach members on their phones, replies route back to the chart"
  + "HIPAA-compliant"    — "End-to-end encrypted, BAA-covered"
  + "Audit log built in" — "Every message is logged with the staffer who sent it"

Quotes (1 new):
  + "Acme has cut our after-hours call volume in half." — Sarah Chen, Office Manager, Mesa Family Practice

Categories (2 new):
  + "Communication"
  + "Patient Experience"

Links (0 changes)

Preconditions (1 new):
  + external_account → https://acme.example.com/signup
```

Ask: **"Apply each section now, or skip / tweak any?"** The partner can apply some now and refine others later.

## Step 4: Apply — one call per section

For each approved section, hit the dedicated endpoint. **Check the response per call** — if one section fails, the others can still apply independently.

```bash
# Identity — PATCH /partner/partner_products/:id
curl -sS -w "\nHTTP %{http_code}\n" -X PATCH "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Acme Health Connect", "summary": "…", "built_by_name": "Acme", "built_by_url": "https://acme.example.com", "icon": "data:image/png;base64,…"}'

# Overview — PATCH /partner/partner_products/:id/overview
curl -sS -w "\nHTTP %{http_code}\n" -X PATCH "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/overview" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"description": "…", "images": [{"url": "data:image/png;base64,…", "alt": "…"}]}'

# Highlights — POST /partner/partner_products/:id/highlights per row
for HL_JSON in "$@"; do
  curl -sS -w "\nHTTP %{http_code}\n" -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/highlights" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "$HL_JSON"
  # body: {"title": "…", "description": "…", "image": "data:image/png;base64,…", "position": 1}
done

# Quotes — same shape, field names are text/author/author_title:
#   {"text": "…", "author": "Sarah Chen", "author_title": "Office Manager, Mesa Family Practice", "author_image": "data:image/png;base64,…", "position": 1}

# Categories — POST by name (creates+attaches if new), or PATCH by id (attach existing):
curl -sS -w "\nHTTP %{http_code}\n" -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/categories" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Communication"}'

# Links — note link_type enum + value (not "url"):
curl -sS -w "\nHTTP %{http_code}\n" -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/links" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"link_type": "url", "value": "https://acme.example.com/docs", "title": "Documentation", "position": 1}'

# Preconditions (this one still requires the `precondition` wrapper key):
curl -sS -w "\nHTTP %{http_code}\n" -X POST "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/preconditions" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"precondition": {"type": "external_account", "url": "https://acme.example.com/signup"}}'
```

For image uploads (icon, overview images, highlight images, quote headshots): convert to base64 data URI client-side before sending. The platform's `ActiveStorageDataUri` initializer decodes data URIs into attachments automatically.

Supported MIME types: `image/png`, `image/jpeg`, `image/svg+xml`, `image/gif`. **Prefer SVG** for icons and dashboard/feature illustrations — marketplace cards render at multiple sizes (badge, listing card, hero), SVG stays crisp at every zoom level, and the files are routinely ~10× smaller than equivalent PNGs (≈9 KB vs ≈80–150 KB for typical listing assets). Use PNG / JPEG only for photographs (quote headshots, product screenshots).

```bash
# PNG (photos / screenshots):
DATA_URI="data:image/png;base64,$(base64 -i path/to/asset.png | tr -d '\n')"

# SVG (icons, illustrations) — same shape, just the MIME:
DATA_URI="data:image/svg+xml;base64,$(base64 -i path/to/icon.svg | tr -d '\n')"
```

### Overview-image reconciliation (`images[]`)

`PATCH /partner_products/:id/overview` with an `images[]` array uses **id-keyed reconciliation**: entries with a matching `id` are edited in place, entries without an `id` are created, any existing images **not represented** in the array are deleted. Omitting the `images` key entirely leaves images unchanged.

This lets a single PATCH swap the hero image without an explicit DELETE step:

```jsonc
// Swap hero image: omit the old image's id, include a new entry with just url+alt
{ "images": [ { "url": "data:image/svg+xml;base64,...", "alt": "Dashboard hero" } ] }
```

```jsonc
// Edit existing image's alt text in place — keep its id, change the field
{ "images": [ { "id": "ovi-XXXX", "alt": "Updated alt text" } ] }
```

```jsonc
// Add a second image while keeping the existing one
{ "images": [ { "id": "ovi-XXXX" }, { "url": "data:image/svg+xml;base64,...", "alt": "Worklist screenshot" } ] }
```

The equivalent reconciliation **does not exist for highlights** — those are managed via individual `POST` / `PATCH` / `DELETE` per id.

## Step 5: Verify

Re-fetch each section's endpoint and confirm it landed. Print a per-section confirmation:

```
Listing applied:
  ✓ Identity       — name, summary, built_by_name, icon updated
  ✓ Overview       — description + 3 images uploaded
  ✓ Highlights     — 3 entries created
  ✓ Quotes         — 1 entry created
  ✓ Categories     — 2 categories attached
  - Links          — skipped (no changes)
  ✓ Preconditions  — 1 step set

Preview your listing: https://app.hint.com/partner/dashboard
```

For Pricing or Requirements changes, the partner emails [devsupport@hint.com](mailto:devsupport@hint.com).

## Troubleshooting

- **422 on slug or type during PATCH /partner/partner_products/:id** — both are create-only via the public API. Strong params silently drop them, so the call usually succeeds but the field is unchanged. To rename a product or change its type, email devsupport@hint.com.
- **404 on a category id** — the partner picked an id that doesn't exist. Re-fetch `GET /partner/product_categories` and pick again. If the desired category isn't in the catalog, POST by name instead — that creates it.
- **422 on `link_type`** — only `url`, `phone`, and `email` are partner-settable. `product_cta` is reserved.
- **Image upload 422** — file is >5MB or wrong MIME. Resize or re-encode. SVG is fine for icons; prefer JPEG / PNG for photos.
- **Marketing-site scrape returns 401 / 403** — the site is gated. Fall back to asking the partner directly for the copy.

For anything else, contact [devsupport@hint.com](mailto:devsupport@hint.com).
