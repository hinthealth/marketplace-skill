# Hint API Conventions (shared fragment)

The single source of truth for hosts, auth, response shapes, pagination, filtering, and reserved env vars. Other files in this repo link here instead of repeating these rules.

## Hosts

- **Hint API**: `https://api.hint.com` — works with both sandbox (`sbx-` prefix) and live API keys. The key determines the environment, not the host.
- **Hint API (sandbox alias)**: `https://api.sandbox.hint.com` — also exists and returns identical data for sandbox keys. No practical reason to switch; `api.hint.com` works for both so the partner doesn't have to swap hosts when promoting from sandbox to live.
- **Partner Portal**: `https://app.hint.com` — single portal for both sandbox and live; partners switch between workspaces inside it.

**Rule:** set `$HINT_API_URL=https://api.hint.com` for both sandbox and live work. The `sbx-` prefix on the API key picks the environment; the host stays the same.

## Authentication

All Partner + Provider API calls use a bearer token:

```
Authorization: Bearer <api_key_or_access_token>
```

Two distinct token types:

- **Partner API key** — partner-wide, set via `HINT_API_KEY` env var on the deployed service. Used to call `/api/partner/*`.
- **Practice access token** — practice-scoped, returned by `POST /api/oauth/tokens` during the `/hint/connect/:code` exchange. Used to call `/api/provider/*` on behalf of a specific practice. Persist this server-side keyed by `practice_id`.

Never use the partner API key for `/api/provider/*` calls — those need the practice-scoped token or they leak cross-practice data.

## List-endpoint response shape

`GET /api/partner/*` and `GET /api/provider/*` list endpoints return **bare JSON arrays**, not `{patients: [...]}` or `{data: [...]}`. Parse with `Array.isArray(res) ? res : []` and iterate directly. Do NOT do `res.patients || res.records || []` — that silently produces an empty array.

## Pagination

Use `limit` + `offset` (not `page` / `page_size`):

```
GET /api/provider/patients?limit=50&offset=100
```

Max `limit` is 100. If a response returns exactly `limit` rows, there are likely more — paginate by incrementing `offset`.

## Date filters

Timestamp fields accept bracketed operators:

```
GET /api/provider/customer_invoices?created_at[gte]=2026-01-01&created_at[lte]=2026-12-31
```

Each timestamp field (`created_at`, `updated_at`, `paid_at`, etc.) accepts `[gte]`, `[gt]`, `[lte]`, `[lt]`, `[eq]`. This is the most common filter idiom after pagination.

## Sandbox `created_at` is the seed-run timestamp

In sandbox, every record's `created_at` (patient, membership, invoice, payment) is the timestamp of the seed run that loaded the data — they're all within seconds of each other. Any chart bucketed on `created_at` will be flat-flat-flat-spike-on-seed-day for every metric. Bucket by domain dates instead: `joined_practice_date` (patients), `start_date` / `end_date` (memberships), `paid_at` / `date` (invoices, payments), `bill_date` / `next_bill_date` (membership billing). This is the correct choice in live too — patients are sometimes backdated for compliance, memberships start mid-period, etc.

## Archived rows are excluded by default

List endpoints filter out archived records by default. Inverse queries:

- `?filter=archived` — only archived
- `?filter=all` — both active and archived

If a partner app shows "no records" and the practice expects to see some, archive state is the first thing to check.

## Reserved env vars

Hint sets these automatically on every Hosted-Mode deploy. Self-Hosted Mode apps set them themselves. Partner-supplied values for these keys are ignored on Hosted Mode.

| Var | Purpose |
|---|---|
| `HINT_API_URL` | Base URL of the Hint API. Use `https://api.hint.com` for both sandbox and live. |
| `HINT_API_KEY` | Partner-wide API key for `/api/partner/*` calls. NOT used for `/api/provider/*` (those need the practice-scoped access token from `/hint/connect/:code`). |
| `HINT_PARTNER_ID` | Stable partner ident (e.g. `ptr-...` / `sbx-ptr-...`). Useful for log scoping. |
| `HINT_WEBHOOK_SECRET` | Used to verify the `X-Hint-Signature` header on `POST /hint/handshake`. The partner finds this in the Partner Portal under **Webhook Settings → Webhooks Signature Key** (the key belongs to the selected backend; most partners have one). |
| `DATABASE_URL` | Postgres connection string (only present when the auto-provisioned sibling database is connectable). |

## Webhook event subscriptions

Marketplace apps that want push-based updates (instead of polling `/api/provider/*`) register a webhook URL that Hint POSTs events to.

**Register a webhook endpoint:**

```bash
curl -X POST "$HINT_API_URL/api/partner/webhook_endpoints" \
  -H "Authorization: Bearer $HINT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"webhook_url": "https://yourapp.example.com/hint/events"}'
```

There is **no per-endpoint event filter today** — a registered endpoint receives every event the partner's integrations emit. The endpoint should ignore event types it doesn't care about.

**Discover which event types fire:**

```bash
curl "$HINT_API_URL/api/partner/webhook_events" \
  -H "Authorization: Bearer $HINT_API_KEY"
```

Returns a flat array of `resource.action` strings (e.g. `["company.created", "customer_invoice.paid", "patient.inactive", ...]`) derived from Hint's live event registry. Use this list to know what event types your handler may see. New `resource.action` pairs may be added without an API version bump — implementations should `default:` ignore unknown types rather than reject.

**Payload shape** delivered to `webhook_url`:

```jsonc
{
  "id":          "evt-...",                       // event ident
  "type":        "patient.created",               // <resource>.<action>
  "practice_id": "prc-...",                       // which practice triggered it
  "created_at":  "2026-05-26T14:23:00.000Z",
  "object":      { ...full resource snapshot... } // shape matches GET /api/provider/<resource>/:id
}
```

The `object` field carries the resource snapshot — its shape matches the corresponding `GET /api/provider/<resource>/:id` response, so client code that already parses that endpoint can reuse the same field accessors.

**Signature verification.** Each request carries an `X-Hint-Signature: sha256=<hmac>` header — HMAC-SHA256 of the raw request body, keyed by the backend's webhook signature key (Partner Portal → Webhook Settings → Webhooks Signature Key, mirrored to the `HINT_WEBHOOK_SECRET` env var on Hosted Mode deploys). Compute the expected signature on receipt and reject mismatches. [`node-template.md`](./node-template.md) ships a Node implementation that wires this up correctly.

**Retries.** Hint retries non-2xx responses with exponential backoff (configured per partner). The endpoint must respond with a 2xx within ~10s to count as delivered; failed deliveries are visible via `GET /partner/webhook_requests`.

## Services list contains only web services

`GET /api/partner/partner_products/:partner_product_id/app/services` returns the partner-managed web services (the ones running the partner's code). Pick the row with `status: 'active'` and read `service_url` for `$APP_URL`. Most apps have exactly one web service.

The auto-provisioned Postgres sibling that backs `DATABASE_URL` is managed entirely by Hint and is not exposed via the API. The connection string is delivered via the `DATABASE_URL` env var injected into the web service.
