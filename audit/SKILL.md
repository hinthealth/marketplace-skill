# /hint-marketplace-audit — Security + Contract Audit of a Deployed App

Runs a pass/fail audit against a deployed marketplace app: confirms it implements the marketplace contract correctly, doesn't leak data across practices, and is wired up consistently with what the Hint Partner Portal expects.

The audit only reads — it never PATCHes anything. Output is a structured report the partner can act on.

## When to use this

- The partner just finished building or retrofitting their app and wants a sanity check before opening a release.
- A new dev joined the partner's team and wants to confirm the previous owner's implementation is correct.
- Something's broken in production and the partner is trying to triage whether it's a marketplace contract issue, an API auth issue, or something else.

## Required reading

Fetch and read these before running anything — the audit checks are derived from them:

1. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/marketplace-contract.md — the routes + signature verification
2. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/api-conventions.md — host + auth rules
3. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/provider-api.md — the practice-scoped token rule
4. https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/_common/provider-api-fields.md — field names + status-vs-enrollment_status + revenue-source gotchas (needed if the audit looks at the app's metric/KPI code)

## Platform URLs

Set `$HINT_API_URL=https://api.hint.com` for both sandbox and live work (Partner Portal at `https://app.hint.com`). Full conventions: [`_common/api-conventions.md`](../_common/api-conventions.md).

## Step 1: Gather Inputs

Ask the partner:

1. **Partner API key** — sandbox (`sbx-...`) or live. The audit runs against whichever environment the key belongs to.
2. **App URL (optional)** — if the partner already knows their `$APP_URL`, save it. Otherwise the audit will discover it from `GET /partner/partner_products/:partner_product_id/app/services`.
3. **Webhooks signature key (optional)** — needed for the "valid signature accepted" probe. Find it in the Partner Portal under **API Keys → Webhooks Signature Key**. Without it, the audit can still run all the negative tests (forged signature, no signature) — just not the positive one.

Verify the key works:

```bash
curl -s "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY"
```

If this returns anything other than 200, stop and report the auth issue.

## Step 2: Inventory the App's Hint-Side State

First look up the partner's product — every app endpoint is scoped to a `partner_product`:

```bash
curl -s "$HINT_API_URL/api/partner/partner_products" -H "Authorization: Bearer $API_KEY"
```

Pick the first entry (or match by name if there are multiple) and save its `id` as `$PRODUCT_ID`. Then:

```bash
# Partner-level config
curl -s "$HINT_API_URL/api/partner/partner" -H "Authorization: Bearer $API_KEY"

# App-level config (handshake URL + role mappings)
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app" -H "Authorization: Bearer $API_KEY"

# Anchors (per-surface source URLs)
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/anchors" -H "Authorization: Bearer $API_KEY"

# Services (deployed URLs, env vars, build/start commands)
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/services" -H "Authorization: Bearer $API_KEY"
```

Each of these returns a JSON document (services + anchors are bare arrays). Collect them — they're the inputs for the rest of the audit.

If `$APP_URL` wasn't provided, pick the row with `status: "active"` from the services list:

```bash
APP_URL=$(curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/services" -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((s['service_url'] for s in json.load(sys.stdin) if s.get('status')=='active' and s.get('service_url')),''))")
```

If `$APP_URL` is empty after that, report **PRE-AUDIT-FAIL: no active web service deployed** and stop.

## Step 3: Run the Checks

For each check below, record `PASS` / `FAIL` / `SKIP` and a one-line explanation. The order doesn't matter — run them all.

### 3.1 Marketplace listing completeness

Required fields:

- `partner.name` — non-empty
- `partner.email` — non-empty, looks like an email
- `partner.redirect_url` — non-empty, starts with `https://`, ends with a trailing slash or `/hint/connect/` (localhost development URLs live in the separate `localhost_redirect_url` column, never here)
- `partner.auth_type` — `automatic_headless` for production-ready apps
- `app.handshake_url` — non-empty, starts with `https://`, points at the same origin as `$APP_URL`
- `anchors` — at least one anchor registered, each with a non-empty `source_url` starting with `https://`

For each missing field, output a one-line remediation pointing at the relevant PATCH endpoint.

### 3.2 Anchor URL reachability

For each anchor:

```bash
curl -sS -o /dev/null -w "GET $ANCHOR_URL → HTTP %{http_code}\n" "$ANCHOR_URL"
```

PASS: 200 or 401 (some apps reject without a session; that's acceptable). FAIL: 404, 5xx, connection refused, TLS error.

### 3.3 Handshake — unsigned request rejected

```bash
curl -sS -o /dev/null -w "%{http_code}" -X POST "$APP_URL/hint/handshake" -H "Content-Type: application/json" -d '{}'
```

PASS: 401. FAIL: anything else (200 means signature verification is missing — **critical security gap**, partner is leaking arbitrary sessions to anyone who guesses the URL).

### 3.4 Handshake — forged signature rejected

```bash
BODY='{"user":{"id":"u-forge"},"practice":{"id":"p-forge"}}'
curl -sS -o /dev/null -w "%{http_code}" -X POST "$APP_URL/hint/handshake" \
  -H "Content-Type: application/json" \
  -H "X-Hint-Signature: sha256=0000000000000000000000000000000000000000000000000000000000000000" \
  -d "$BODY"
```

PASS: 401. FAIL: 200 (the handler is accepting the header's presence rather than verifying its value — a real HMAC compare against `0000…` would still reject. Look for code that short-circuits when `X-Hint-Signature` is set, or a comparator that ignores the body).

### 3.5 Handshake — valid signature accepted (only if webhook secret supplied)

```bash
BODY='{"user":{"id":"u-test","email":"audit@hint.com"},"practice":{"id":"p-test","name":"Audit Practice"}}'
SIG="sha256=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$WEBHOOK_SECRET" | awk '{print $NF}')"
curl -sS -o /dev/null -w "%{http_code}" -X POST "$APP_URL/hint/handshake" \
  -H "Content-Type: application/json" \
  -H "X-Hint-Signature: $SIG" \
  -d "$BODY"
```

PASS: 200. FAIL: 401 (signature verification is too strict — likely a raw-body capture bug; the framework re-serialized the body before the handler signed it).

SKIP: if the partner didn't provide the webhook secret.

### 3.6 HTTPS-only

Walk `partner.redirect_url`, `app.handshake_url`, every `anchor.source_url`. Each must start with `https://` — the prod URL columns reject plain http. Localhost development URLs live in the separate `localhost_*` siblings (`localhost_redirect_url`, `localhost_handshake_url`, `localhost_source_url`), which are http-only and sandbox-partners-only; the per-session `localhost_mode` flag is computed by Hint, not set on the app. Do not move a localhost URL into a prod column.

PASS: every prod URL is https. FAIL: any plain http URL in a prod column.

### 3.7 Env-var hygiene

For each service in `GET /partner/partner_products/$PRODUCT_ID/app/services`, fetch the full record:

```bash
curl -s "$HINT_API_URL/api/partner/partner_products/$PRODUCT_ID/app/services/$SERVICE_ID" -H "Authorization: Bearer $API_KEY"
```

Inspect the response's `env_vars` (jsonb object of partner-supplied custom vars). Reserved keys MUST NOT appear there:

- `HINT_API_URL`
- `HINT_API_KEY`
- `HINT_PARTNER_ID`
- `HINT_WEBHOOK_SECRET`
- `DATABASE_URL`

PASS: no reserved keys in `env_vars`. FAIL: any of them present (Hint ignores them in practice — system vars always win — but their presence signals the partner thinks they're configuring something they aren't).

### 3.8 Anchor / surface coverage

Some surfaces are sensitive to misconfiguration. For each anchor:

- `core_page`: `source_url` should end in `/hint/core_page` (convention; not enforced).
- `clinical_interaction`: `source_url` should end in `/hint/clinical_interaction`. Anchor's `interaction_type` field should be set if the partner expects to filter by interaction type.
- `settings`: `source_url` should end in `/hint/settings`. Anchor's `settings_label` should be set (otherwise the tab shows the app's generic name).

The partner is free to host the surface at any path that returns valid HTML — the convention just makes templates and audit reports easier to read. WARN (not FAIL) on deviations.

### 3.9 Service status sanity

For each service:

- `status: 'active'` — PASS
- `status: 'provisioning'` — WARN, deploy in progress
- `status: 'provisioning_failed'` — FAIL, partner needs to contact support or retry

### 3.10 Cross-practice tenancy probe (CRITICAL)

Static analysis can't reliably catch tenancy bugs — the only test that catches mis-scoping is exercising the app from two practices and confirming each can't read the other's data. This check requires two sandbox practices, fresh OAuth tokens for both, and a known write endpoint on the app — the most setup-heavy check in the suite. Run it last; skip cleanly (with a logged note) if the partner can't provide the inputs.

**Prerequisites:**
- The partner has at least two sandbox practices set up (create via Partner Portal → Sandboxes if not).
- API keys for both practices' installations of the app — these are different from the partner-wide API key. They come from the OAuth-exchange that runs on `/hint/connect/:code`.
- A way to write at least one piece of practice-scoped data via the app (call this the "probe write" — usually a POST to one of the app's own API routes).

**Probe sequence:**

1. As **practice A**: simulate handshake + connect → obtain `session_key_A` + `access_token_A` + `practice_id_A`.
2. As practice A: write a unique sentinel value into the app's state (POST to whatever the app's "create message" / "store config" / "save record" endpoint is, with `session_key_A`).
3. Read it back as practice A — confirm the sentinel appears.
4. As **practice B**: simulate handshake + connect → obtain `session_key_B`, `access_token_B`, `practice_id_B`.
5. As practice B: hit the same read endpoint with `session_key_B`. **Sentinel from step 2 must NOT appear in the response.**
6. As practice B: write a different sentinel. Confirm practice A's read with `session_key_A` still doesn't see practice B's sentinel.

PASS: each practice's reads return only its own writes. FAIL: any sentinel appears in the other practice's reads (cross-practice data leak — **critical, app must not go to live until fixed**).

If the app has no partner-controlled write endpoints (read-only against `/api/provider/*`), tenancy is automatic — the practice-scoped access_token enforces isolation at the Hint API layer. Skip this check with a note: `SKIPPED — app does not persist tenant state.`

Surface fixes from the [`retrofit` tenancy audit](https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/retrofit/SKILL.md) — schema changes + per-table `practice_id` + every query scoped. See `_common/marketplace-contract.md` "Tenancy" section.

## Step 4: Output the Report

Print a structured report grouped by severity. Example:

```
Hint Marketplace Audit — <partner.name>
Run against: $HINT_API_URL ($APP_URL)
Generated: <ISO timestamp>

CRITICAL (block release):
  ✗ 3.3 Handshake accepts unsigned requests — apps are leaking sessions. Fix signature verification immediately.

HIGH:
  ✗ 3.6 anchor 'core_page' source_url uses http:// (prod URLs must be https)

MEDIUM:
  ⚠ 3.1 partner.email is empty
  ⚠ 3.8 settings anchor has no settings_label

INFO / PASS:
  ✓ 3.2 all anchor URLs reachable
  ✓ 3.4 forged signatures rejected
  ✓ 3.5 valid signatures accepted (smoke test passed)
  ✓ 3.7 no reserved keys in custom env_vars
  ✓ 3.9 all services active

Remediation:
  - 3.3: Audit POST /hint/handshake handler. Confirm raw body is captured before JSON parsing and constant-time HMAC compare is used. See _common/marketplace-contract.md.
  - 3.6: PATCH /partner/partner_products/$PRODUCT_ID/app/anchors/<anchor_id> with source_url starting in https://. For local development set the anchor's localhost_source_url instead — the prod source_url must stay https.
  - 3.1: PATCH /partner/partner -d '{"partner":{"email":"..."}}'.
  - 3.8: PATCH /partner/partner_products/$PRODUCT_ID/app/anchors/<anchor_id> -d '{"anchor":{"settings_label":"..."}}'.
```

## Step 5: Severity Levels

- **CRITICAL** — security or functional break. App should not go live until fixed. Includes: handshake accepts unsigned requests, handshake accepts forged signatures, plain-http URLs leaking session keys, services in `provisioning_failed`, cross-practice data leak (3.10 probe fails).
- **HIGH** — functional issue. App will fail in some real-world scenarios. Includes: anchor URLs returning 404 / 5xx, missing handshake_url, missing redirect_url.
- **MEDIUM** — quality / UX issue. App works but looks incomplete. Includes: missing partner.email, missing anchor labels, partner.name placeholder text.
- **WARN** — convention deviation. Not wrong, just unusual. Includes: anchor source_url paths that don't match the template convention.
- **INFO / PASS** — everything is fine, surfaced for confidence.

## Troubleshooting

- **All checks FAIL with auth errors** — the API key is for a different environment than `$HINT_API_URL`. Sandbox key against live host (or vice versa) returns 401 on every call. Check the key prefix.
- **Smoke tests time out** — the deployed service isn't reachable from the audit's network. If running from a hint-internal network, that's expected for some self-hosted apps; ask the partner to run the audit themselves.
- **Forged-signature test returns 200 but unsigned returns 401** — extremely unusual; usually means the handler accepts ANY value for `X-Hint-Signature` as long as the header is present. Fail loudly.

For audit help, contact [devsupport@hint.com](mailto:devsupport@hint.com) with the report output attached.
