# Hint Marketplace Contract (shared fragment)

Every Hint marketplace app — regardless of stack or hosting — has to implement three HTTP routes and consume the reserved env vars. This file is the canonical description of that contract; individual skills link to it instead of repeating it.

## Required routes

| Route | What it does |
|---|---|
| `POST /hint/handshake` | Receives a signed payload from Hint at install/embed time. The app verifies the `X-Hint-Signature` header (HMAC-SHA256 of the request body, key = the partner's webhook secret), mints a session key, persists `{session_key, user, practice}` server-side, and returns the session key. |
| `POST /hint/connect/:code` | Receives an OAuth code from Hint after a practice installs the app. The app exchanges the code at `POST $HINT_API_URL/api/oauth/tokens` for a practice-scoped access token and persists `{partner_id, practice_id, access_token}` keyed by practice. |
| `GET /hint/<anchor_type>?session_key=...` | Renders the embedded UI for the surface type. Looks up the session by `session_key`, recovers the practice context, then renders the surface. `<anchor_type>` is one of `core_page`, `clinical_interaction`, or `settings`. |

## Required env vars

The deployed app reads `HINT_API_URL`, `HINT_API_KEY`, `HINT_PARTNER_ID`, `HINT_WEBHOOK_SECRET`, and (optionally) `DATABASE_URL` from `process.env` (or the equivalent in its language). Hint sets them automatically on Hosted-Mode services; Self-Hosted Mode apps set them themselves at deploy time. Full descriptions: see [`api-conventions.md` § Reserved env vars](./api-conventions.md#reserved-env-vars).

## Handshake payload shape

After signature verification, parse `request.body` as JSON. Top-level field reference:

| Path | Type | Notes |
|---|---|---|
| `timestamp` | integer | Unix seconds at the moment Hint signed the request. The signature covers the body, including this, so it doubles as a replay-defense nonce — reject requests with a `timestamp` older than a few minutes if you want strict freshness. |
| `practice.id` | string | Public id for the practice (e.g. `prc-xxxx`). Use this as the tenancy key for everything the app stores on behalf of this practice. |
| `practice.name` | string | Display name. Safe to render in UI. |
| `user.id` | string | Public id of the signed-in staff user. |
| `user.email` | string | Login email. |
| `user.name` / `user.first_name` / `user.last_name` | string | Display name parts. |
| `user.phones` | object[] | Array of phone records (`{ id, ... }`). May be empty. |
| `user.partner_roles` | string[] | Roles the partner assigned to this user under their App config. Use these for in-app RBAC; they're separate from Hint's practice-level permissions. |
| `integration.id` | string | The Hint integration record's id. Persist for support / debugging. |
| `access_token` | string | **Session-scoped api key minted for THIS embed session.** Use this to call `/api/provider/*` from the app instead of the practice-wide OAuth token where possible — see the section below. |
| `access_context` | string | Either `standard` (normal staff user) or `platform_support` (a Hint admin acting on the practice's behalf, e.g. for support). Apps may want to render a "viewing as Hint support" banner when this is `platform_support`. |
| `patient.id` | string | Only on `clinical_interaction` and `clinical_chart` surfaces — public id of the patient being viewed. |
| `interaction.id` | string | Only on `clinical_interaction` surfaces — public id of the open clinical interaction. |

The payload may carry additional fields over time. Decode permissively (ignore unknowns); only require the fields the app actually reads.

### Use the handshake's `access_token` to call the Provider API

The `access_token` Hint includes in the handshake body is a **session-scoped api key** generated specifically for the lifetime of this embed session. Its scope is the same practice + user that the handshake identifies, so for any `/api/provider/*` calls the app needs to make WHILE this surface is mounted, prefer it:

```
Authorization: Bearer <handshake.access_token>
```

This is preferable to the longer-lived practice OAuth access token (obtained via `POST /hint/connect/:code`) because:

- It's automatic — every handshake mints a fresh one; no separate exchange step.
- Its lifetime is the embed session, not the practice's lifetime — leaks die quickly.
- It carries the acting user's identity, so server-side audit logs attribute correctly.

Use the OAuth practice token (the one stored under `practice_id` after `/hint/connect/:code`) only for **server-to-server work that happens outside an embed session** — webhooks, scheduled jobs, async fanout to other practices. For "the embedded surface is open and needs to read /provider/patients", the handshake token is the right tool.

## Signature verification

`POST /hint/handshake` MUST verify the request signature before doing anything else. Pseudocode:

```
expected = "sha256=" + hmac_sha256(HINT_WEBHOOK_SECRET, raw_request_body)
provided = request.header["X-Hint-Signature"]
if not constant_time_eq(expected, provided): return 401
```

Use constant-time comparison (e.g. Node `crypto.timingSafeEqual`, Ruby `Rack::Utils.secure_compare`, Python `hmac.compare_digest`). String equality leaks timing information.

## Tenancy — every practice's data must be isolated

A marketplace app is a single deployed service that every practice installing it shares. The handshake payload identifies which practice each request is acting on behalf of — and **the app is responsible for using that identity to keep every practice's data isolated from every other practice's data.**

The rule, stated as crisply as possible:

> Every database table the app creates MUST have a non-null `practice_id` column. Every query — read AND write — MUST filter by `practice_id` sourced from the current session. No global queries against tables that hold tenant data.

Where the `practice_id` comes from:

1. `POST /hint/handshake` arrives with a signed payload that includes `practice.id`. The app persists `{ session_key, user, practice_id, ... }` server-side.
2. `POST /hint/connect/:code` returns `{ access_token, practice_id, ... }`. The app persists `{ practice_id, access_token }` keyed by `practice_id`.
3. `GET /hint/<anchor_type>?session_key=...` looks up the session by `session_key` to recover the `practice_id`, then scopes every subsequent query to it.

A reference helper every handler should go through (Node.js form; port the shape to whatever stack the app uses):

```js
function requireSession(req, res) {
  const sessionKey = req.headers['x-hint-session-key'] || new URL(req.url, 'http://x').searchParams.get('session_key');
  const session = sessions[sessionKey];
  if (!session) {
    res.writeHead(401, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'No session' }));
    return null;
  }
  return session; // { practice_id, access_token, user, ... }
}
```

Correct (every read/write scoped):

```sql
SELECT * FROM messages WHERE practice_id = $1 AND author_id = $2;
INSERT INTO messages (practice_id, body, author_id) VALUES ($1, $2, $3);
```

Wrong (cross-practice leak):

```sql
SELECT * FROM messages WHERE author_id = $1;          -- ❌ returns every practice's data
INSERT INTO messages (body, author_id) VALUES ($1, $2); -- ❌ writes without tenancy
```

The same rule applies to access tokens (used to call `/api/provider/*`): an app MUST use the access_token of the practice making the current request, never a token from a different practice's session. Cross-practice token reuse is a data-leak vector — Hint will revoke the access of any app caught doing it.

If the app does not persist any practice-scoped state (e.g. a pure presentation surface that calls `/api/provider/*` and renders the response without writing anything), tenancy is automatic — the access token is already practice-scoped. But the moment the app adds any persisted state (sessions count; see the `requireSession` example), `practice_id` columns become mandatory.

## Subscribing to events (optional)

The three required routes above cover install + embed. Apps that want **push-based** updates when practice data changes (instead of polling `/api/provider/*`) can additionally register a webhook URL with Hint — `POST /partner/webhook_endpoints` — and Hint will POST events (`patient.created`, `customer_invoice.paid`, `membership.updated`, etc.) to it with the same `X-Hint-Signature` HMAC the handshake uses.

Not required for v1. Most marketplace apps poll for what they need. See [`api-conventions.md` § "Webhook event subscriptions"](./api-conventions.md#webhook-event-subscriptions) for the registration call, the canonical event-type discovery endpoint (`GET /partner/integration_events`), the event payload shape, and signature verification notes.

## Smoke test

A correctly-implemented app responds this way to unauthenticated probes:

```bash
curl -sS -o /dev/null -w "GET /                              → HTTP %{http_code}\n" "$APP_URL/"
curl -sS -o /dev/null -w "POST /hint/handshake (unsigned)    → HTTP %{http_code}\n" -X POST "$APP_URL/hint/handshake"
curl -sS -o /dev/null -w "GET /hint/<anchor_type> (no sess)  → HTTP %{http_code}\n" "$APP_URL/hint/core_page"
```

Expected:

- `GET /` → 200 if the app implements a health check at `/`, 404 if it doesn't. Both are fine — the contract doesn't require a root route.
- `POST /hint/handshake` unsigned → 401 (signature verification is working). A 200 means signature verification is missing — that's a critical security gap. A 404 means the route doesn't exist.
- `GET /hint/<anchor_type>` without session → 200 or 401 (both acceptable; some apps render a "no session" placeholder, others reject).
