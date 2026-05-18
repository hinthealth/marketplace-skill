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
| `HINT_WEBHOOK_SECRET` | Used to verify the `X-Hint-Signature` header on `POST /hint/handshake`. The partner finds this in the Partner Portal under **API Keys → Webhooks Signature Key**. |
| `DATABASE_URL` | Postgres connection string (only present when the auto-provisioned sibling database is connectable). |
