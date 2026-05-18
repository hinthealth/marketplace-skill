# Hint Provider API (shared fragment)

The Provider API is what marketplace apps use to read + write practice data on behalf of the practice that installed them. Full reference: https://developers.hint.com/reference.

## Authentication

All `/api/provider/*` calls use the **practice-scoped access token** returned by `POST /api/oauth/tokens` during the `/hint/connect/:code` exchange — NOT the partner-wide `HINT_API_KEY`. Persist the access token server-side, keyed by `practice_id`, and look it up on every Provider API call.

```
Authorization: Bearer <practice_access_token>
```

Using the partner-wide key against `/api/provider/*` returns 401 (or worse, on some endpoints, leaks cross-practice data — never assume "it worked, must be right").

## Key endpoints

- `GET /api/provider/patients` — list patients
- `GET /api/provider/patients/:id` — get patient details
- `GET /api/provider/memberships` — list memberships
- `GET /api/provider/practitioners` — list practitioners
- `GET /api/provider/charges` — list charges
- `GET /api/provider/invoices` — list invoices

Full reference: https://developers.hint.com/reference

## Response-shape gotchas

Read these before writing client code:

- **List endpoints return a bare JSON array**, not `{patients: [...]}` or `{data: [...]}`. Parse with `Array.isArray(res) ? res : []` and iterate directly. Do NOT do `res.patients || res.records || []` — that silently produces an empty array.
- **Pagination uses `limit` + `offset`** (not `page` / `page_size`). Example: `GET /api/provider/patients?limit=50&offset=100`. Max `limit` is 100. If a response returns exactly `limit` rows, paginate.
- **Archived rows are excluded by default.** Inverse: `?filter=archived` (only archived), `?filter=all` (both). If an app shows "no records" and the practice expects to see some, archive state is the first thing to check.
- **Sandbox vs live is host-level, not key-level.** Sandbox traffic goes to `https://api.sandbox.hint.com`, live traffic to `https://api.hint.com`. Sandbox API keys (prefix `sbx-`) only work against the sandbox host; live keys only against the live host. Pick the host based on the API key, not the endpoint path.

## Hint JS SDK

For surfaces that need real-time practice context (current patient, current interaction, current user), load the SDK in the embedded HTML:

```html
<!-- $HINT_API_URL is https://api.sandbox.hint.com for sandbox, https://api.hint.com for live -->
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

The SDK runs inside the iframe Hint embeds, so the host it's served from must match the Hint API host the app is integrated with.
