# Hint Provider API (shared fragment)

The Provider API is what marketplace apps use to read + write practice data on behalf of the practice that installed them.

## Reading the docs

- **MCP server (recommended for agents)**: `https://developers.hint.com/mcp` — install this and the agent can query endpoint definitions, parameters, response schemas, and example payloads natively. Set it up before exploring the API; it'll save hours, and it's the authoritative way to discover field names and object schemas.
- **Inline field reference**: [`provider-api-fields.md`](./provider-api-fields.md) — schema sketch + gotchas for the five most-used resources (patients, memberships, customer_invoices, payments, practitioners). Use this when the MCP server isn't available — it's enough to write client code that returns correct numbers instead of guessing field names. Critical for revenue/membership-count metrics: read it before any chart or KPI work.
- **llms.txt index + per-endpoint `.md`**: when MCP isn't an option, [`https://developers.hint.com/llms.txt`](https://developers.hint.com/llms.txt) lists every guide and reference endpoint with its `.md` URL. Fetch the index once to discover slugs, then `curl https://developers.hint.com/reference/<slug>.md` for any one endpoint's full OpenAPI definition. Example: `couponlistallcoupons-1.md`, `patientlistallpatients-1.md`. Slugs are not perfectly guessable (lowercased tag + operationId, no separator) — start from `llms.txt`, don't hand-derive them.
- **Browsable reference**: https://developers.hint.com/reference — useful as a human-readable view of a specific endpoint. The index page itself is JS-rendered so scraping it is unreliable; use `llms.txt` for programmatic discovery.

## Authentication

All `/api/provider/*` calls use the **practice-scoped access token** returned by `POST /api/oauth/tokens` during the `/hint/connect/:code` exchange — NOT the partner-wide `HINT_API_KEY`. Persist the access token server-side, keyed by `practice_id`, and look it up on every Provider API call.

```
Authorization: Bearer <practice_access_token>
```

**428 means you used the wrong key.** If a `/api/provider/*` call returns `428 Precondition Required` ("This action requires a Practice"), you sent the partner-wide `HINT_API_KEY` instead of a practice-scoped access token. The partner API key cannot call `/api/provider/*` directly — every provider call has to be on behalf of a specific practice, scoped via the access token. The 428 is the platform refusing to act without that scope.

## Endpoint discovery

Use the MCP server (above) for the canonical list of available endpoints, parameters, and response schemas. Two gotchas worth knowing up-front:

- **There is no `GET /api/provider/charges`** as a top-level resource. Charges are nested under invoices: `GET /api/provider/customer_invoices/:id/charges`, or inline them via `GET /api/provider/customer_invoices?expand=charges`. A bare `/charges` returns 404.
- **There is no `GET /api/provider/invoices`** — partners usually want `customer_invoices` (patient-facing invoices), or `practice_invoices` (what the practice owes Hint).

## Response conventions

Every Provider API client should follow the conventions in [`api-conventions.md`](./api-conventions.md): list endpoints return bare JSON arrays, pagination is `limit`/`offset`, date filters use bracketed operators (`?created_at[gte]=...`), and archived rows are excluded by default. Read that file first — getting the response shape wrong silently produces empty results.

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
