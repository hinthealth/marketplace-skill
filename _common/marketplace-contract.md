# Hint Marketplace Contract (shared fragment)

Every Hint marketplace app — regardless of stack or hosting — has to implement three HTTP routes and consume two reserved env vars. This file is the canonical description of that contract; individual skills link to it instead of repeating it.

## Required routes

| Route | What it does |
|---|---|
| `POST /hint/handshake` | Receives a signed payload from Hint at install/embed time. The app verifies the `X-Hint-Signature` header (HMAC-SHA256 of the request body, key = the partner's webhook secret), mints a session key, persists `{session_key, user, practice}` server-side, and returns the session key. |
| `POST /hint/connect/:code` | Receives an OAuth code from Hint after a practice installs the app. The app exchanges the code at `POST $HINT_API_URL/api/oauth/tokens` for a practice-scoped access token and persists `{partner_id, practice_id, access_token}` keyed by practice. |
| `GET /hint/<anchor_type>?session_key=...` | Renders the embedded UI for the surface type. Looks up the session by `session_key`, recovers the practice context, then renders the surface. `<anchor_type>` is one of `core_page`, `clinical_interaction`, or `settings`. |

## Required env vars

The deployed app reads these from `process.env` (or the equivalent in its language). Hint sets them automatically on Hosted-Mode services; Self-Hosted Mode apps set them themselves at deploy time.

| Env var | Purpose |
|---|---|
| `HINT_API_URL` | Base URL of the Hint API. **Sandbox**: `https://api.sandbox.hint.com`. **Live**: `https://api.hint.com`. The two hosts are not interchangeable — sandbox keys (prefix `sbx-`) only work against the sandbox host, live keys only against the live host. |
| `HINT_API_KEY` | Partner-wide API key for `/api/partner/*` calls. NOT used for `/api/provider/*` calls (those need the practice-scoped access token from `/hint/connect/:code`). |
| `HINT_PARTNER_ID` | Stable partner ident (e.g. `ptr-...` / `sbx-ptr-...`). Useful for log scoping. |
| `HINT_WEBHOOK_SECRET` | Used to verify the `X-Hint-Signature` header on `POST /hint/handshake`. The partner finds this in the Partner Portal under **API Keys → Webhooks Signature Key**. |
| `DATABASE_URL` | Postgres connection string. Only present when the auto-provisioned sibling database is connectable. |

## Signature verification

`POST /hint/handshake` MUST verify the request signature before doing anything else. Pseudocode:

```
expected = "sha256=" + hmac_sha256(HINT_WEBHOOK_SECRET, raw_request_body)
provided = request.header["X-Hint-Signature"]
if not constant_time_eq(expected, provided): return 401
```

Use constant-time comparison (e.g. Node `crypto.timingSafeEqual`, Ruby `Rack::Utils.secure_compare`, Python `hmac.compare_digest`). String equality leaks timing information.

## Smoke test

A correctly-implemented app responds this way to unauthenticated probes:

```bash
curl -sS -o /dev/null -w "GET /                              → HTTP %{http_code}\n" "$APP_URL/"
curl -sS -o /dev/null -w "POST /hint/handshake (unsigned)    → HTTP %{http_code}\n" -X POST "$APP_URL/hint/handshake"
curl -sS -o /dev/null -w "GET /hint/<anchor_type> (no sess)  → HTTP %{http_code}\n" "$APP_URL/hint/core_page"
```

Expected: `GET /` → 200 (any health check). `POST /hint/handshake` unsigned → 401 (signature verification is working). `GET /hint/<anchor_type>` without session → 200 or 401 (both acceptable; some apps render a "no session" placeholder, others reject).

A `404` on `/hint/handshake` means the route doesn't exist. A `200` on an unsigned handshake means signature verification is missing — that's a critical security gap.
