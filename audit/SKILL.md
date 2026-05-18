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

## Platform URLs

- **Hint API (sandbox)**: `https://api.sandbox.hint.com` — for `sbx-` API keys
- **Hint API (live)**: `https://api.hint.com` — for live API keys
- **Partner Portal**: `https://app.hint.com`

Set `$HINT_API_URL` based on the API key prefix.

## Step 1: Gather Inputs

Ask the partner:

1. **Partner API key** — sandbox (`sbx-...`) or live. The audit runs against whichever environment the key belongs to.
2. **App URL (optional)** — if the partner already knows their `$APP_URL`, save it. Otherwise the audit will discover it from `GET /partner/app/services`.
3. **Webhooks signature key (optional)** — needed for the "valid signature accepted" probe. Find it in the Partner Portal under **API Keys → Webhooks Signature Key**. Without it, the audit can still run all the negative tests (forged signature, no signature) — just not the positive one.

Verify the key works:

```bash
curl -s "$HINT_API_URL/api/partner/partner" \
  -H "Authorization: Bearer $API_KEY"
```

If this returns anything other than 200, stop and report the auth issue.

## Step 2: Inventory the App's Hint-Side State

```bash
# Partner-level config
curl -s "$HINT_API_URL/api/partner/partner" -H "Authorization: Bearer $API_KEY"

# App-level config (handshake URL + role mappings)
curl -s "$HINT_API_URL/api/partner/app" -H "Authorization: Bearer $API_KEY"

# Anchors (per-surface source URLs)
curl -s "$HINT_API_URL/api/partner/app/anchors" -H "Authorization: Bearer $API_KEY"

# Services (deployed URLs, env vars, build/start commands)
curl -s "$HINT_API_URL/api/partner/app/services" -H "Authorization: Bearer $API_KEY"
```

Each of these returns a JSON document (services + anchors are bare arrays). Collect them — they're the inputs for the rest of the audit.

If `$APP_URL` wasn't provided, pick the web service whose `service_url` is non-null and `status: "active"`:

```bash
APP_URL=$(curl -s "$HINT_API_URL/api/partner/app/services" -H "Authorization: Bearer $API_KEY" \
  | python3 -c "import sys,json; print(next((s['service_url'] for s in json.load(sys.stdin) if s.get('service_url') and s.get('status')=='active'),''))")
```

If `$APP_URL` is empty after that, report **PRE-AUDIT-FAIL: no active web service deployed** and stop.

## Step 3: Run the Checks

For each check below, record `PASS` / `FAIL` / `SKIP` and a one-line explanation. The order doesn't matter — run them all.

### 3.1 Marketplace listing completeness

Required fields:

- `partner.name` — non-empty
- `partner.email` — non-empty, looks like an email
- `partner.redirect_url` — non-empty, starts with `https://` (or `http://localhost` in localhost_mode), ends with a trailing slash or `/hint/connect/`
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

PASS: 401. FAIL: 200 (signature verification is wired up but the comparison is broken — usually string equality on the hex digest).

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

Walk `partner.redirect_url`, `app.handshake_url`, every `anchor.source_url`. Each must start with `https://`. Localhost URLs (`http://localhost:*`) are acceptable only if the app has `localhost_mode` enabled — check `app.localhost_mode` in the GET response.

PASS: every URL is https or localhost-with-localhost-mode. FAIL: any plain http URL outside localhost_mode.

### 3.7 Env-var hygiene

For each service in `GET /partner/app/services` with `service_type: 'web'`, fetch the full record:

```bash
curl -s "$HINT_API_URL/api/partner/app/services/$SERVICE_ID" -H "Authorization: Bearer $API_KEY"
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

- `core_page`: `source_url` should end in `/hint/core_page` (convention; not enforced but consistent with templates).
- `clinical_interaction`: `source_url` should end in `/hint/clinical_interaction`. Anchor's `interaction_type` field should be set if the partner expects to filter by interaction type.
- `settings`: `source_url` should end in `/hint/settings`. Anchor's `settings_label` should be set (otherwise the tab shows the app's generic name).

WARN (not FAIL): partner is free to use other paths; just flag.

### 3.9 Service status sanity

For each service:

- `status: 'active'` — PASS
- `status: 'provisioning'` — WARN, deploy in progress
- `status: 'provisioning_failed'` — FAIL, partner needs to contact support or retry

## Step 4: Output the Report

Print a structured report grouped by severity. Example:

```
Hint Marketplace Audit — <partner.name>
Run against: $HINT_API_URL ($APP_URL)
Generated: <ISO timestamp>

CRITICAL (block release):
  ✗ 3.3 Handshake accepts unsigned requests — apps are leaking sessions. Fix signature verification immediately.

HIGH:
  ✗ 3.6 anchor 'core_page' source_url uses http:// in non-localhost-mode app

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
  - 3.6: PATCH /partner/app/anchors/<anchor_id> with source_url starting in https://, or enable localhost_mode for development.
  - 3.1: PATCH /partner/partner -d '{"partner":{"email":"..."}}'.
  - 3.8: PATCH /partner/app/anchors/<anchor_id> -d '{"anchor":{"settings_label":"..."}}'.
```

## Step 5: Severity Levels

- **CRITICAL** — security or functional break. App should not go live until fixed. Includes: handshake accepts unsigned requests, handshake accepts forged signatures, plain-http URLs leaking session keys, services in `provisioning_failed`.
- **HIGH** — functional issue. App will fail in some real-world scenarios. Includes: anchor URLs returning 404 / 5xx, missing handshake_url, missing redirect_url.
- **MEDIUM** — quality / UX issue. App works but looks incomplete. Includes: missing partner.email, missing anchor labels, partner.name placeholder text.
- **WARN** — convention deviation. Not wrong, just unusual. Includes: anchor source_url paths that don't match the template convention.
- **INFO / PASS** — everything is fine, surfaced for confidence.

## Troubleshooting

- **All checks FAIL with auth errors** — the API key is for a different environment than `$HINT_API_URL`. Sandbox key against live host (or vice versa) returns 401 on every call. Check the key prefix.
- **Smoke tests time out** — the deployed service isn't reachable from the audit's network. If running from a hint-internal network, that's expected for some self-hosted apps; ask the partner to run the audit themselves.
- **Forged-signature test returns 200 but unsigned returns 401** — extremely unusual; usually means the handler accepts ANY value for `X-Hint-Signature` as long as the header is present. Fail loudly.

For audit help, contact [devsupport@hint.com](mailto:devsupport@hint.com) with the report output attached.
