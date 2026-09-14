# Partner credentials (shared fragment)

A practice usually has several partners installed. When one of those partners has
issued the practice an API credential, your app can use it to call that partner's API
**on the practice's behalf** — so an app built against your own surface can pull data
from another partner the practice already uses.

This is a cross-partner capability by design: the credentials you can see are the
ones the *practice* holds, not only the ones your own product issued.

## The two endpoints

Both are `/api/provider/*` calls and use the **practice-scoped access token** —
the same token as every other Provider API call. See
[`provider-api.md`](./provider-api.md) § Authentication.

### 1. Discover what the practice has

```
GET /api/provider/partner_credentials
```

```json
[
  {
    "partner_name": "Acme Labs",
    "product_name": "Acme Results Sync",
    "product_slug": "acme-results-sync",
    "base_url": "https://api.acmelabs.example",
    "call_path": "direct"
  }
]
```

Only products holding an **active** credential are listed. The response carries **no
secret**, so this is safe to surface in your UI — it is how the practice picks which
integration they want your app to use.

### 2. Fetch one credential

```
GET /api/provider/installations/<product_slug>/credential
```

```json
{
  "id": "ppcred_...",
  "base_url": "https://api.acmelabs.example",
  "payload": "<the partner-issued secret>",
  "call_path": "direct"
}
```

`product_slug` is the value from the discovery response — not an installation id.

**Fetch one partner at a time**, the one you are about to call. Pulling every
credential up front means a single compromised app leaks every partner the practice
uses instead of one.

## Rules

**1. The credential never reaches the browser.** Fetch it server-side and make the
partner call server-side; return only the partner's response to your UI.

> ⚠️ The template's `/hint/api/provider/*` proxy forwards whatever path the browser
> asks for. Exclude the credential path from it — see
> [`node-template.md`](./node-template.md#calling-hints-provider-api-from-the-embedded-ui).
> Discovery through the proxy is fine; the fetch is not.

**2. Never log it and never expose it through a debug route.** If you add an
env-inspection or health route, report presence and length only (`_set`, `_len`,
`last4`) — never a value, and never the practice access token either.

**3. Branch on `call_path`.**

| `call_path` | What it means | What your app does |
|---|---|---|
| `direct` | Call the partner's API server-to-server | Use `payload` as the bearer credential against `base_url` |
| `proxy` | The partner expects Hint to broker the call | Do **not** handle the raw secret. Treat `payload` as absent and surface the integration as unavailable |

Code that assumes `direct` will mishandle a `proxy` partner.

**4. Treat it as rotatable.** Fetch it when you need it, or at boot — never bake it
into committed config or a build artifact. A partner can rotate it at any time, and
it is deactivated when the practice uninstalls. A `404` means there is no active
credential any more: re-run discovery rather than retrying the fetch.

**5. These calls only work from your deployed Hint app.** Both endpoints refuse
requests that do not originate from your app running on Hint's managed deployment
platform, so a token copied onto a laptop cannot read a credential. Expect `403
Access forbidden.` when testing from anywhere else — that is the gate working, not a
broken token. Exercise this path from a deployed revision.

## Worked example

```js
// Server-side. `accessToken` is the practice-scoped token from the session row.
async function callPartner(accessToken, productSlug, path) {
  const credential = await hintApiAs(
    accessToken, 'GET', `/api/provider/installations/${encodeURIComponent(productSlug)}/credential`
  ).then((r) => (r.status === 200 ? JSON.parse(r.body) : null));

  if (!credential) throw new Error('No active credential for ' + productSlug);
  if (credential.call_path === 'proxy') throw new Error('Partner requires brokered calls');

  const upstream = await fetch(credential.base_url.replace(/\/$/, '') + path, {
    headers: {
      Authorization: `Bearer ${credential.payload}`,
      Accept: 'application/json',
    },
  });
  return upstream.json();   // only this is returned to the browser
}
```

## Troubleshooting

| Symptom | Cause |
|---|---|
| `403 Access forbidden.` | The call did not come from your deployed Hint app — see rule 5 |
| `404` on the fetch | No active credential for that product; the partner may have revoked or rotated it |
| `404` on the fetch, slug looks right | You passed an installation id instead of `product_slug` |
| `428 Precondition Required` | You used the partner-wide `HINT_API_KEY` instead of the practice-scoped access token |
| Discovery returns `[]` | The practice holds no active partner credentials — not an error |
