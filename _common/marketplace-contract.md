# Hint Marketplace Contract (shared fragment)

Every Hint marketplace app — regardless of stack or hosting — has to implement three HTTP routes and consume the reserved env vars. This file is the canonical description of that contract; individual skills link to it instead of repeating it.

## Required routes

| Route | What it does |
|---|---|
| `POST /hint/handshake` | Receives a signed payload from Hint at install/embed time. The app verifies the `X-Hint-Signature` header (HMAC-SHA256 of the request body, key = the partner's webhook secret), mints a session key, persists `{session_key, user, practice}` server-side, and returns the session key. |
| `POST /hint/connect/:code` | Receives an authorization code from Hint after a practice installs the app. The app exchanges the code at `POST $HINT_API_URL/api/partner/installations/connect` for the installation (the practice-scoped credential is in `api_keys[0].token`) and persists `{partner_id, practice_id, access_token}` keyed by practice. The response also carries the installed `product` (`{ id, name, slug }`) — record its `id` if you offer more than one product, so you can tell which one each practice installed. |
| `GET /hint/<surface_type>?session_key=...` | Renders the embedded UI for the surface type. Looks up the session by `session_key`, recovers the practice context, then renders the surface. `<surface_type>` is one of `core_page`, `clinical_interaction`, or `settings`. |

## Activation mode: `connect` always fires; `pending` vs `active` is separate

A product's **activation mode** decides *when a connection goes live* — it does **not** gate the connect flow. Whatever the mode, Hint sends the `POST /hint/connect/:code` request for **every** headless install, so your connect handler must always provision/link the practice and store its access token — including for installs that are not active yet. (This is the key thing to get right: a `partner_activate` or `practice_activate` install still calls `connect`, not just `instant` ones.)

| Mode | State after the practice finishes setup | Who activates |
|---|---|---|
| `instant` (default) | `active` immediately | nobody — active on install |
| `practice_activate` | `active` when the practice finishes setup in Hint | the practice |
| `partner_activate` | `pending` until you activate it | you, via `POST /api/partner/installations/:id/activate` |

Only `active` connections receive webhooks and (if you bill through Hint) start billing. So build the connect handler to be **idempotent and not assume the connection is active yet** — provision on `connect`, and treat "receiving events / billing" as something that only begins once the connection is active. For `partner_activate`, you can exchange the code via `POST /api/partner/installations/connect` with `{ "activate": false }` to fetch the practice key while the install stays `pending`, then activate later. List installs awaiting your action with `GET /api/partner/installations?status=pending`.

## Device capabilities in the embed (camera / microphone / geolocation)

Embedded surfaces run in a cross-origin iframe, so capabilities like `navigator.mediaDevices.getUserMedia` are blocked by the browser unless Hint delegates them to the app's origin. Delegation is opt-in per app: set `browser_allow_list` (e.g. `["camera", "microphone"]`; supported: `camera`, `microphone`, `geolocation`) via `PATCH /partner/products/:id/app`, and Hint renders the app's embed iframes with a matching Permissions Policy `allow` attribute. The end user still sees the browser's normal permission prompt. Changes apply at embed time — already-open surfaces must be reloaded. Current iOS honors the delegation (camera/microphone/geolocation verified end-to-end in iOS Safari, including `facingMode` switching and `applyConstraints` zoom); older iOS versions may still block getUserMedia in cross-origin iframes, so keep a `<input type="file" accept="image/*" capture>` fallback for those.

Surface sizing: clinical surfaces (`clinical_interaction`, `clinical_chart`) size the embed iframe **only** from the app's `resized` reports — include `hint-sdk.js` (it auto-reports height via a ResizeObserver from load) or the surface is clipped to a ~150px strip. The same applies to `core_page` surfaces with auto-adjust height disabled.

Error-handling guidance: on a `NotAllowedError`, check the handshake's `browser_allow_list` first. Capability not in the list → show "This app hasn't been granted camera access in Hint — enable it under App Settings." Capability in the list → the user denied the browser prompt; point them at the browser's site permissions.

## Localhost mode: CORS must allow BOTH portal origins

In localhost mode the handshake is browser-mediated: the portal hosting the surface calls your local server directly. Which portal that is depends on the surface type — the provider portal for `core_page`/`settings`, the **clinical portal** for `clinical_interaction`/`clinical_chart`. Do not hardcode one origin: echo the request `Origin` header back in `Access-Control-Allow-Origin` when it matches an allowlist containing both portal origins for your environment, send `Vary: Origin`, allow the `Content-Type` and `X-Hint-Signature` headers, and handle the `OPTIONS` preflight.

## Required env vars

The deployed app reads `HINT_API_URL`, `HINT_API_KEY`, `HINT_PARTNER_ID`, `HINT_WEBHOOK_SECRET`, and (optionally) `DATABASE_URL` from `process.env` (or the equivalent in its language). Hint sets them automatically on Hosted-Mode services; Self-Hosted Mode apps set them themselves at deploy time. Full descriptions: see [`api-conventions.md` § Reserved env vars](./api-conventions.md#reserved-env-vars).

## Handshake payload shape

After signature verification, parse `request.body` as JSON. Top-level field reference:

| Path | Type | Notes |
|---|---|---|
| `timestamp` | string | ISO-8601 timestamp of the moment Hint signed the request (e.g. `2025-10-30T14:20:19.316986-03:00`). The signature covers the body, including this, so it doubles as a replay-defense nonce — reject requests whose `timestamp` is older than a few minutes if you want strict freshness. |
| `practice.id` | string | Public id for the practice (e.g. `prc-xxxx`). Use this as the tenancy key for everything the app stores on behalf of this practice. |
| `practice.name` | string | Display name. Safe to render in UI. |
| `user.id` | string | Public id of the signed-in staff user. |
| `user.email` | string | Login email. |
| `user.name` / `user.first_name` / `user.last_name` | string | Display name parts. |
| `user.phones` | object[] | Array of phone records (`{ id, ... }`). May be empty. |
| `user.partner_roles` | string[] | Roles Hint resolved for this user, drawn from the role list the partner declared in their App config. Use these for in-app RBAC; they're separate from Hint's practice-level permissions. **May be empty** — an empty array means the user has no access; render an access-denied state rather than erroring. |
| `integration.id` | string | The Hint integration record's id. Persist for support / debugging. |
| `access_token` | string | **Session-scoped api key minted for THIS embed session.** Use this to call `/api/provider/*` from the app instead of the practice-wide access token where possible — see the section below. |
| `browser_allow_list` | string[] | Browser capabilities Hint has delegated to this app's iframes (`camera`, `microphone`, `geolocation`; empty = none). Preflight against this before calling `getUserMedia`: if the capability is missing here, the fix is the App Settings toggle in Hint — the browser will throw the same `NotAllowedError` as a user denial, so telling the cases apart requires this field. |
| `access_context` | string | Either `standard` (normal staff user) or `platform_support` (a Hint employee accessing the app in a platform-operator capacity). See [Handling `platform_support` sessions](#handling-platform_support-sessions) — correct handling is a required capability. |
| `patient.id` | string | Only on `clinical_interaction` and `clinical_chart` surfaces — public id of the patient being viewed. |
| `interaction.id` | string | Only on `clinical_interaction` surfaces — public id of the open clinical interaction. |

The payload may carry additional fields over time. Decode permissively (ignore unknowns); only require the fields the app actually reads.

### Handling `platform_support` sessions

`access_context` tells the app what kind of session the handshake represents:

- `standard` — a regular practice staff user, accessing the app with their assigned `partner_roles`. This is the default.
- `platform_support` — a Hint employee accessing the app in their own capacity as a platform operator, not as a practice user. This happens when a practice reports a support issue to Hint, or when a Hint team member trains a practice on the app. It is **delegation, not impersonation** — the employee is not pretending to be a practice user. The `partner_roles` sent alongside is always the partner's designated admin role, so support has full visibility.

Handling `platform_support` correctly is a **required capability**. When `access_context` is `platform_support`, the app MUST:

- **Not provision a persistent user** in the practice's account from the handshake `user`. These sessions are ephemeral — store session state only (keyed by practice), the same way you'd treat a session from your own support team. Do not create a customer user record.
- **Scope access to the handshake's practice** with admin-level (or read-only, per the app's policy) visibility. Never expose another practice's data.
- **Attribute audit-log entries to a platform-support session**, not to a customer user, so vendor-initiated access is distinguishable from customer activity.
- **Surface a visible indicator** (e.g. a "Hint support session" banner) so it's clear the account is being viewed by Hint.

Decode permissively: `access_context` may gain values over time, so treat any unrecognized value as non-`standard` and default to the most conservative handling. In healthcare, vendor access to a customer's environment carries BAA and compliance implications — this distinction keeps audit trails accurate and access appropriately scoped.

### Use the handshake's `access_token` to call the Provider API

The `access_token` Hint includes in the handshake body is a **session-scoped api key** generated specifically for the lifetime of this embed session. Its scope is the same practice + user that the handshake identifies, so for any `/api/provider/*` calls the app needs to make WHILE this surface is mounted, prefer it:

```
Authorization: Bearer <handshake.access_token>
```

This is preferable to the longer-lived practice access token (obtained via `POST /hint/connect/:code`) because:

- It's automatic — every handshake mints a fresh one; no separate exchange step.
- Its lifetime is the embed session, not the practice's lifetime — leaks die quickly.
- It carries the acting user's identity, so server-side audit logs attribute correctly.

Use the practice access token (the one stored under `practice_id` after `/hint/connect/:code`) only for **server-to-server work that happens outside an embed session** — webhooks, scheduled jobs, async fanout to other practices. For "the embedded surface is open and needs to read /provider/patients", the handshake token is the right tool.

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
2. `POST /hint/connect/:code` exchanges the code at `POST /api/partner/installations/connect`, which returns the installation `{ practice: { id }, product: { name, slug }, api_keys: [{ token }], ... }`. The app persists `{ practice_id, access_token: api_keys[0].token }` keyed by `practice_id`.
3. `GET /hint/<surface_type>?session_key=...` looks up the session by `session_key` to recover the `practice_id`, then scopes every subsequent query to it.

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

Not required for v1. Most marketplace apps poll for what they need. See [`api-conventions.md` § "Webhook event subscriptions"](./api-conventions.md#webhook-event-subscriptions) for the registration call, the canonical event-type discovery endpoint (`GET /partner/webhook_events`), the event payload shape, and signature verification notes.

## Smoke test

A correctly-implemented app responds this way to unauthenticated probes:

```bash
curl -sS -o /dev/null -w "GET /                              → HTTP %{http_code}\n" "$APP_URL/"
curl -sS -o /dev/null -w "POST /hint/handshake (unsigned)    → HTTP %{http_code}\n" -X POST "$APP_URL/hint/handshake"
curl -sS -o /dev/null -w "GET /hint/<surface_type> (no sess)  → HTTP %{http_code}\n" "$APP_URL/hint/core_page"
```

Expected:

- `GET /` → 200 if the app implements a health check at `/`, 404 if it doesn't. Both are fine — the contract doesn't require a root route.
- `POST /hint/handshake` unsigned → 401 (signature verification is working). A 200 means signature verification is missing — that's a critical security gap. A 404 means the route doesn't exist.
- `GET /hint/<surface_type>` without session → 200 or 401 (both acceptable; some apps render a "no session" placeholder, others reject).
