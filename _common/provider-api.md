# Hint Provider API (shared fragment)

The Provider API is what marketplace apps use to read + write practice data on behalf of the practice that installed them.

## Reading the docs

- **Browsable reference**: https://developers.hint.com/reference. The index page is JS-rendered, so scraping it for endpoint discovery is unreliable — direct links to specific endpoint pages survive better, but the canonical way to read these docs from an AI agent is the MCP server below.
- **MCP server (recommended for agents)**: developers.hint.com is hosted on ReadMe, which exposes an MCP server. Install URL: `https://developers.hint.com/mcp`. Once installed, the agent can query endpoint definitions, parameters, response schemas, and example payloads natively — no scraping, no guessing field names. Set this up before exploring the API; it'll save hours.
- **Object schema discovery without docs**: if MCP isn't an option, use the introspection pattern in [Discovering object schemas](#discovering-object-schemas) below.

## Authentication

All `/api/provider/*` calls use the **practice-scoped access token** returned by `POST /api/oauth/tokens` during the `/hint/connect/:code` exchange — NOT the partner-wide `HINT_API_KEY`. Persist the access token server-side, keyed by `practice_id`, and look it up on every Provider API call.

```
Authorization: Bearer <practice_access_token>
```

**428 means you used the wrong key.** If a `/api/provider/*` call returns `428 Precondition Required` ("This action requires a Practice"), you sent the partner-wide `HINT_API_KEY` instead of a practice-scoped access token. The partner API key cannot call `/api/provider/*` directly — every provider call has to be on behalf of a specific practice, scoped via the access token. The 428 is the platform refusing to act without that scope.

## Key endpoints

Verified against the live route table — these exist as top-level resources:

- `GET /api/provider/patients`, `GET /api/provider/patients/:id` — list + show patients
- `GET /api/provider/memberships`, `GET /api/provider/memberships/:id` — list + show memberships (and their member rows)
- `GET /api/provider/practitioners` — list practitioners
- `GET /api/provider/customer_invoices` — list invoices billed to patients. Supports `?expand=charges` to inline the line items.
- `GET /api/provider/customer_invoices/:id/charges` — list the charges (line items) on a specific invoice. **Note:** charges are NOT a top-level `/api/provider/charges` resource — they're nested under `customer_invoices`. A bare `GET /api/provider/charges` returns 404.
- `GET /api/provider/practice_invoices` — list invoices the practice owes Hint (not patient invoices)
- `GET /api/provider/payments` — list payments
- `GET /api/provider/companies`, `GET /api/provider/companies/:id/sponsorships` — companies + their sponsored memberships
- `GET /api/provider/practitioners/:id`, `GET /api/provider/locations`

There is no `GET /api/provider/invoices` — partners usually want `customer_invoices`.

## Discovering object schemas

The doc site renders schemas in JS, so scraping for field names is unreliable. The fastest way to confirm a real schema is to introspect a live response. While iterating, add a temporary debug route to your app that returns the raw provider-API response untouched:

```js
if (req.method === 'GET' && url.pathname === '/debug/provider-sample') {
  const session = requireSession(req, res); if (!session) return;
  const resp = await hintApi('GET', '/api/provider/customer_invoices?limit=1', session.access_token);
  res.writeHead(200, { 'Content-Type': 'application/json' });
  return res.end(JSON.stringify(resp.body, null, 2));
}
```

Hit it once, read the keys, remove the route before promoting to live. Faster than guessing `amount` vs `amount_cents` vs `total_in_cents` across a redeploy cycle.

## Response-shape gotchas

Read these before writing client code:

- **List endpoints return a bare JSON array**, not `{patients: [...]}` or `{data: [...]}`. Parse with `Array.isArray(res) ? res : []` and iterate directly. Do NOT do `res.patients || res.records || []` — that silently produces an empty array.
- **Pagination uses `limit` + `offset`** (not `page` / `page_size`). Example: `GET /api/provider/patients?limit=50&offset=100`. Max `limit` is 100. If a response returns exactly `limit` rows, paginate.
- **Date filters use ReadMe-style bracketed operators**: `?created_at[gte]=2026-01-01&created_at[lte]=2026-12-31`. Same shape for `updated_at`, `paid_at`, etc. Each timestamp field accepts `[gte]`, `[gt]`, `[lte]`, `[lt]`, `[eq]`. This is the most common provider-API filter idiom after pagination.
- **Archived rows are excluded by default.** Inverse: `?filter=archived` (only archived), `?filter=all` (both). If an app shows "no records" and the practice expects to see some, archive state is the first thing to check.

## Sandbox vs live hosts

Per the [official docs](https://developers.hint.com/reference), both `https://api.sandbox.hint.com` and `https://api.hint.com` exist (along with `https://api.staging.hint.com`). Empirically, sandbox keys (`sbx-` prefix) accept calls against both `api.hint.com` and `api.sandbox.hint.com` and return identical data. **Use `https://api.hint.com` everywhere** — it's the simplest configuration, and the partner doesn't have to swap the host when promoting from sandbox to live (the key prefix is the only thing that changes).

## Hint JS SDK

For surfaces that need real-time practice context (current patient, current interaction, current user), load the SDK in the embedded HTML:

```html
<script src="$HINT_API_URL/hint-sdk.js"></script>
<script>
  HintSDK.init(() => {
    console.log('User:', HintSDK.user);              // { id, name, email, partner_roles }
    console.log('Patient:', HintSDK.currentPatient); // { id, name } or null
    console.log('Interaction:', HintSDK.interaction); // { id } or null
  });
  HintSDK.onCurrentPatientChanged((patient) => {
    // Update UI when the selected patient changes
  });
</script>
```

The SDK runs inside the iframe Hint embeds, so the host it's served from must match the `$HINT_API_URL` the app is integrated with.
