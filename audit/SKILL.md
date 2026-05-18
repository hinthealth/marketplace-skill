# /hint-marketplace-audit — Security + Contract Audit of a Deployed App

**Status:** 📋 not yet implemented. This skill is on the roadmap. Track progress at https://github.com/hinthealth/marketplace-skill/issues or contact [devsupport@hint.com](mailto:devsupport@hint.com).

## What this skill will do

Given a partner's API key (sandbox or live) and the deployed app's `$APP_URL`, run a pass/fail audit of marketplace-contract compliance + security posture, and produce a report the partner can act on.

Checks (planned):

1. **Handshake signature verification** — POST `/hint/handshake` with an unsigned payload; expect 401. POST with a forged signature; expect 401. POST with a valid signature (the skill mints one using the partner's webhook secret); expect 200 + session_key.
2. **Headless connect token storage** — confirm the practice access token returned by `/api/oauth/tokens` is persisted server-side, never echoed back in the connect response, never exposed via a query param.
3. **Provider API scoping** — confirm every `/api/provider/*` call uses the practice-scoped token, not the partner-wide `HINT_API_KEY`. Cross-practice data leaks happen when this is wrong.
4. **Webhook verification** — if the app subscribes to Hint webhooks, confirm it verifies the Standard Webhooks signature (Svix) on every payload.
5. **Env-var hygiene** — partner-supplied env vars should not include `HINT_API_KEY`, `HINT_WEBHOOK_SECRET`, or `DATABASE_URL` (those are reserved + system-managed). Audit via the public `GET /partner/app/services/:id` response.
6. **Anchor URL correctness** — every anchor's `source_url` resolves on `$APP_URL` with HTTP 200 + Content-Type `text/html` when given a valid `session_key`.
7. **HTTPS-only** — confirm all anchor URLs + handshake URL are `https://` (localhost is allowed only when the partner has `localhost_mode` enabled).
8. **Marketplace listing completeness** — partner has `name`, `redirect_url`, `auth_type` set; app has `handshake_url` + at least one anchor.

Output: a structured pass/fail table + a remediation checklist.

## Until this skill ships

Run the smoke-test curls from `create-app`'s Self-Hosted Mode Step 4 manually:

```bash
curl -sS -o /dev/null -w "GET /  → HTTP %{http_code}\n" "$APP_URL/"
curl -sS -o /dev/null -w "POST /hint/handshake (unsigned, expect 401) → HTTP %{http_code}\n" -X POST "$APP_URL/hint/handshake"
```

Or open a ticket at [devsupport@hint.com](mailto:devsupport@hint.com) and we'll audit by hand.
