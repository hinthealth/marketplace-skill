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

### Calling `/api/provider/*` from the embedded UI — use the local proxy

When the embedded surface (Core Page, Clinical Interaction, etc.) needs to call a Provider endpoint **from the browser**, do NOT call `https://api.hint.com/api/provider/...` directly — the browser has no access token (and shouldn't; exposing it would let any page on the embed origin act as the practice). Every such direct call returns `401`.

Instead, route through the template's `/hint/api/provider/*` proxy:

```js
// ✅ RIGHT — local proxy, server attaches the practice's access_token
fetch('/hint/api/provider/patients?limit=10', {
  headers: { 'x-hint-session-key': SESSION_KEY },
}).then(r => r.json());
```

The proxy looks up `session.access_token` from the session row (Postgres-backed in the template), forwards the call to Hint with `Authorization: Bearer <access_token>`, and pipes the response back. Query strings, request bodies, and HTTP methods all pass through unchanged.

A `503 "Practice has not completed headless connect yet"` means the session exists but the access_token hasn't landed yet — the connect callback is still in flight. Retry once after a couple of seconds.

For server-to-server calls (background jobs, webhook handlers) that already have a stored access token but no live session, call `https://api.hint.com/api/provider/...` directly with that token. The proxy is for in-browser code acting on behalf of the currently-loaded practice.

Full template-side code is documented in [`node-template.md` § "Calling Hint's Provider API from the embedded UI"](./node-template.md#calling-hints-provider-api-from-the-embedded-ui).

## Endpoint discovery

Use the MCP server (above) for the canonical list of available endpoints, parameters, and response schemas. A few gotchas worth knowing up-front:

- **There is no `GET /api/provider/charges`** as a top-level resource. Charges are nested under invoices: `GET /api/provider/customer_invoices/:id/charges`, or inline them via `GET /api/provider/customer_invoices?expand=charges`. A bare `/charges` returns 404.
- **There is no `GET /api/provider/invoices`** — partners usually want `customer_invoices` (patient-facing invoices), or `practice_invoices` (what the practice owes Hint).
- **`/api/provider/users` and `/api/provider/practitioners` are NOT the same list.** They overlap but model different concepts and carry different fields, so picking the wrong one means the app silently has the wrong data even when the endpoint returns rows. Pick by what the app actually needs:
  - **`/users`** — staff accounts that can log into the Hint dashboard for the practice (admins, billing staff, schedulers, clinicians who happen to log in). Useful when the app needs "who has portal access" or "list users for an in-app permission picker". `/users` often returns `[]` in fresh sandboxes simply because no one has signed up yet — that's not the only reason to prefer `/practitioners`.
  - **`/practitioners`** — clinicians at the practice with credentialing identity: NPI, specialty, billing identity, signature-on-file, schedule-able. This is what shows up on encounters, what gets attributed in billing, what patients book with. Use this for "list doctors at this practice", "attribute revenue to a clinician", "render the visit-notes author dropdown".
  - The two overlap when a credentialed clinician also has a portal login — but a practitioner without a login is still a practitioner, and a staff user without credentials is not a practitioner. Don't conflate them.

## Response conventions

Every Provider API client should follow the conventions in [`api-conventions.md`](./api-conventions.md): list endpoints return bare JSON arrays, pagination is `limit`/`offset`, date filters use bracketed operators (`?created_at[gte]=...`), and archived rows are excluded by default. Read that file first — getting the response shape wrong silently produces empty results.

## Rate-limited endpoints — keep detail-fetch concurrency low

Detail (`/{id}`) endpoints on `/api/provider/*` rate-limit aggressively — empirically, fanning out detail fetches across a panel of ~80 patients at concurrency 5+ triggers 429s within the first few seconds, and at concurrency 3 still produces multiple retries per fetch. Two relevant endpoints surfaced first:

- `GET /api/provider/patients/{id}/interactions/lab/{id}` — fanning out one fetch per active member of a panel routinely 429s under modest concurrency.
- `GET /api/provider/interactions/{id}` (any type) — same shape.

The list-form siblings (`GET /api/provider/interactions?type=lab&...`) have looser limits and should be the default fetch shape — paginate the list, then only detail-fetch individual records when the UI genuinely needs the full body.

**Recommended concurrency caps for partner apps:**

| Workload | Cap |
|---|---|
| General detail fetches behind a user-visible action | ≤3 |
| Backstop / batch fill (24 h refresh, initial seed) | ≤1 |
| Bulk list fetch (paginated `GET /interactions?type=...`) | full speed, but page with `limit=100` and respect any `Retry-After` |

The template's `hintApi()` wrapper handles single-request retries on `429` with exponential backoff, but partners building dashboard-style apps will still need to **gate the fan-out itself**, not just per-request retries — a 100-request burst at concurrency 5 produces a 429 cascade that retries forever even if each individual retry "works". Use a simple semaphore in JS (`p-limit` or a hand-rolled `Promise.all` chunker) or equivalent in your stack.

## Delta queries via `updated_at[gt]`

List endpoints with an `updated_at` filter (interactions, memberships, patients, etc.) accept the bracket-notation operators `[gte]`, `[gt]`, `[lte]`, `[lt]` — same shape as the `created_at` filter mentioned above. **Use `updated_at[gt]` to fetch only what changed since the last sync** — the foundation of any caching layer. Without it, a dashboard app re-fetches the full panel on every visit, which scales linearly with practice size and routinely hits 8–25 s for ~80 members.

```
GET /api/provider/interactions?type=lab&updated_at[gt]=2026-05-22T18:00:00Z
GET /api/provider/memberships?updated_at[gt]=2026-05-22T18:00:00Z
GET /api/provider/patients?updated_at[gt]=2026-05-22T18:00:00Z
```

Pair with a Postgres-backed snapshot table keyed by `practice_id` and a `last_fetched_at` cursor: read the cached snapshot for instant render, fire `updated_at[gt]=<last_fetched_at>` to fetch only the delta, merge, and update the cursor. For correctness, also do a full backstop fetch once per 24 h per practice in case any deltas were missed. See [`_common/caching-patterns.md`](./caching-patterns.md) for the full recipe (snapshot table + snapshot-first render + delta refresh + 24 h backstop + per-practice and global advisory locks + HTML fragment swap on the client).

## Hint JS SDK

For surfaces that need real-time practice context (current patient, current interaction, current user), load the SDK in the embedded HTML:

```html
<script src="$HINT_API_URL/hint-sdk.js"></script>
<script>
  HintSDK.init(() => {
    // Ready — fields below are populated.
  });
</script>
```

The SDK runs inside the iframe Hint embeds, so the host it's served from must match the `$HINT_API_URL` the app is integrated with.

### API surface

| Member | Type | Notes |
|---|---|---|
| `HintSDK.init(callback)` | function | Pass a callback that fires once the SDK has connected to the host. All other members are unsafe to read before `init` resolves. |
| `HintSDK.user` | object | `{ id, name, email, partner_roles }`. The currently signed-in staff user; `partner_roles` is an array of role-name strings (matching whatever the partner configured under `partner_roles` in their App config). |
| `HintSDK.currentPatient` | object \| null | `{ id, name }` if the user is viewing a patient (clinical_interaction / core_page surfaces); `null` otherwise. |
| `HintSDK.interaction` | object \| null | `{ id }` if the surface is a clinical interaction; `null` otherwise. Most `core_page` and `settings` surfaces will see `null` here. |
| `HintSDK.onCurrentPatientChanged(callback)` | function | Subscribes to current-patient updates. Callback receives the new patient object (or `null`). Use this when the surface needs to react to the user switching charts without reloading. |

`HintSDK.currentPatient` is the only field that changes after `init`; `user` and `interaction` are fixed for the lifetime of the surface. Critical for `clinical_interaction` apps that need to follow the chart selection — without `onCurrentPatientChanged`, the surface will stale-render the patient it was opened with.

## What partner apps cannot match exactly

There are a handful of UI-side numbers in Hint that partner apps cannot perfectly reproduce from the public API. Don't chase the gap — flag it to the partner upfront.

- **Active Members KPI**: Hint's UI count and the API count typically differ by 3–5% because Hint's KPI excludes a few edge cases (very-recent activations, in-flight cancellations) that the public API surfaces. See [`provider-api-fields.md`](./provider-api-fields.md) under memberships for the exact field/scope detail. Apps that want a "matches the UI exactly" number should defer to Hint's reporting, not reconstruct it client-side.
