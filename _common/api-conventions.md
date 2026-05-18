# Hint API Conventions (shared fragment)

This file is referenced by multiple skills in this repo. It documents the cross-cutting conventions every Hint marketplace integration needs to follow, so individual skills don't repeat them.

## Hosts

- **Hint API**: `https://api.hint.com` — works with both sandbox (`sbx-` prefix) and live API keys. The key determines the environment, not the host.
- **Hint API (sandbox-only alias)**: `https://api.sandbox.hint.com` — listed in the official docs; accepts sandbox keys and returns identical data to `api.hint.com`. Use `api.hint.com` everywhere for simplicity so the partner doesn't have to swap hosts when promoting from sandbox to live.
- **Hint API (staging)**: `https://api.staging.hint.com` — Hint-internal pre-production environment. Partners don't generally hit this directly.
- **Partner Portal**: `https://app.hint.com` — single portal for both sandbox and live; partners switch between workspaces inside it.

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

## Archived rows are excluded by default

List endpoints filter out archived records by default. Inverse queries:

- `?filter=archived` — only archived
- `?filter=all` — both active and archived

If a partner app shows "no records" and the practice expects to see some, archive state is the first thing to check.

## Reserved env vars

These are managed by Hint and set automatically on every deploy. Partner-supplied values for these keys are ignored:

| Var | Value |
|---|---|
| `HINT_API_URL` | Sandbox or live host depending on the partner |
| `HINT_API_KEY` | Partner API key |
| `HINT_PARTNER_ID` | Stable partner ident (e.g. `ptr-...` / `sbx-ptr-...`) |
| `HINT_WEBHOOK_SECRET` | Used to verify HMAC signatures on `/hint/handshake` payloads |
| `DATABASE_URL` | Postgres connection string (only present when the auto-provisioned sibling database is connectable) |
