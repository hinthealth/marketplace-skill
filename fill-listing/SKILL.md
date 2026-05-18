# /hint-marketplace-fill-listing — Populate the Marketplace Listing

Guides the partner through filling in their marketplace listing — the name, tagline, overview, feature highlights, customer quotes, categories, links, icon, and screenshots that practices see when browsing the Hint marketplace.

The skill is structured by **listing section**. Each section maps to its own public-API endpoint (one per concern, not one mega-PATCH) so the partner can update — or re-run — just the parts they want. The skill scrapes the partner's marketing site (with permission) and asks targeted questions to fill in what the scrape misses, then writes each section through on approval.

## Listing sections + status

| Section | Public API today? | Source |
|---|---|---|
| **Identity** — `name`, `summary` (≤80-char tagline), `built_by_name`, `built_by_url`, `icon` | 📋 not yet | `PATCH /partner/product` (planned) |
| **Overview** — long description (≤500 chars) + overview images | 📋 not yet | `PATCH /partner/product/overview` (planned) |
| **Highlights** — 3-5 feature cards (title + description + image) | 📋 not yet | `POST /partner/product/highlights` (planned) |
| **Quotes** — up to 4 customer testimonials (body + author + image) | 📋 not yet | `POST /partner/product/quotes` (planned) |
| **Categories** — 1-3 from Hint's category catalog | 📋 not yet | `PATCH /partner/product/categories` (planned) |
| **Links** — up to 6 supporting URLs (docs, blog, etc.) | 📋 not yet | `POST /partner/product/links` (planned) |
| **Pricing** — `pricing.description` + tier name | 📋 not yet | `PATCH /partner/product/pricing` (planned) |
| **Requirements** — prerequisites text | 📋 not yet | `PATCH /partner/product/requirements` (planned) |

> Today, every section needs a Hint admin to populate it through the Partner Portal. The skill is structured against the planned public surface so adoption is zero-cost once each endpoint lands — fill in the implementation per section as `[planned]` flips to `[available]` in the table above.
>
> Track the public-API rollout in [`partner-app-deployment.md`](https://github.com/hinthealth/hint-api) under "Follow-up PRs (planned)" — each section corresponds to a separate small PR.

## Why per-section, not one mega-PATCH

- Partners refresh different sections at different cadences (logo monthly, quotes when a new case study lands, pricing when tiers change).
- Per-section endpoints make permission scoping + audit logging cleaner — a partner can grant a 3rd-party tool "highlights write only".
- An LLM-generated draft might be 90% right; the partner edits one section and re-applies it without overwriting the others.
- The admin-side `Partner::Product::ReplaceListing` is one big writer because the hinter UI saves the entire LLM output at once. The partner-side workflow is incremental.

## Required reading

- https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/api-conventions.md — hosts, auth

## Platform URLs

- **Hint API**: `https://api.hint.com` — accepts both sandbox (`sbx-`) and live keys; the key determines the environment.
- **Partner Portal**: `https://app.hint.com`

Set `$HINT_API_URL=https://api.hint.com` for both sandbox and live work; the API key prefix determines the environment.

## Step 1: Gather Inputs

Ask the partner:

1. **API key** — sandbox (`sbx-...`) or live.
2. **Which sections do you want to fill?** Show the table above. Default to "all sections that are unset". The partner can opt-in/out per section.
3. **(Optional) Marketing site URL.** If supplied, the skill fetches the page and extracts candidate content for each section (logo, summary, highlights, quotes, links).
4. **(Optional) Existing assets** — local file paths for any images the partner wants to upload directly (icon, overview images, highlight thumbnails, quote headshots).

Verify the API key + read current state:

```bash
curl -s "$HINT_API_URL/api/partner/partner" -H "Authorization: Bearer $API_KEY"
curl -s "$HINT_API_URL/api/partner/product" -H "Authorization: Bearer $API_KEY"   # [planned]
```

If `product.type` is not `app`, stop and tell the partner to contact [devsupport@hint.com](mailto:devsupport@hint.com).

## Step 2: Scrape + Q&A — per section

For each section the partner opted into, gather inputs in this order: existing state → marketing-site scrape → developer Q&A for whatever's still missing. Skip a section cleanly if everything's already populated.

### 2.1 Identity (`name`, `summary`, `built_by_*`, `icon`)

- **Existing**: pull `product.name`, `product.summary`, `product.built_by_name`, `product.built_by_url`, `product.icon_url` from `GET /partner/product`.
- **Scrape**: page `<title>` → candidate `name`. First `<h1>` or hero copy → candidate `summary` (strip the company/product name; the marketplace card already shows it separately). Page `<img class*="logo">` or `/logo.*` → candidate `icon`. Page domain → candidate `built_by_url`.
- **Q&A** for whatever the scrape didn't fill: ask the partner directly. Cap `summary` at 80 chars. Confirm icon file format is PNG / JPEG / SVG / GIF, ≤5MB.

### 2.2 Overview (`description`, `overview_images[]`)

- **Existing**: `product.overview.description`, `product.overview.images[]`.
- **Scrape**: first two paragraphs of the marketing page body after the hero → candidate `description` (cap at 500 chars, 2-3 sentences). All `<img>` tags with non-decorative size in the hero / "features" section → candidate `overview_images` (max 4, surface URLs for the partner to confirm before download).
- **Q&A**: trim or replace any candidate the partner doesn't approve. Don't auto-download images the partner hasn't OK'd — show URLs first.

### 2.3 Highlights (3-5 entries; each `title`, `description`, `image`)

- **Existing**: `product.highlights[]`.
- **Scrape**: `<h2>+<p>` pairs in the marketing page's features section → candidates. Cap title at 5 words, description at 140 chars. Look for an accompanying `<img>` next to each pair → candidate `image`.
- **Q&A**: confirm or rewrite each title/description. Ideal count is 3-5 — if the scrape yields more, ask the partner to pick.

### 2.4 Quotes (up to 4 testimonials)

- **Existing**: `product.quotes[]`.
- **Scrape**: `<blockquote>` tags, `<div class*="testimonial">`, or `<q>` elements → candidates. Strip surrounding quote marks. Each needs a `body`, `author_name`, `author_title` (format: "Role, Company"), and optional `author_image`. **Never invent quotes** — if the marketing page has none, return empty and ask if the partner wants to manually add any.
- **Q&A**: confirm each quote verbatim. Don't paraphrase.

### 2.5 Categories (1-3 entries)

- **Existing**: `product.categories[]`.
- **Available catalog**: `GET /partner/product_categories` (planned). The skill must reuse one of the canonical category names; inventing categories fragments the marketplace browse experience.
- **Q&A**: show the catalog, let the partner pick 1-3. Default to a single category if obvious from the marketing page (look for keywords matching catalog entries).

### 2.6 Links (up to 6 supporting URLs)

- **Existing**: `product.links[]`.
- **Scrape**: header / footer nav links matching patterns like `docs`, `help`, `pricing`, `blog`, `case studies`, `terms`. Filter out social media + same-page anchors. Each link has `name` + `url` + `link_type` (planned enum: `docs`, `blog`, `pricing`, `support`, `legal`, `other`).
- **Q&A**: confirm or trim. Marketplace renders these as a row of pill-shaped links — keep them tight.

### 2.7 Pricing (description + tier)

- **Existing**: `product.pricing.description`, `product.pricing.tier`.
- The LLM listing generator deliberately doesn't fill this — pricing is too high-stakes to draft automatically. **Always Q&A**, never scrape.
- Ask: "What pricing model? Free, per-practice subscription, per-user, usage-based?" Cap the description at one paragraph.

### 2.8 Requirements (prerequisites text)

- **Existing**: `product.requirements`.
- **Q&A**: "Are there any prerequisites a practice needs before installing? (e.g. existing Stripe account, specific EHR data, paid Hint tier)." Free-text, plain language.

## Step 3: Show the Diff — per section

For each section the partner filled in, show a per-section diff block. Don't bundle all sections into one giant review — partners scan section-by-section, and grouping reflects how they'll edit it later.

```
Identity (1 change):
  summary:    ""  →  "Practice messaging that lives where your team already works"

Overview (2 changes):
  description:        ""           →  "Acme Health Connect adds secure two-way messaging…"
  overview_images:    []           →  3 images from acme.com (review URLs: …)

Highlights (3 changes):
  + Highlight 1:  "Two-way SMS"        — "Reach members on their phones, replies route back to the chart"
  + Highlight 2:  "HIPAA-compliant"    — "End-to-end encrypted, BAA-covered"
  + Highlight 3:  "Audit log built in" — "Every message is logged with the staffer who sent it"

Quotes (1 change):
  + Quote: "Acme has cut our after-hours call volume in half." — Sarah Chen, Office Manager, Mesa Family Practice

Categories (1 change):
  categories: []  →  ["Communication", "Patient Experience"]

Links (0 changes)
Pricing (1 change):
  description:  ""  →  "Free during sandbox testing. $50/practice/month after launch."

Requirements (skipped — no prerequisites)
```

Ask: **"Apply each section now, or skip / tweak any?"** The partner can apply some now and refine others later.

## Step 4: Apply — one call per section

For each approved section, hit the dedicated endpoint. **Check the response per call** — if one section fails, the others can still apply independently (per-section is the whole point).

```bash
# Identity — PATCH /partner/product (planned)
curl -sS -w "\nHTTP %{http_code}\n" -X PATCH "$HINT_API_URL/api/partner/product" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"product": {"name": "Acme Health Connect", "summary": "…", "icon": "data:image/png;base64,…"}}'

# Overview — PATCH /partner/product/overview (planned)
curl -sS -w "\nHTTP %{http_code}\n" -X PATCH "$HINT_API_URL/api/partner/product/overview" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"overview": {"description": "…", "images": [{"url": "data:image/png;base64,…", "alt": "…"}]}}'

# Highlights — POST /partner/product/highlights per row (planned)
for HL in $HIGHLIGHTS; do
  curl -sS -w "\nHTTP %{http_code}\n" -X POST "$HINT_API_URL/api/partner/product/highlights" \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg t "$TITLE" --arg d "$DESC" '{highlight: {title: $t, description: $d}}')"
done

# Quotes, categories, links, pricing, requirements — analogous per-section calls.
```

For image uploads (icon, overview images, highlight images, quote headshots): convert to base64 data URI client-side before sending. The platform's `ActiveStorageDataUri` initializer decodes data URIs into attachments automatically.

```bash
DATA_URI="data:image/png;base64,$(base64 -i path/to/asset.png | tr -d '\n')"
```

## Step 5: Verify

Re-fetch `GET /partner/product` (and `/overview`, `/highlights`, etc.) and confirm each section landed. Print a per-section confirmation:

```
Listing applied:
  ✓ Identity     — name, summary, built_by_name, icon updated
  ✓ Overview     — description + 3 images uploaded
  ✓ Highlights   — 3 entries created
  ✓ Quotes       — 1 entry created
  ✓ Categories   — 2 categories set
  - Links        — skipped (no changes)
  ✓ Pricing      — description set
  - Requirements — skipped (no prerequisites)

Preview your listing: https://app.hint.com/partner/dashboard
```

## Troubleshooting

- **Most sections return 404** — the per-section public endpoints aren't all live yet. Check the status table at the top of this file. Until each endpoint ships, the partner must ask [devsupport@hint.com](mailto:devsupport@hint.com) to populate that section.
- **Image upload 422** — file is >5MB or wrong MIME. Resize or re-encode. SVG is fine for icons; prefer JPEG / PNG for photos.
- **Category not found** — the partner picked a category name that isn't in Hint's canonical catalog. Re-show the catalog and have them pick again (or request a new category from [devsupport@hint.com](mailto:devsupport@hint.com)).
- **Marketing-site scrape returns 401 / 403** — the site is gated. Fall back to asking the partner directly for the copy.

For anything else, contact [devsupport@hint.com](mailto:devsupport@hint.com).
