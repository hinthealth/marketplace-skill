# /hint-marketplace-fill-listing — Generate the Marketplace Listing

**Status:** 📋 not yet implemented. This skill is on the roadmap. Track progress at https://github.com/hinthealth/marketplace-skill/issues or contact [devsupport@hint.com](mailto:devsupport@hint.com).

## What this skill will do

Generate a complete marketplace listing for the partner's app — the copy, categorization, and screenshots that practices see when browsing the Hint marketplace — and apply it via the public API. The skill takes three kinds of input and synthesizes them into a polished listing:

1. **Existing API state** — fetched via `GET /partner/partner` and `GET /partner/app`. Anything already set (name, logo, description) is the starting point; the skill won't overwrite without confirmation.
2. **Developer Q&A** — guided prompts the skill asks the partner: target audience, top three use cases, key feature list, pricing model, integration footprint, etc.
3. **Marketing-site URLs** — the skill fetches the partner's existing marketing pages (with permission), extracts product positioning + feature copy, and reuses it where appropriate so the marketplace listing is consistent with what practices see elsewhere.

Output (planned):

- `PATCH /partner/partner` — `name`, `email`, `redirect_url` (if not set), `description` (product overview)
- `PATCH /partner/product` — `name`, `type=app`, category assignments, primary value-prop copy
- `product_overview.description` + `product_overview.images[]` — partner can supply image URLs or the skill can scrape from the marketing site
- (Optional) `core_page_icon` — base64-encoded data URI uploaded via `POST /partner/app/anchors` if the partner provides one

The skill is a "draft + review" workflow: it generates the full listing, shows it to the partner for review, then writes through to the API on confirmation. No surprise PATCHes.

## Until this skill ships

Use the public API directly:

```bash
curl -X PATCH "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"partner": {"name": "...", "email": "...", "redirect_url": "..."}}'
```

Or fill the listing through the Partner Portal at https://app.hint.com.
