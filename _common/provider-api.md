# Hint Provider API (shared fragment)

The Provider API is what marketplace apps use to read + write practice data on behalf of the practice that installed them.

## Reading the docs

- **MCP server (recommended for agents)**: `https://developers.hint.com/mcp` — install this and the agent can query endpoint definitions, parameters, response schemas, and example payloads natively. Set it up before exploring the API; it'll save hours, and it's the authoritative way to discover field names and object schemas.
- **Inline field reference**: [`provider-api-fields.md`](./provider-api-fields.md) — schema sketch + gotchas for the five most-used resources (patients, memberships, customer_invoices, payments, practitioners). Use this when the MCP server isn't available — it's enough to write client code that returns correct numbers instead of guessing field names. Critical for revenue/membership-count metrics: read it before any chart or KPI work.
- **llms.txt index + per-endpoint `.md`**: when MCP isn't an option, [`https://developers.hint.com/llms.txt`](https://developers.hint.com/llms.txt) lists every guide and reference endpoint with its `.md` URL. Fetch the index once to discover slugs, then `curl https://developers.hint.com/reference/<slug>.md` for any one endpoint's full OpenAPI definition. Example: `couponlistallcoupons-1.md`, `patientlistallpatients-1.md`. Slugs are not perfectly guessable (lowercased tag + operationId, no separator) — start from `llms.txt`, don't hand-derive them.
- **Browsable reference**: https://developers.hint.com/reference — useful as a human-readable view of a specific endpoint. The index page itself is JS-rendered so scraping it is unreliable; use `llms.txt` for programmatic discovery.

## Authentication

All `/api/provider/*` calls use the **practice-scoped access token** returned by `POST /api/partner/installations/connect` (as `api_keys[0].token`) during the `/hint/connect/:code` exchange — NOT the partner-wide `HINT_API_KEY`. Persist the access token server-side, keyed by `(HINT_PRODUCT_ID, practice_id)` (records which products a practice installed on a shared-backend Postgres; the token is the same across a practice's products), and look it up on every Provider API call.

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

## Creating an interaction when the partner has several installed products

`POST /api/provider/patients/{id}/interactions/partner` (and the `/interactions/lab` sibling) attributes
the interaction to one of your installed products, so Hint can reopen the right app later. It works out
which one in this order:

1. `partner_product_id` in the request body, if you send one;
2. the product whose app session minted the API key, when the call comes from an embedded surface;
3. the practice's only installed product of yours, when there is exactly one.

If the practice has installed **more than one** of your products and neither of the first two applies, the
create fails with `400` and a message beginning `Multiple products are installed for this practice`. This
is the common trap for a partner with several marketplace listings: the same code that works at every
single-product practice starts failing at a two-product one, with nothing about the request having changed.

Send `partner_product_id` explicitly whenever your backend knows which product it is acting for — it is
accepted at every practice, single-product ones included, so there is no reason to make it conditional.
The value is the product's `ppro-…` id, and it is validated against that practice's installed products, so
naming a product the practice has not installed is also a `400`.

Calls made from inside an embedded surface do not need it: the session already names the product.

## `patient_access` — which interactions the patient can see

Clinical interactions carry a boolean `patient_access` field on both the list (`GET /api/provider/interactions`) and detail endpoints. It is `true` when the interaction has been shared with the patient and is viewable by them, `false` otherwise. Use it to decide what a patient-facing surface should display — do not assume every interaction returned by the Provider API is patient-viewable.

**Labs: show only the PDF report to patients.** For lab interactions, even when `patient_access` is `true`, display only the lab's PDF report (the file under the interaction's `report`/files) to patients — not the raw structured result values. This is a data-display restriction from the lab data source (Health Gorilla). Raw result fields can be used in provider-facing views, but patient-facing surfaces must render the PDF only.

## Downloading interaction files

Clinical interactions carry their attachments as a `files` array on the interaction response, each entry `{ id, filename }`. That payload names the files but is not itself downloadable — the raw storage keys are private. To get bytes you exchange a file for a short-lived signed URL through one of two endpoints:

```
GET /api/provider/interactions/{id}/files/download_urls          → every file on the interaction
GET /api/provider/interactions/{id}/files/{file_id}/download_url  → one file, by its `id` from the files array
```

Both return the same element shape (`download_urls` a bare array of them, `download_url` a single one):

```json
{
  "id": "Fah-3TWDJuQ1MSe-lBTiPh",
  "filename": "lab-results.pdf",
  "url": "https://practice-bucket.s3.amazonaws.com/...?X-Amz-Signature=...",
  "expires_at": "2026-07-01T12:05:00.000Z"
}
```

**URLs are PDF-only.** Only files that resolve to `application/pdf` get a signed `url`. On `download_urls` a non-PDF stays in the array with `url: null` and `expires_at: null` (so you can tell "not downloadable" from "not returned"). On `download_url` a non-PDF is a `422`, and an `id` that isn't on that interaction is a `404`.

**URLs are short-lived — sign at download time.** `expires_at` is about 5 minutes out and the link is meant to be used once, not saved. Call the endpoint when the user actually clicks download, then redirect to (or fetch) the returned `url` — do not cache it, store it in your DB, or precompute it during a sync.

This is the mechanism behind the labs-PDF rule above: to show a patient a lab's PDF report, take that file's `id` from the interaction's `files` array and fetch its `download_url`.

**Available since API version `2026-07-01`.** Clients pinned to an earlier version get `404` on both routes.

## Delta queries via `updated_at[gt]`

List endpoints with an `updated_at` filter (interactions, memberships, patients, etc.) accept the bracket-notation operators `[gte]`, `[gt]`, `[lte]`, `[lt]` — same shape as the `created_at` filter mentioned above. Useful when an app needs "what changed since the last sync" instead of the full list:

```
GET /api/provider/interactions?type=lab&updated_at[gt]=2026-05-22T18:00:00Z
GET /api/provider/memberships?updated_at[gt]=2026-05-22T18:00:00Z
GET /api/provider/patients?updated_at[gt]=2026-05-22T18:00:00Z
```

Most apps don't need this — fetching the full list on each render, with the template's built-in `429` retry, is the default. The delta operator is the primitive you'd build on for an explicit caching layer; the actual snapshot-and-merge recipe is in [`_common/caching-patterns.md`](./caching-patterns.md) and is only worth reaching for once the symptoms in the next section appear.

## Advanced: caching for fan-out dashboards

Skip this section unless the app is a Core Page dashboard summarizing data across the practice's whole patient panel **and** the partner has hit one of these symptoms in production:

- Cold loads >8 s on a ~80-member panel.
- `429` rate-limit cascades in the logs when multiple users open the surface concurrently.
- The partner explicitly asks for a caching pattern.

If any of those apply, [`_common/caching-patterns.md`](./caching-patterns.md) has the full recipe (snapshot table + snapshot-first render + delta refresh + 24 h backstop + per-practice and global advisory locks + HTML fragment swap on the client). It's ~150 lines of Postgres + JS and adds real complexity — don't bake it into v1 reflexively.

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
| `HintSDK.user` | object | `{ id, name, email, first_name, last_name, phones }`. The currently signed-in staff user. Note: `partner_roles` and `access_context` are NOT on the SDK user object — they are delivered only in the server-to-server handshake payload (see [`marketplace-contract.md`](./marketplace-contract.md#handshake-payload-shape)). Read them on your server, not in the browser. |
| `HintSDK.currentPatient` | object \| null | `{ id, name }` if the user is viewing a patient (clinical_interaction / core_page surfaces); `null` otherwise. |
| `HintSDK.interaction` | object \| null | `{ id }` if the surface is a clinical interaction; `null` otherwise. Most `core_page` and `settings` surfaces will see `null` here. |

Every SDK field is fixed for the lifetime of the surface, `currentPatient` included. Read it once in the `init` callback. If the user switches to another patient on a surface that allows it, Hint tears the embed down and mounts it again with a fresh `init()` carrying the new patient — your page reloads from scratch rather than being notified in place. Keep anything that must survive that reload on your backend, keyed by practice and patient ID, and never build cross-patient state into one surface.

## What partner apps cannot match exactly

There are a handful of UI-side numbers in Hint that partner apps cannot perfectly reproduce from the public API. Don't chase the gap — flag it to the partner upfront.

- **Active Members KPI**: Hint's UI count and the API count typically differ by 3–5% because Hint's KPI excludes a few edge cases (very-recent activations, in-flight cancellations) that the public API surfaces. See [`provider-api-fields.md`](./provider-api-fields.md) under memberships for the exact field/scope detail. Apps that want a "matches the UI exactly" number should defer to Hint's reporting, not reconstruct it client-side.
